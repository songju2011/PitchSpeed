# 구속 측정기 (PitchSpeed)

고속 촬영으로 야구공을 자동 추적해서 구속을 km/h · mph로 보여주는 안드로이드 앱.
Kotlin + Jetpack Compose + Camera2. 서버·외부 API 없음, OpenCV 없음(순수 Kotlin).

## 빌드

**Android Studio**: 이 폴더를 열고(Open) Gradle 동기화가 끝나면 ▶ Run.
(Android Studio Ladybug 이상, JDK 17. 처음 한 번은 Gradle 8.9와 라이브러리를 내려받는다.)

**PC 없이**: 이 폴더를 GitHub 저장소에 올리면 `.github/workflows/build-apk.yml`이
자동으로 빌드한다. Actions 탭 → 실행 결과 → Artifacts → `PitchSpeed-debug-apk`.

## 폴더 구조

```
app/src/main/java/com/pitchspeed/app/
├─ MainActivity.kt                화면 이동
├─ camera/
│  ├─ CaptureConfig.kt            고속 해상도/fps 조회 → 최대 fps 선택
│  ├─ HighSpeedCameraController.kt 고속 세션, 셔터 고정, SENSOR_TIMESTAMP 수집
│  ├─ EncoderRingBuffer.kt        H.264 링 버퍼(최근 8초) → 클립 저장
│  └─ MotionDetector.kt           움직임 감지(자동 녹화 시작)
├─ analysis/
│  ├─ VideoFrameReader.kt         영상 → YUV 프레임
│  ├─ BallDetector.kt             프레임 차분 + 크기·원형도 필터 + 서브픽셀 중심
│  ├─ Kalman1D.kt, BallTracker.kt 칼만 필터 추적
│  ├─ PolyFit.kt, SpeedCalculator.kt 직선/포물선 피팅, 이상치 제거, 게이트 보간, 오차
│  ├─ TimestampMatcher.kt         녹화 프레임 ↔ 센서 타임스탬프(ns) 짝짓기
│  ├─ ReplayFrameCache.kt         슬로모션 리플레이용 프레임 이미지
│  ├─ PitchAnalyzer.kt            분석 전체 순서
│  └─ Types.kt                    Calibration, LensModel 등
├─ data/                          Room 기록, JSON 저장, 갤러리 영상 불러오기
└─ ui/                            화면들 (홈·가이드·보정·측정·분석·결과·기록)
```

## 동작 방식에서 알아둘 점

- **공 추적은 녹화 직후에 한다.** 고속 촬영 세션은 YUV 프레임을 실시간으로 내주지 않는다
  (미리보기와 인코더 Surface만 허용). 그래서 링 버퍼로 녹화 → 클립 저장 → 디코딩한
  YUV 프레임을 백그라운드에서 분석하는 순서다. 5초짜리 240fps 클립이면 수십 초 걸릴 수 있다.
- **셔터 수동 고정은 기기마다 다르다.** 고속 세션에서 수동 노출을 무시하는 기기가 많다.
  앱은 수동 고정을 걸어 본 뒤 실제 적용됐는지 확인하고, 안 되면 자동 노출(-1EV) + AE 잠금으로
  바꾼다. 화면 왼쪽 위에 실제 셔터 속도와 어떤 방식인지 나온다.
- **240fps가 안 나오는 기기가 있다.** 기본 카메라 앱은 슬로모션이 돼도 다른 앱에는
  120fps까지만 열어 주거나 아예 고속 촬영을 안 열어 주는 기기가 있다(특히 일부 삼성).
  그때는 가능한 최대 fps로 찍고 화면에 표시한다.
- **렌즈 왜곡 보정**은 기기가 LENS_DISTORTION 값을 줄 때만 보정 화면에 스위치가 나온다.
  기기가 스스로 왜곡을 펴 주는 모드가 있으면 그걸 켜고 앱에서는 다시 펴지 않는다.
- **예상 오차(±)**는 거리 보정(기준선 양 끝 탭 오차) + 시간(타임스탬프) + 궤적 피팅 잔차를
  합친 2시그마 값이다. 공이 기준선과 다른 거리(깊이)로 날아가서 생기는 오차는 포함되지 않는다.
- **갤러리 영상**은 파일에 기록된 시간으로 계산한다. 슬로모션 영상은 불러올 때
  실제 촬영 fps를 고를 수 있다. 일부 구간만 슬로모션인 영상(삼성 슬로모션 편집본 등)은 맞지 않는다.
