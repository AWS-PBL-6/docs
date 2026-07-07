# AE-Sentinel Git Repository Split Strategy

## 1. 목적

현재 워크스페이스는 아키텍처 설계와 초기 계약 정리를 위해 한 곳에 모아둔 상태다.  
실제 협업 단계에서는 팀 책임 경계와 배포 단위를 기준으로 Git 저장소를 분리하는 편이 맞다.

이 문서는 다음 목표를 기준으로 한다.

- 팀원별 소유권을 명확히 분리한다.
- 코드 리뷰 범위를 줄인다.
- 배포 권한과 AWS 계정 책임을 저장소 경계와 일치시킨다.
- 공통 계약 문서는 별도 저장소에서 관리한다.

## 2. 권장 전략

권장안은 `멀티 레포 + 공통 docs/contracts 저장소`다.

이 프로젝트는 단순한 프론트엔드 앱이 아니라 아래 4개 성격이 명확히 다르다.

- 웹 애플리케이션
- 애플리케이션 백엔드
- 엣지/IoT 디바이스 및 시뮬레이터
- ML 학습/서빙

여기에 공통 문서와 인터페이스 계약은 별도 협업 저장소로 관리한다.

## 3. 추천 저장소 구성

### 3.1. 반드시 분리할 저장소

| 저장소 역할 | 권장 GitHub repo | 현재 디렉토리 |
| --- | --- | --- |
| Frontend | `AWS-PBL-6/Frontend` | `apps/web-dashboard` |
| Backend | `AWS-PBL-6/Backend` | `backend/application-api`, `infra/application-account` |
| Edge / IoT | `AWS-PBL-6/Edge-IoT` | `edge/esp32-firmware`, `edge/publisher-simulator`, `infra/iot-account` |
| ML / AI | `AWS-PBL-6/ML` | `ml/training-pipeline`, `ml/endpoint-runtime`, `ml/prompt-assets`, `infra/ml-account` |
| Docs / Contracts | `AWS-PBL-6/docs` | `docs`, `packages/contracts` |

### 3.2. 선택적으로 나중에 분리할 저장소

아래는 지금 당장 분리하지 않아도 된다.

- `AWS-PBL-6/Infra`  
  이유: 현재는 각 계정 인프라가 각 도메인과 강하게 결합되어 있다.

- `AWS-PBL-6/Contracts`  
  이유: 현재 `packages/contracts`는 협업 문서 성격이 강하다.  
  다만 추후 OpenAPI/JSON Schema codegen, SDK 배포, runtime validator 패키지화가 시작되면 `docs`에서 분리하는 편이 맞다.

## 4. 왜 이렇게 나누는가

### Frontend

- React/Amplify 배포 주기가 빠르다.
- 백엔드/ML 변경 없이 독립 배포가 많다.
- UI 개발자가 가장 자주 만지는 영역이다.

### Backend

- API Gateway, Lambda, DynamoDB, Cognito, SNS가 한 묶음이다.
- 시스템 통합 중심 계정이 `Application Account`이므로 백엔드와 인프라를 함께 보는 편이 낫다.

### Edge / IoT

- ESP32 펌웨어와 MQTT publish 경로는 웹/백엔드와 릴리즈 주기가 다르다.
- 디바이스 테스트와 IoT Rule은 별도 ownership이 필요하다.

### ML / AI

- SageMaker 학습, 엔드포인트 런타임, 프롬프트 자산은 배포와 검증 방식이 완전히 다르다.
- 학습 코드와 서비스 코드 리뷰를 분리하는 편이 낫다.

### Docs / Contracts

- 모든 팀이 참조하지만 어느 한 팀 소유도 아니다.
- 인터페이스 계약은 합의 문서이므로 코드 저장소에 종속되면 안 된다.

## 5. 현재 레포를 어떻게 취급할지

현재 `/Users/mac/Desktop/Coding/_aipbl` 는 **작업용 통합 워크스페이스**로 유지하는 편이 가장 안전하다.

즉, 이 저장소는 다음 역할만 담당한다.

- 초기 구조 설계
- 디렉토리 스캐폴드
- 공통 논의
- 여러 저장소 내용을 한 번에 바라보는 로컬 작업 공간

실제 canonical source of truth 는 각 분리된 원격 저장소가 된다.

## 6. 운영 규칙

### 공통 규칙

- 각 저장소는 자신의 도메인 코드만 가진다.
- `docs/contracts` 내용은 직접 복붙하지 않고 `AWS-PBL-6/docs`를 기준 문서로 참조한다.
- 인터페이스 변경은 먼저 `AWS-PBL-6/docs`에서 합의하고, 그다음 각 구현 저장소에 반영한다.

### 브랜치 규칙

- `main`: 배포 가능 상태
- 기능 브랜치: `feat/...`
- 계약 변경 브랜치: `contract/...`
- 긴급 수정: `hotfix/...`

### PR 규칙

- 계약 스키마 변경은 반드시 관련 구현 PR과 링크한다.
- 위험도 enum, payload 필드, REST/WebSocket 응답 구조 변경은 docs repo PR이 먼저다.

## 7. 추천 생성 순서

1. `AWS-PBL-6/Backend`
2. `AWS-PBL-6/Edge-IoT`
3. `AWS-PBL-6/ML`
4. `AWS-PBL-6/Frontend`
5. `AWS-PBL-6/docs`

`Frontend`와 `docs`는 이미 생성되어 있으므로, 실질적으로는 `Backend`, `Edge-IoT`, `ML` 생성이 우선이다.

## 8. 내가 다음 단계에서 할 작업

사용자가 GitHub repo를 만든 뒤 원격 URL을 주면, 나는 아래를 수행한다.

1. 각 대상 repo에 대응하는 디렉토리 집합을 export 한다.
2. export 결과를 독립 Git repo로 초기화한다.
3. 해당 remote 를 연결한다.
4. 초기 commit 을 만든다.
5. `main` 브랜치로 push 한다.

## 9. repo 생성 요청 템플릿

아래 형식으로 repo URL만 주면 된다.

```text
Backend: https://github.com/AWS-PBL-6/Backend.git
Edge-IoT: https://github.com/AWS-PBL-6/Edge-IoT.git
ML: https://github.com/AWS-PBL-6/ML.git
Frontend: https://github.com/AWS-PBL-6/Frontend.git
Docs: https://github.com/AWS-PBL-6/docs.git
```

## 10. 주의사항

- 지금 `Frontend` 원격에는 전체 구조가 한 번 올라간 상태이므로, 장기적으로는 프론트 repo를 `apps/web-dashboard` 전용 내용으로 정리하는 편이 맞다.
- 원격 내용을 정리할지, 새 repo로 다시 시작할지는 사용자가 결정해야 한다.
- 내가 이후에 분리 push를 수행하면, 각 repo는 현재 워크스페이스의 일부만 담게 된다.
