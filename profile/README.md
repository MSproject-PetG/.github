# PetG 🐾 — HomeCam 기반 반려동물 훈련 보조 서비스

로컬(노트북 웹캠)로 촬영되는 영상을 웹에서 확인하고, **사람 + 반려동물(강아지) 동시 감지 → 사람 자세(앉음/서있음) 판정**을 통해 훈련 단계를 안내하는 프로젝트입니다.

- Front는 **HTTPS(Azure App Service)** 로 배포
- 로컬 카메라 서버는 **HTTP + 포트포워딩** 환경이라, HTTPS 프론트에서 직접 접근 시 **Mixed Content** 문제가 발생
- 이를 해결하기 위해 **Relay(중계) 서버(HTTPS)** 가 프론트 요청을 받아 로컬 카메라 서버로 **HTTP 프록시**합니다.

> ⚠️ 참고: 문서/아키텍처에 Azure ML이 등장할 수 있으나, **최종 서비스 운영에서는 Azure ML을 사용하지 않았습니다.**  
> (초기 PoC/정리 단계에서만 검토·기록이 존재)

---

## Architecture

> 리포지토리에 이미지 추가 후 경로만 맞춰주세요.

![PetG Architecture](../images/architecture.jpg)

---

## Key Features

- **실시간 영상 스트리밍(MJPEG)**: 로컬 카메라 서버가 `/video`로 MJPEG 스트림 제공
- **촬영/훈련 모드 제어**: 프론트 → Relay(HTTPS) → 로컬 카메라 서버(HTTP)
- **훈련 상태 머신(AI 서버)**
  - `detect` 모드: 사람 + 강아지가 **동시에 감지**되면 `pose`로 전환
  - `pose` 모드: 사람/강아지 중 하나라도 일정 횟수 누락되면 `detect`로 복귀(리셋)
- **자세 판정(앉음/서있음)**
  - YOLO Pose keypoint(hip/ankle) Y좌표 차이 기반 임계값으로 판정
- **로그인/회원가입(Auth + MySQL + JWT)**: 회원가입/로그인 API 구축

---

## Components

### 1) Front (React) — Azure App Service
- React SPA 배포
- App Service Deployment Center를 통해 GitHub 연동(워크플로우/파이프라인 구성)
- 배포 시 `npm test` 단계에서 테스트 파일 미존재 오류가 나면 `--passWithNoTests`로 처리
- App Service는 기본적으로 Node 앱처럼 실행하려 하므로, CRA 정적 빌드 실행을 위해 Startup Command를 명시:
  - `serve -s build`

### 2) AI Server (Flask + Ultralytics YOLO) — Azure VM
- `/upload`로 프레임 업로드 받음
- 객체 탐지 모델(`yolo11n.pt`)로 사람(person=0) + 강아지(dog=16) 동시 여부 확인
- 조건 만족 시 포즈 추정 모델(`yolo11n-pose.pt`)로 자세 판정
- `/pose-result`로 프론트에 “현재 단계 안내 문구” 제공

### 3) Relay Server (Flask Proxy) — Azure VM
- HTTPS 프론트 요청을 받아 로컬 카메라 서버(HTTP)로 프록시
- `/control`, `/pose-result`, `/video` 요청을 그대로 전달/중계

### 4) Local Camera Server (Flask + OpenCV) — Local + Port Forwarding
- 웹캠 캡처 & MJPEG 스트리밍 제공
- 훈련 모드에서는 프레임을 AI 서버로 주기 전송 (`https://<ai-domain>/upload`)

### 5) Auth Server (Flask + MySQL + JWT) — Azure VM
- `/register`: 회원가입(비밀번호 해시 저장)
- `/login`: 로그인(JWT 발급)
- (문서상 `/profile` 테스트 API 계획이 있으나, 현재 코드 기준으로는 `/register`, `/login`이 핵심)

---

## Data Flow

### HomeCam Mode (영상 시청)
1. Front에서 “스트리밍 시작” → Relay `/control`
2. Relay가 로컬 카메라 서버 `/control`로 전달
3. Front는 Relay `/video`를 `<img src="...">`로 구독해 MJPEG 시청

### Training Mode (훈련)
1. Front에서 “훈련 시작” → 카메라 서버가 프레임을 주기적으로 AI 서버 `/upload`에 전송
2. AI 서버는 `detect → pose` 상태 전환 후 자세 판정 수행
3. Front는 주기적으로 `/pose-result`를 폴링해 안내 문구를 표시

---

## API Spec

### AI Server (`https://<ai-domain>`)
#### `POST /upload`
- `multipart/form-data`
  - `frame`: JPEG 이미지 파일
- 응답 예시
  - 감지 대기: `{ "ready": false, "message": "사람 또는 동물 없음" }`
  - 준비 완료: `{ "ready": true, "message": "훈련 준비 완료" }`
  - 포즈 판정: `{ "pose": "앉음" | "서 있음" | "판단 불가" | "감지 실패" }`
  - 리셋: `{ "reset": true, "message": "사람 또는 동물 사라짐 → 대기 상태 전환" }`

#### `GET /pose-result`
- 응답 예시
  ```json
  {
    "result": "강아지와 사람을 한 화면에 나오게 해주세요!",
    "message": "객체 탐지 중",
    "num": -4
  }
