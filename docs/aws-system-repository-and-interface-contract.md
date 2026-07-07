# AE-Sentinel 저장소 구조 및 인터페이스 계약

> 기준 문서:
>
> - [ae_sentinel_recommended_architecture.md](/Users/mac/Desktop/Coding/_aipbl/docs/ae_sentinel_recommended_architecture.md)
> - [pnu-cloud-ai-pbl-proposal-parsed.md](/Users/mac/Desktop/Coding/_aipbl/docs/pnu-cloud-ai-pbl-proposal-parsed.md)

## 1. 결정 사항

- 시스템 통합의 중심은 `Account A - Application Account`로 둔다.
- 프론트엔드, 백엔드, 엣지, ML은 분리 개발하되 `packages/contracts`를 단일 계약 소스로 사용한다.
- 실시간 경로에는 `feature event`만 올리고, 원시 파형(raw waveform)은 `S3` 적재 경로로 분리한다.
- `SageMaker Endpoint`는 반드시 `Orchestrator Lambda`만 호출한다.
- `Bedrock`은 물리 계산을 하지 않고, 결정론적 결과를 설명 문장과 UI 친화 JSON으로 가공하는 역할만 담당한다.

## 2. 권장 저장소 구조

아래 구조는 3인 팀이 병렬 개발하면서도 계약 충돌을 줄이기 위한 기준안이다.

```text
docs/
  ae_sentinel_recommended_architecture.md
  pnu-cloud-ai-pbl-proposal-parsed.md
  aws-system-repository-and-interface-contract.md

apps/
  web-dashboard/                  # React + Amplify

backend/
  application-api/               # API Gateway + Lambda handlers
    src/
      handlers/
      services/
      repositories/
      clients/
      models/
    tests/

edge/
  esp32-firmware/                # ESP32 sensor sampling, FFT, MQTT publish
  publisher-simulator/           # 발표/로컬 검증용 이벤트 발생기

ml/
  training-pipeline/             # SageMaker processing/training/registry
  endpoint-runtime/              # inference container/entrypoint spec
  prompt-assets/                 # Bedrock prompt templates

packages/
  contracts/
    schemas/
      common/
      iot/
      ml/
      app/
      websocket/
    examples/

infra/
  application-account/           # Amplify, API GW, Lambda, DynamoDB, Cognito, SNS
  iot-account/                   # IoT Core, IoT Rule, device policy, raw S3
  ml-account/                    # SageMaker, model registry, endpoint, ML S3

scripts/
  dev/
  validation/
```

## 3. 팀원-디렉토리 소유권

| 팀원 | 주 소유 디렉토리 | 책임 범위 |
| --- | --- | --- |
| 김태훈 | `apps/web-dashboard`, `backend/application-api`, `infra/application-account` | React, Amplify, API Gateway, Lambda API, DynamoDB, Cognito, SNS, 실시간 UI |
| 김세엽 | `edge/esp32-firmware`, `edge/publisher-simulator`, `infra/iot-account` | 센서 샘플링, FFT/특징 추출, MQTT publish, IoT Core/Rule |
| 서강재 | `ml/training-pipeline`, `ml/endpoint-runtime`, `ml/prompt-assets`, `infra/ml-account` | SageMaker 학습·배포, 모델 규격, 물리 엔진 입력 규격, Bedrock 프롬프트 |
| 공통 | `packages/contracts`, `docs` | JSON Schema, 예제 payload, 인터페이스 합의 문서 |

## 4. 인터페이스 원칙

- 모든 JSON 필드는 `camelCase`를 사용한다.
- 시간은 UTC ISO 8601 문자열로 통일한다. 예: `2026-07-07T07:00:00.000Z`
- 단위가 있는 수치는 필드명에 단위를 포함한다. 예: `sampleRateHz`, `tensionEstimateKn`, `radiusMeters`
- enum 값은 대문자 문자열로 통일한다. 예: `NORMAL`, `CAUTION`, `DANGER`
- 계약 변경 시 `schemaVersion`을 반드시 올린다.
- 중복 처리 기준 키는 `eventId`이며, 저장 기준 trace 키는 `traceId`다.

## 5. 공통 이벤트 Envelope

IoT -> Backend, Backend 내부 이벤트, WebSocket 메시지는 공통 envelope를 갖는다.

```json
{
  "schemaVersion": "1.0.0",
  "eventType": "ae.feature.detected",
  "eventId": "evt-20260707-0001",
  "traceId": "trc-20260707-0001",
  "source": {
    "account": "iot",
    "service": "iot-rule",
    "deviceId": "esp32-01"
  },
  "occurredAt": "2026-07-07T07:00:00.000Z",
  "sentAt": "2026-07-07T07:00:00.120Z",
  "payload": {}
}
```

## 6. IoT -> Backend 계약

### 6.1. 전달 방식

- `AWS IoT Rule`이 `Account A`의 `Ingestion Lambda`를 cross-account invoke 한다.
- 실시간 경로에는 `features` 중심 payload만 전달한다.
- 원시 센서 버퍼는 필요 시 `Account B S3`에 보관하고, payload에는 `rawObjectKey`만 남긴다.

### 6.2. Payload 스키마

```json
{
  "schemaVersion": "1.0.0",
  "eventType": "ae.feature.detected",
  "eventId": "evt-20260707-0001",
  "traceId": "trc-20260707-0001",
  "source": {
    "account": "iot",
    "service": "iot-rule",
    "deviceId": "esp32-01"
  },
  "occurredAt": "2026-07-07T07:00:00.000Z",
  "sentAt": "2026-07-07T07:00:00.120Z",
  "payload": {
    "siteId": "yard-a",
    "berthId": "berth-01",
    "lineId": "mooring-03",
    "windowMs": 500,
    "sampleRateHz": 20000,
    "sequenceNo": 4182,
    "features": {
      "rms": 0.18,
      "peakAmplitude": 0.76,
      "energy": 14.2,
      "dominantFrequencyHz": 4230,
      "riseTimeMs": 8.4,
      "durationMs": 41.0
    },
    "context": {
      "materialType": "synthetic-rope",
      "tensionEstimateKn": 145,
      "temperatureC": 26.1,
      "humidityPct": 78
    },
    "rawObjectKey": "raw/yard-a/berth-01/mooring-03/2026/07/07/evt-20260707-0001.bin"
  }
}
```

### 6.3. `Ingestion Lambda` 책임

- JSON Schema 검증
- 필수 필드 누락 검사
- 수치 범위 검사
- `eventId` 기반 idempotency 검사
- 검증 성공 시 `ValidatedFeatureEvent`로 내부 전달

## 7. Backend 내부 Lambda 호출 규약

### 7.1. 권장 Lambda 구성

| Lambda | 트리거 | 역할 |
| --- | --- | --- |
| `app-ingest-iot-event` | IoT Rule | IoT 이벤트 검증, 중복 제거, 내부 호출 시작 |
| `app-run-inference` | Lambda Invoke | SageMaker 요청, 위험도 판정, 후속 흐름 조정 |
| `app-calculate-snapback` | Lambda Invoke | `DANGER` 시 폴리곤/반경 계산 |
| `app-dispatch-alert` | Async Invoke | SNS 발송 및 알림 이력 저장 |
| `app-websocket-broadcast` | Async Invoke | 실시간 상태 push |
| `app-query-lines` | API Gateway REST | 계류삭 최신 상태 조회 |
| `app-query-events` | API Gateway REST | 이벤트 이력 조회 |
| `app-ack-event` | API Gateway REST | 경보 확인 처리 |

### 7.2. 호출 규칙

- `Ingestion -> Orchestrator`: `RequestResponse`
- `Orchestrator -> SageMaker Endpoint`: 동기 호출
- `Orchestrator -> Snapback`: `RequestResponse`, `riskLevel == DANGER`일 때만 호출
- `Orchestrator -> Alert/WebSocket`: `Event` 비동기 호출
- `Query/Command REST Lambda`는 IoT·ML 서비스에 직접 접근하지 않고 `DynamoDB` 기준으로 응답한다.

### 7.3. 내부 호출 Payload

```json
{
  "schemaVersion": "1.0.0",
  "eventType": "ae.feature.validated",
  "eventId": "evt-20260707-0001",
  "traceId": "trc-20260707-0001",
  "validatedAt": "2026-07-07T07:00:00.180Z",
  "payload": {
    "siteId": "yard-a",
    "berthId": "berth-01",
    "lineId": "mooring-03",
    "features": {
      "rms": 0.18,
      "peakAmplitude": 0.76,
      "energy": 14.2,
      "dominantFrequencyHz": 4230,
      "riseTimeMs": 8.4,
      "durationMs": 41.0
    },
    "context": {
      "materialType": "synthetic-rope",
      "tensionEstimateKn": 145,
      "temperatureC": 26.1,
      "humidityPct": 78
    }
  }
}
```

## 8. Backend -> ML 계약

### 8.1. 요청 규격

`SageMaker Endpoint` 입력은 최대한 작고 안정적으로 유지한다. 학습/추론에 직접 쓰는 필드만 보낸다.

```json
{
  "schemaVersion": "1.0.0",
  "requestId": "req-20260707-0001",
  "traceId": "trc-20260707-0001",
  "eventId": "evt-20260707-0001",
  "modelInput": {
    "peakAmplitude": 0.76,
    "energy": 14.2,
    "dominantFrequencyHz": 4230,
    "riseTimeMs": 8.4,
    "durationMs": 41.0,
    "tensionEstimateKn": 145,
    "temperatureC": 26.1,
    "humidityPct": 78
  }
}
```

### 8.2. 응답 규격

```json
{
  "schemaVersion": "1.0.0",
  "requestId": "req-20260707-0001",
  "traceId": "trc-20260707-0001",
  "riskLevel": "DANGER",
  "riskScore": 0.962,
  "classProbabilities": {
    "NORMAL": 0.012,
    "CAUTION": 0.026,
    "DANGER": 0.962
  },
  "modelVersion": "ae-model-v3",
  "inferenceMs": 42
}
```

### 8.3. ML 팀 고정 책임

- `modelInput` 필드명은 ML 팀이 임의 변경하지 않는다.
- 입력 특징량이 바뀌면 `schemaVersion`과 `modelVersion`을 함께 올린다.
- 추론 응답에는 반드시 `riskLevel`, `riskScore`, `classProbabilities`, `modelVersion`이 포함된다.

## 9. Backend -> Frontend REST / WebSocket 규약

### 9.1. REST 경로

모든 외부 API는 `/v1` prefix를 사용한다.

| Method | Path | 설명 |
| --- | --- | --- |
| `GET` | `/v1/lines` | 전체 계류삭 최신 상태 조회 |
| `GET` | `/v1/lines/{lineId}` | 특정 계류삭 상세 조회 |
| `GET` | `/v1/events` | 위험·주의 이벤트 목록 조회 |
| `GET` | `/v1/events/{eventId}` | 이벤트 상세 조회 |
| `GET` | `/v1/dashboard/summary` | 대시보드 요약 통계 |
| `POST` | `/v1/alerts/test` | 테스트 경보 생성 |
| `POST` | `/v1/events/{eventId}/ack` | 경보 확인 처리 |

### 9.2. REST 응답 Envelope

```json
{
  "requestId": "req-20260707-1001",
  "timestamp": "2026-07-07T07:00:01.210Z",
  "data": {},
  "error": null
}
```

에러 응답:

```json
{
  "requestId": "req-20260707-1002",
  "timestamp": "2026-07-07T07:00:01.350Z",
  "data": null,
  "error": {
    "code": "LINE_NOT_FOUND",
    "message": "lineId mooring-03 was not found"
  }
}
```

### 9.3. WebSocket 메시지 규격

```json
{
  "type": "risk.danger.detected",
  "version": "1.0.0",
  "traceId": "trc-20260707-0001",
  "sentAt": "2026-07-07T07:00:01.220Z",
  "data": {
    "lineId": "mooring-03",
    "riskLevel": "DANGER",
    "riskScore": 0.962,
    "snapbackZone": {
      "radiusMeters": 18.4,
      "polygon": [
        { "x": 132.4, "y": 84.2 },
        { "x": 171.6, "y": 95.7 },
        { "x": 158.1, "y": 131.8 }
      ]
    },
    "alertMessage": "3번 계류삭 파단 위험. 표시된 위험구역 밖으로 즉시 대피하십시오."
  }
}
```

권장 이벤트 타입:

- `line.status.updated`
- `risk.caution.detected`
- `risk.danger.detected`
- `snapback.zone.updated`
- `alert.sent`
- `device.offline`

## 10. `packages/contracts`에 먼저 고정할 파일

1주차에 아래 스키마부터 먼저 고정하는 것을 권장한다.

- `schemas/common/event-envelope.v1.json`
- `schemas/iot/ae-feature-event.v1.json`
- `schemas/iot/validated-feature-event.v1.json`
- `schemas/ml/inference-request.v1.json`
- `schemas/ml/inference-response.v1.json`
- `schemas/app/line-status.v1.json`
- `schemas/app/event-detail.v1.json`
- `schemas/websocket/ws-risk-message.v1.json`

예제 payload는 `packages/contracts/examples`에 저장한다.

## 11. 구현 우선순위

1. `packages/contracts` 스키마와 example JSON부터 확정한다.
2. `edge/publisher-simulator`로 IoT payload를 먼저 재현한다.
3. `backend/application-api`에서 `Ingestion -> Orchestrator -> SageMaker` 최소 경로를 붙인다.
4. `DynamoDB` 저장 스키마와 `GET /v1/lines`, `GET /v1/events`를 구현한다.
5. 이후 `WebSocket`, `SNS`, `Bedrock`, `Cognito`를 순차적으로 붙인다.

## 12. 지금 시점의 핵심 합의

- IoT 팀의 완료 기준은 `ae.feature.detected` 계약을 깨지 않고 `Ingestion Lambda`까지 전달하는 것이다.
- ML 팀의 완료 기준은 `inference-request.v1` 입력에 대해 `inference-response.v1`을 안정적으로 반환하는 것이다.
- 애플리케이션 팀의 완료 기준은 위 두 계약을 받아 저장, 알림, 조회, 시각화까지 end-to-end로 연결하는 것이다.
