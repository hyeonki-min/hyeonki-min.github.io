---
title: "dcm4che로 DICOM JPEG Lossless 압축 구현하기"
date: 2026-05-04 00:30:00 +0900
categories: [DICOM, Backend]
tags: [Java, DICOM, dcm4che, JPEG-Lossless, Transfer-Syntax, PACS]
---

의료 영상 시스템을 개발하다 보면 DICOM 파일을 PACS로 전송하거나 장기 보관해야 하는 경우가 많습니다. 이때 파일 크기를 줄이기 위해 압축을 고려할 수 있지만, 의료 영상에서는 픽셀 값이 진단과 후처리의 근거가 되기 때문에 일반적인 손실 압축을 무작정 적용하기 어렵습니다.

이런 경우 원본 픽셀 값을 보존하면서 파일 크기를 줄일 수 있는 JPEG Lossless 압축을 고려할 수 있습니다.

이번 글에서는 Java DICOM 라이브러리인 dcm4che를 사용해 DICOM 파일을 JPEG Lossless Transfer Syntax로 변환하는 예제를 정리합니다.

## 1. JPEG Lossless란?

DICOM에서 JPEG Lossless는 픽셀 값을 손실 없이 압축하는 Transfer Syntax입니다.

대표적으로 사용하는 UID는 다음과 같습니다.

```
1.2.840.10008.1.2.4.70
```

dcm4che에서는 이 Transfer Syntax를 다음 상수로 사용할 수 있습니다.

```
UID.JPEGLosslessSV1
```

비슷한 이름으로 JPEG-LS Lossless도 있지만, 두 방식은 서로 다른 Transfer Syntax입니다.

```
JPEG Lossless      : 1.2.840.10008.1.2.4.70
JPEG-LS Lossless   : 1.2.840.10008.1.2.4.80
```

이 글에서는 JPEG Lossless, 즉 `1.2.840.10008.1.2.4.70`을 기준으로 설명합니다.

## 2. 왜 JPEGLosslessSV1을 사용하나요?

의료 영상에서는 픽셀 값이 중요합니다.

CT, MR 같은 영상에서 픽셀 값은 단순한 화면 색상이 아니라 진단, 측정, 후처리 알고리즘의 입력값이 될 수 있습니다. 따라서 원본 보존이 필요한 경우에는 손실 압축보다 무손실 압축을 우선 고려해야 합니다.

JPEGLosslessSV1을 사용하는 이유는 다음과 같습니다.

1. DICOM 표준 Transfer Syntax입니다.
2. 무손실 압축이므로 픽셀 값을 보존할 수 있습니다.
3. MR/CT 같은 16bit grayscale 영상에 사용할 수 있습니다.
4. dcm4che Transcoder에서 목적 Transfer Syntax로 지정할 수 있습니다.
5. PACS나 Viewer가 지원하면 표준 DICOM으로 전송하거나 보관할 수 있습니다.

비슷한 무손실 Transfer Syntax로는 JPEG-LS, JPEG 2000 Lossless, RLE Lossless도 있습니다.

```
JPEG Lossless SV1   : 1.2.840.10008.1.2.4.70
JPEG-LS Lossless    : 1.2.840.10008.1.2.4.80
JPEG 2000 Lossless  : 1.2.840.10008.1.2.4.90
RLE Lossless        : 1.2.840.10008.1.2.5
```

다만 운영 환경에서는 압축률만 보고 선택하면 안 됩니다. 대상 PACS, Viewer, Gateway가 해당 Transfer Syntax를 지원하는지 반드시 확인해야 합니다.

## 3. 의존성 추가

예제는 Gradle 기준으로 작성합니다.
```gradle
repositories {
    mavenCentral()

    maven {
        url = uri("https://maven.dcm4che.org/")
    }
}

dependencies {
    implementation "org.dcm4che:dcm4che-core:5.34.1"
    implementation "org.dcm4che:dcm4che-imageio:5.34.1"
    implementation "org.dcm4che:dcm4che-imageio-opencv:5.34.1"
}
```

단순히 DICOM 태그를 읽는 정도라면 dcm4che-core만으로도 가능합니다. 하지만 JPEG Lossless 압축처럼 Pixel Data를 재인코딩하려면 `dcm4che-imageio`, `dcm4che-imageio-opencv`가 필요합니다.


## 4. JPEG Lossless 압축 코드

dcm4che에서는 Transcoder를 사용해 DICOM 파일의 Transfer Syntax를 변경할 수 있습니다.

```java
private static final String JPEG_LOSSLESS_TS = UID.JPEGLosslessSV1;

public static void compress(File inputFile, File outputFile) throws IOException {
    try (Transcoder transcoder = new Transcoder(inputFile)) {
        transcoder.setIncludeFileMetaInformation(true);
        transcoder.setDestinationTransferSyntax(JPEG_LOSSLESS_TS);

        transcoder.transcode(new Transcoder.Handler() {
            @Override
            public OutputStream newOutputStream(
                Transcoder transcoder,
                Attributes dataset
            ) throws IOException {
                return new FileOutputStream(outputFile);
            }
        });
    }
}
```

핵심은 다음 두 줄입니다.
```
transcoder.setIncludeFileMetaInformation(true);
transcoder.setDestinationTransferSyntax(UID.JPEGLosslessSV1);
```
`setDestinationTransferSyntax()`는 출력 DICOM의 목적 Transfer Syntax를 JPEG Lossless로 지정합니다.

`setIncludeFileMetaInformation(true)`는 출력 DICOM에 File Meta Information을 포함하도록 설정합니다. 이 설정이 빠지면 압축 결과를 검증할 때 Transfer Syntax가 기대한 대로 보이지 않거나, File Meta Information이 의도대로 기록되지 않을 수 있습니다.

람다로 조금 더 간단히 작성하면 다음과 같습니다.
```java
public static void compress(File inputFile, File outputFile) throws IOException {
    try (Transcoder transcoder = new Transcoder(inputFile)) {
        transcoder.setIncludeFileMetaInformation(true);
        transcoder.setDestinationTransferSyntax(UID.JPEGLosslessSV1);

        transcoder.transcode((transcoderRef, dataset) ->
            new FileOutputStream(outputFile)
        );
    }
}
```

## 5. dcm4che Transcoder 동작 원리

dcm4che의 Transcoder는 DICOM 파일을 읽으면서 Transfer Syntax를 변환합니다.

대략적인 흐름은 다음과 같습니다.

1. 입력 DICOM 파일을 엽니다.
2. Source Transfer Syntax를 확인합니다.
3. Destination Transfer Syntax를 설정합니다.
4. Dataset을 읽습니다.
5. PixelData를 만나면 기존 Pixel Data를 디코딩합니다.
6. 목적 Transfer Syntax에 맞게 Pixel Data를 다시 인코딩합니다.
7. Dataset과 File Meta Information을 출력합니다.

transcode()를 호출하면 dcm4che는 Pixel Data를 읽고, OpenCV 기반 ImageIO Writer를 통해 JPEG Lossless bitstream을 생성합니다.

그 결과 DICOM의 Pixel Data는 일반 raw pixel data가 아니라 encapsulated 형태로 저장됩니다.

개념적으로는 다음과 같은 구조가 됩니다.
```
DICOM File
 ├─ File Meta Information
 │  └─ TransferSyntaxUID = 1.2.840.10008.1.2.4.70
 ├─ Dataset
 └─ Encapsulated Pixel Data
```
즉, 결과 파일은 일반 JPEG 이미지 파일이 아닙니다. DICOM Dataset 안의 Pixel Data가 JPEG Lossless 방식으로 압축된 DICOM 파일입니다.

## 6. OpenCV Native 라이브러리 로딩 실패

의존성을 추가한 뒤 압축 코드를 실행하면 다음과 같은 오류를 만날 수 있습니다.
```java
Exception in thread "main" javax.imageio.IIOException: Native JPEG encoding error
        at org.dcm4che3.opencv.NativeJPEGImageWriter.write(NativeJPEGImageWriter.java:174)
        at org.dcm4che3.imageio.codec.Transcoder.compressFrame(Transcoder.java:752)
        at org.dcm4che3.imageio.codec.Transcoder.compressPixelData(Transcoder.java:595)
        at org.dcm4che3.imageio.codec.Transcoder.processPixelData(Transcoder.java:516)
        at org.dcm4che3.imageio.codec.Transcoder.access$800(Transcoder.java:71)
        at org.dcm4che3.imageio.codec.Transcoder$1.readValue(Transcoder.java:469)
        at org.dcm4che3.io.DicomInputStream.readAttributes(DicomInputStream.java:772)
        at org.dcm4che3.io.DicomInputStream.readAllAttributes(DicomInputStream.java:632)
        at org.dcm4che3.imageio.codec.Transcoder.transcode(Transcoder.java:442)
        at min.hyeonki.dcm4che_example.compressor.DicomJpegLosslessCompressor.compress(DicomJpegLosslessCompressor.java:30)
        at min.hyeonki.dcm4che_example.compressor.DicomJpegLosslessCompressor.main(DicomJpegLosslessCompressor.java:20)
Caused by: java.lang.UnsatisfiedLinkError: 'long org.opencv.core.Mat.n_Mat(int, int, int)'
        at org.opencv.core.Mat.n_Mat(Native Method)
        at org.opencv.core.Mat.<init>(Mat.java:37)
        at org.weasis.opencv.data.ImageCV.<init>(ImageCV.java:28)
        at org.weasis.opencv.op.ImageConversion.toMat(ImageConversion.java:256)
        at org.weasis.opencv.op.ImageConversion.toMat(ImageConversion.java:187)
        at org.dcm4che3.opencv.NativeJPEGImageWriter.write(NativeJPEGImageWriter.java:128)
        ... 10 more
```
이 오류는 Java 클래스는 찾았지만, OpenCV Native 라이브러리가 로드되지 않았다는 의미입니다.

dcm4che-imageio-opencv는 JPEG 압축과 해제를 위해 OpenCV Native 라이브러리를 사용합니다. 따라서 jar 의존성만 추가한다고 끝나는 것이 아니라, OS별 Native 라이브러리도 실행 환경에서 로드되어야 합니다.

Windows 기준으로는 OpenCV DLL이 필요합니다.
```
opencv_java.dll
```
DLL을 단순히 시스템 폴더에 넣을 수도 있지만 권장하지 않습니다.
```
비추천 위치:
C:\Windows\System32
JDK bin 폴더
```
이 방식은 내 PC에서는 동작할 수 있지만, 다른 개발자 PC나 CI 환경에서는 재현되지 않을 수 있습니다. 또한 다른 프로젝트와 DLL 버전 충돌이 발생할 가능성도 있습니다.

더 나은 방식은 프로젝트 내부나 Gradle build 디렉터리에 Native 라이브러리를 두고, 실행 시 java.library.path로 연결하는 것입니다.

예를 들어 프로젝트 내부에 다음과 같이 둘 수 있습니다.
```
dcm4che-example
├─ native
│  └─ opencv_java.dll
└─ build.gradle
```
그리고 build.gradle에서 실행 시 Native 라이브러리 경로를 지정합니다.
```gradle
application {
    mainClass = "min.hyeonki.dcm4che_example.compressor.DicomJpegLosslessCompressor"

    applicationDefaultJvmArgs = [
        "-Djava.library.path=${projectDir}/native"
    ]
}
```
코드에서 명시적으로 로딩 테스트를 하고 싶다면 다음 코드를 추가할 수 있습니다.
```
static {
    System.loadLibrary("opencv_java");
}
```
이 단계에서 실패한다면 압축 코드의 문제가 아니라 Native 라이브러리 경로 또는 DLL 로딩 문제로 볼 수 있습니다.

## 7. 압축 대상 DICOM 확인하기

압축이 적용되지 않을 때는 먼저 입력 DICOM이 실제로 압축 가능한 영상인지 확인해야 합니다.

다음 코드는 DICOM 파일의 주요 속성을 출력합니다.
```java
public static void print(File file) throws IOException {
    if (!file.exists()) {
        throw new IllegalArgumentException(
            "File does not exist: " + file.getAbsolutePath()
        );
    }

    try (DicomInputStream dis = new DicomInputStream(file)) {
        Attributes fmi = dis.readFileMetaInformation();
        Attributes dataset = dis.readDataset(-1, -1);

        String transferSyntax = fmi != null
            ? fmi.getString(Tag.TransferSyntaxUID)
            : dis.getTransferSyntax();

        System.out.println("========== " + file.getName() + " ==========");
        System.out.println("File size              : " + file.length());
        System.out.println("Transfer Syntax UID    : " + transferSyntax);
        System.out.println("SOP Class UID          : " + dataset.getString(Tag.SOPClassUID));
        System.out.println("Rows                   : " + dataset.getInt(Tag.Rows, -1));
        System.out.println("Columns                : " + dataset.getInt(Tag.Columns, -1));
        System.out.println("Number Of Frames       : " + dataset.getString(Tag.NumberOfFrames, "1"));
        System.out.println("Samples Per Pixel      : " + dataset.getInt(Tag.SamplesPerPixel, -1));
        System.out.println("Photometric            : " + dataset.getString(Tag.PhotometricInterpretation));
        System.out.println("Bits Allocated         : " + dataset.getInt(Tag.BitsAllocated, -1));
        System.out.println("Bits Stored            : " + dataset.getInt(Tag.BitsStored, -1));
        System.out.println("Pixel Representation   : " + dataset.getInt(Tag.PixelRepresentation, -1));
        System.out.println("Has PixelData          : " + dataset.contains(Tag.PixelData));
    }
}
```
예제에 사용한 MRBRAIN.DCM은 다음과 같은 값을 가지고 있었습니다.
```
========== MRBRAIN.DCM ==========
File size              : 526336
Transfer Syntax UID    : 1.2.840.10008.1.2.1
Transfer Syntax Name   : Explicit VR Little Endian
SOP Class UID          : 1.2.840.10008.5.1.4.1.1.4
Rows                   : 512
Columns                : 512
Number Of Frames       : 1
Samples Per Pixel      : 1
Photometric            : MONOCHROME2
Bits Allocated         : 16
Bits Stored            : 12
Pixel Representation   : 0
Has PixelData          : true
```
이 조건이면 JPEG Lossless 압축 대상으로 볼 수 있습니다.
```
PixelData 있음
BitsAllocated = 16
BitsStored = 12
MONOCHROME2
MR Image Storage
```
반대로 다음과 같은 경우에는 압축이 적용되지 않거나, 압축 대상이 아닐 수 있습니다.
```
PixelData가 없는 DICOM
Structured Report
Encapsulated PDF
metadata-only DICOM
BitsAllocated = 1인 일부 영상
이미 압축된 영상이지만 디코딩 코덱이 없는 경우
```

## 8. 압축 결과 확인하기

압축된 파일이 실제로 JPEG Lossless Transfer Syntax를 사용하는지 확인하려면 File Meta Information의 TransferSyntaxUID를 확인하면 됩니다.
```java
private static DicomFileInfo readInfo(File file) throws IOException {
    if (!file.exists()) {
        throw new IllegalArgumentException("File does not exist: " + file.getAbsolutePath());
    }

    try (DicomInputStream dis = new DicomInputStream(file)) {
        Attributes fileMetaInformation = dis.readFileMetaInformation();

        String transferSyntaxUid;
        if (fileMetaInformation != null) {
            transferSyntaxUid = fileMetaInformation.getString(Tag.TransferSyntaxUID);
        } else {
            transferSyntaxUid = dis.getTransferSyntax();
        }

        return new DicomFileInfo(
            file.getName(),
            file.getAbsolutePath(),
            file.length(),
            transferSyntaxUid,
            resolveTransferSyntaxName(transferSyntaxUid)
        );
    }
}
```
정상적으로 압축되었다면 다음 값이 출력되어야 합니다.
```
Transfer Syntax   : 1.2.840.10008.1.2.4.70
Transfer Name     : JPEG Lossless, Non-Hierarchical, First-Order Prediction
```
만약 다음처럼 출력된다면 JPEG Lossless 압축이 적용되지 않은 것입니다.
```
Validation        : FAIL - JPEG Lossless Transfer Syntax가 아닙니다.
Expected          : 1.2.840.10008.1.2.4.70
Actual            : 1.2.840.10008.1.2.1
```
1.2.840.10008.1.2.1은 Explicit VR Little Endian입니다. 즉, 출력 파일이 아직 비압축 Transfer Syntax로 저장된 상태입니다.


## 9. OpenCV Native 라이브러리 로딩 문제

dcm4che-imageio-opencv를 추가해도 Native 라이브러리가 자동으로 로드되는 것은 아닙니다. JPEG 압축 과정에서는 OpenCV Native 라이브러리가 필요하므로 java.library.path 설정 또는 배포 환경 구성이 필요합니다.


## 10. 운영 환경에서 고려할 점

dcm4che를 사용하면 Transcoder를 통해 DICOM 파일의 Transfer Syntax를 비교적 간단하게 변경할 수 있습니다.

JPEG Lossless 압축에서 중요한 설정은 다음 두 가지입니다.
```
transcoder.setIncludeFileMetaInformation(true);
transcoder.setDestinationTransferSyntax(UID.JPEGLosslessSV1);
```
UID.JPEGLosslessSV1은 `1.2.840.10008.1.2.4.70`에 해당하는 DICOM 표준 JPEG Lossless Transfer Syntax입니다. 원본 픽셀 값을 보존하면서 파일 크기를 줄일 수 있기 때문에 의료 영상 저장이나 PACS 전송 시 유용하게 사용할 수 있습니다.

다만 DICOM 압축은 단순한 파일 변환이 아닙니다. Pixel Data 존재 여부, Native Codec 로딩, PACS/Viewer의 Transfer Syntax 지원 여부까지 함께 확인해야 안정적으로 사용할 수 있습니다.