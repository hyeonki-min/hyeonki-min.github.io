---
title: "Rate Limiter 정리"
date: 2025-11-19 18:00:00 +0900
categories: [Backend, RateLimit]
tags: [Java, RateLimiter, TokenBucket, SlidingWindow]
---

일반적으로 API 호출에는 초당 제한이 걸려있습니다. 트래픽의 처리율을 제어하기 위함인데, Dos(Denial of Service) 공격에 의한 자원 고갈을 방지하고, 추가 요청에 대한 처리를 제한하여 비용을 절감하고, 봇이나 사용자의 잘못된 이용으로 유발된 트래픽을 걸러내어 서버 과부하를 막을 수 있습니다.

예시 빗썸 api 제한 안내

![빗썸 api 제한](/assets/img/rate-limiters/bithumb.PNG)


## 처리율 제한 장치(Rate Limiter)

처리율 제한 장치는 클라이언트 또는 서비스가 보내는 트래픽을 제어하기 위한 장치입니다. 특정 시간 동안 허용할 수 있는 요청 수를 제어하는 것으로 일반적으로 API Gateway에서 구현되고, SSL 종단, 사용자 인증, IP허용 목록 관리 등 을 지원하는 미들웨어입니다.

자바, 스프링 기반으로 애플리케이션 레이어에서 처리율 제한 알고리즘을 구현하여 정리해보겠습니다.

## 전제 조건

초당 10회 제한 API를 구현하라

## 토큰 버킷(Token Bucket)

### 개념

양동이에 토큰이 차오르고, 요청이 들어올 때 토큰을 꺼내 쓰는 모델

1초에 10개의 토큰이 생성됨, 요청 1건 = 토큰 1개 소비, 토큰이 없으면 거부

### 동작 방식

1. 용량 10인 토큰 버킷이 있다
2. 토큰 공급기는 매초 1개의 토큰을 추가한다, 버킷이 가득 차면 추가로 공급된 토큰은 버려진다
3. 각 요청은 처리될 때마다 하나의 토큰을 사용한다.
4. 충분한 토큰이 없는 경우, 해당 요청은 버려진다.

### 장점

- 구현이 쉽다.
- 메모리 사용 측면에서 효율적이다.
- Burst traffic 허용

### 단점

- 버킷 크기와 토큰 공급률이라는 두 개 인자를 적절히 튜닝 하는 것이 어렵다.
- 적절한 시간 기반 제한과는 결이 다름

### 예시

```java
public class TokenBucketRateLimiter {

    // 사용자별 버킷 저장소
    private final Map<String, TokenBucket> buckets = new ConcurrentHashMap<>();

    public boolean allowRequest(String key, int capacity, double refillRatePerSec) {
        TokenBucket bucket = buckets.computeIfAbsent(key,
                k -> new TokenBucket(capacity, refillRatePerSec));
        return bucket.tryConsume(1);
    }

    private static class TokenBucket {
        private final int capacity;
        private final double refillRatePerSec;
        private double tokens;
        private long lastRefillTimestamp;

        TokenBucket(int capacity, double refillRatePerSec) {
            this.capacity = capacity;
            this.refillRatePerSec = refillRatePerSec;
            this.tokens = capacity;
            this.lastRefillTimestamp = System.nanoTime();
        }

        synchronized boolean tryConsume(int requestedTokens) {
            refill();
            if (tokens >= requestedTokens) {
                tokens -= requestedTokens;
                return true;
            }
            return false;
        }

        private void refill() {
            long now = System.nanoTime();
            double elapsedSeconds = (now - lastRefillTimestamp) / 1_000_000_000.0;
            double newTokens = elapsedSeconds * refillRatePerSec;
            if (newTokens > 0) {
                tokens = Math.min(capacity, tokens + newTokens);
                lastRefillTimestamp = now;
            }
        }
    }
}
```

### 구조 요약

**allowRequest()**

- 동일한 key로 여러 번 요청하면 항상 같은 tokenbucket 객체를 사용
- kye마다 하나씩 버킷
    - capacity : 버킷 최대 토큰 수
    - refillRatePerSec : 초당 토큰 생성량

**tryConsume()**

- refill()로 먼저 토큰 보충
- 토큰이 충분하면 요청 허용
- 토큰이 부족하면 요청 거부
- synchronized로 race condition 예방

**refill()**

- 지난 리필 이후 얼머나 시간이 지났는지 계산
- 그 시간 동안 생성할 통큰 수 계산
- 토큰 추가
- 마지막 리필 시간 업데이트

## 누출 버킷(Leaky Bucket)

### 개념

양동이에서 일정한 속도로 물이 한방울씩 떨어지는 모델

요청은 버킷에 쌓이고 초당 10회 속도로 처리됨, 초과 요청은 큐에 쌓이거나 버림

### 동작 방식

1. 요청이 도착하면 큐에 쌓고
2. 가득 차있는 경우에는 새 요청은 버린다
3. 지정된 시간마다 큐에서 요청을 꺼내어 처리한다

### 장점

- 큐의 크기가 제한되어 메모리 사용량 측면에서 효율적
- 고정된 처리율로 안정적 출력이 가능

### 단점

- 대기열이 길어지면 응답 지연 증가
- 실시간 API보다 비동기 처리/워커 패턴에 적합

### 예시

```java
public class LeakyBucketRateLimiter {

    private final Map<String, LeakyBucket> buckets = new ConcurrentHashMap<>();

    public boolean allowRequest(String key, int capacity, double leakRatePerSec) {
        LeakyBucket bucket = buckets.computeIfAbsent(
                key, k -> new LeakyBucket(capacity, leakRatePerSec));
        return bucket.tryPush();
    }

    private static class LeakyBucket {
        private final int capacity;
        private final double leakRatePerSec;
        private final Deque<Long> queue; // timestamp queue
        private long lastLeakTime;

        LeakyBucket(int capacity, double leakRatePerSec) {
            this.capacity = capacity;
            this.leakRatePerSec = leakRatePerSec;
            this.queue = new ArrayDeque<>(capacity);
            this.lastLeakTime = System.nanoTime();
        }

        synchronized boolean tryPush() {
            leak();  // 먼저 일정 개수 제거

            // 큐가 꽉 차 있으면 drop = 429
            if (queue.size() >= capacity) {
                return false;
            }

            // 큐에 요청을 추가 (timestamp 저장)
            queue.addLast(System.nanoTime());
            return true;
        }

        private void leak() {
            long now = System.nanoTime();
            double elapsedSec = (now - lastLeakTime) / 1_000_000_000.0;
            int leaks = (int)(elapsedSec * leakRatePerSec);

            for (int i = 0; i < leaks && !queue.isEmpty(); i++) {
                queue.pollFirst(); // FIFO
            }

            if (leaks > 0) {
                lastLeakTime = now;
            }
        }
    }
}
```

### 구조 요약

**allowRequest()**

- 동일한 key로 여러 번 요청하면 항상 같은 tokenbucket 객체를 사용
- kye마다 하나씩 버킷
    - capacity : 버킷 최대 토큰 수
    - refillRatePerSec : 초당 버킷 요청 처리량

**tryPush()**

- leak()로 오래된 요청 제거
- 현재 큐가 capacity만큼 차 있으면 거부
- 큐에 공간이 있으면 timestamp 넣기

**leak()**

- lastLeakTime 이후 몇 초가 지났는지 계산
- 그 시간 동안 처리되었어야 할 요청 개수 계산
- queue에서 초당 처리량만큼 pollFirst()
- 처리된 경우 lastLeakTime 업데이트

## 고정 윈도 카운터(Fixed Window Counter)

### 개념

매 초 단위로 카운터를 리셋하는 단순한 제한 방식

특정 1초 구간 안에서 요청 수를 세고 초과시 즉시 429

### 동작 방식

1. 타임라인을 고전된 간격의 윈도로 나누고, 각 윈도마다 카운터를 붙임
2. 요청이 들어올 때마다 현재 윈도우 키의 카운터를 증가
3. 카운터 한도를 넘으면 거부
4. 윈도우가 바뀌면 카운터 리셋

### 장점

- 구현이 매우 단순
- 메모리 효율이 좋다
- 이해하기 쉽다

### 단점

- Boundary Effect, 윈도 경계 부근에서 일시적으로 많은 트래픽이 몰리는 경우, 기대했던 한도보다 많은 양의 요청을 처리하게 됨

### 예시

```java
public class FixedWindowRateLimiter {

    private static class Window {
        final long windowStart;
        final AtomicInteger count = new AtomicInteger(0);

        Window(long windowStart) {
            this.windowStart = windowStart;
        }

        boolean tryAcquire(int limit) {
            while (true) {
                int current = count.get();
                if (current >= limit) return false;
                if (count.compareAndSet(current, current + 1)) return true;
            }
        }
    }

    private final ConcurrentHashMap<String, Window> windows = new ConcurrentHashMap<>();

    public boolean allowRequest(String key, int limit, int windowSizeInSeconds) {
        long now = System.currentTimeMillis();
        long windowSizeMillis = windowSizeInSeconds * 1000;
        long windowStart = now / windowSizeMillis * windowSizeMillis;

        Window window = windows.compute(key, (k, w) -> {
            if (w == null || w.windowStart != windowStart) {
                return new Window(windowStart);
            }
            return w;
        });

        return window.tryAcquire(limit);
    }
}
```

### 구조 요약

**allowRequest()**

- 현재 요청이 속한 윈도우의 시간 시각 계산
- key 별로 윈도우 객체 관리
- 해당 window의 카운터 증가 시도

**tryAcquire()**

- 현재 count 조회
- count가 한도보다 많으면 거부
- 아니면 compareAndSet로 동시성 안전하게 증가, 실패하면 다시 시도

## 이동 윈도 로깅(Sliding Window Logging)

### 개념

고정 윈도 경계 문제를 완화하기 위해 현재 시각 기준으로 직전 1초 동안의 요청수를 보고 제한하는 방식

들어온 요청마다 timestamp를 저장, 현재시간 - 1000ms 이전 요청은 제거

### 동작 방식

1. 요청마다 timestamp를 저장
2. 현재시간 - 1000ms 이전 요청은 제거
3. 남아 있는 요청 수가 limit을 초과하면 버림

### 장점

- 이론적으로 가장 정확한 초당 제한 구현 가능

### 단점

- 정확히 1초에 10회만 허용 같은 제한 정책이 엄격함
- 다량의 메모리 사용

### 예시

```java
public class SlidingWindowLogRateLimiter {

    private final ConcurrentHashMap<String, WindowLog> logs = new ConcurrentHashMap<>();

    public boolean allowRequest(String key, int limit, long windowSizeInSeconds) {
        long now = System.currentTimeMillis();
        WindowLog log = getOrCreateLog(key);
        long windowMillis = windowSizeInSeconds * 1000L;
        return log.allow(now, limit, windowMillis);
    }

    private WindowLog getOrCreateLog(String key) {
        WindowLog existing = logs.get(key);
        if (existing != null) {
            return existing;
        }

        WindowLog newLog = new WindowLog();
        WindowLog race = logs.putIfAbsent(key, newLog);
        return race != null ? race : newLog;
    }

    private static class WindowLog {
        // 요청 timestamp 목록 (ms)
        private final Deque<Long> timestamps = new ArrayDeque<>();

        synchronized boolean allow(long nowMillis, int limit, long windowMillis) {
            long boundary = nowMillis - windowMillis;

            // 1) 윈도우 밖(오래된) 요청들 정리
            while (!timestamps.isEmpty() && timestamps.peekFirst() <= boundary) {
                timestamps.pollFirst();
            }

            // 2) 현재 윈도우 내 요청 수 확인
            if (timestamps.size() >= limit) {
                // 이미 꽉 찼으니 거절
                return false;
            }

            // 3) 아직 여유가 있으니 허용 + 현재 요청 기록
            timestamps.addLast(nowMillis);
            return true;
        }
    }
}
```

### 구조 요약

**allowRequest()**

- 현재 시간 획득
- key에 해당하는 WindowLog 가져오기
- windowMillis단위로 슬라이딩 판단 수행

**getOrCreateLog()**

- 이미 존재하는 WindowLog 있으면 반환
- 없으면 새로 만들고 putIfAbset로 race condition방지

**allow()**

- 슬라이딩 윈도우 경계를 계산
- 오래된 요청 제거
- 현재 윈도우 내 요청 수 확인
- 요청 허용 + 현재 timestamp 추가

## 이동 윈도 카운터(Sliding Window Counter)

### 개념

바로 이전 윈도와 현재 윈도의 카운터를 이용해 가중합으로 근사하여 제한

직전 1초 동안의 요청 수를 근사치로 계산, 두 구간(현재 1초, 직전 1초)구간의 카운터를 가중치로 결합

### 동작 방식

1. 직전 1초동안의 요청 수를 근사치로 계산
2. 두 구간의 카운터를 가중치로 결합

### 장점

- fixed window처럼 버스트가 심하지 않음
- 정확도는 sliding window logging 보다 낮지만 비용은 적음

### 단점

- 적정 시간대에 도착한 요청이 균등하게 분포되어 있다고 가정한 상태에서 추정치를 계산하기 때문에 다소 느슨.

### 예시

```java
synchronized boolean allow(long nowMillis, int limit) {
    long windowStart = alignedWindowStart(nowMillis);

    // 윈도우 이동 처리
    if (windowStart > currentWindowStart) {
        long diff = windowStart - currentWindowStart;

        if (diff == windowSizeMillis) {
            // 정확히 한 윈도우만큼만 이동: current → previous
            previousCount = currentCount;
        } else {
            // 두 개 이상 윈도우를 건너뛰었으면 이전 윈도우는 의미 없음
            previousCount = 0;
        }
        currentWindowStart = windowStart;
        currentCount = 0;
    }

    // fraction: 현재 윈도우 진행률 (0 ~ 1)
    double fraction = (double) (nowMillis - currentWindowStart) / windowSizeMillis;
    if (fraction < 0) fraction = 0;
    if (fraction > 1) fraction = 1;

    double estimated =
            previousCount * (1.0 - fraction) +
            currentCount;

    if (estimated >= limit) {
        return false;
    }

    currentCount++;
    return true;
}
```

### 구조 요약

**allowRequst()**

- key마다 WindowCounter 생성
- 요청마다 counter.allow() 호출

**allow()**

- 이번 요청이 속한 윈도우 계산
- 윈도우가 변경되었는 지 확인
- 현재 윈도 경과 비율 계산
- 이전 윈도 + 현재의 윈도의 가중치 합 계산
- estimated ≥ limit 여부 체크

## 예시코드

모든 코드는 [깃헙](https://github.com/hyeonki-min/spring-labs)에 있습니다.

### 참고문헌

알렉스 쉬, 가상 면접 사례로 배우는 대규모 시스템 설계 기초, 2021