# AE-Sentinel 저장소 구조 및 인터페이스 계약

> 기준 문서:
>
> - [ae_sentinel_recommended_architecture.md](/Users/mac/Desktop/Coding/_aipbl/docs/docs/ae_sentinel_recommended_architecture.md)
> - [pnu-cloud-ai-pbl-proposal-parsed.md](/Users/mac/Desktop/Coding/_aipbl/docs/docs/pnu-cloud-ai-pbl-proposal-parsed.md)

## 1. 결정 사항

- 시스템 통합의 중심은 `Application Backend`로 둔다. (IoT · 백엔드 · ML은 **단일 팀 계정**을 공유하며, 이 구분은 계정 경계가 아니라 동일 계정 내 도메인 경계다.)
- 프론트엔드, 백엔드, 엣지, ML은 분리 개발하되 `docs/packages/contracts`를 단일 계약 소스로 사용한다.
- 실시간 경로에는 `feature event`만 올리고, 원시 파형(raw waveform)은 `S3` 적재 경로로 분리한다.
- `SageMaker Endpoint`는 반드시 `Orchestrator Lambda`만 호출한다.
- `Bedrock`은 물리 계산을 하지 않고, 결정론적 결과를 설명 문장과 UI 친화 JSON으로 가공하는 역할만 담당한다.
- **AE 센서는 안벽 볼라드(계선주)에 고정**된다(2026-07 확정). `Bollard`는 영구 자산이고, 각 계류삭(`Line`)의 `bollardId`는 불변 귀속처, `vesselId`는 접안 세션 동안만 붙는 시간제 연결이다. 항만 안전 기준으로 **하나의 볼라드에는 최대 2개 라인**만 묶을 수 있고(초과 시 `BOLLARD_FULL`), **미계선(접안 선박 없음) 라인은 항상 `NORMAL`**이다. 계류삭 생성 시 `lineId`를 생략하면 서버가 `mooring-<날짜>-<hex>`로 자동 생성한다.
- 관제 대시보드는 안벽 도면 대신 실제 지도/디지털 트윈 위에 선박·볼라드를 표시한다. **볼라드 클릭 → 볼라드 상세**(연결 라인·수정/삭제·빈 볼라드 센서 매핑), **계류삭 클릭 → 계류삭 상세**(실시간 상태 + 3D 홋줄 + 매핑된 라인의 초음파(AE) 수치)를 보여준다. AE 수치의 표시 스키마는 §8.4 학습 데이터셋과 정합을 유지한다(현재 프론트 `LineDetail`은 구 AE-Damage-Detection 스키마 표시를 유지 중이며, 홋줄 AE 스키마로의 전환은 진행 예정 — §8.4 참조).

## 2. 권장 저장소 구조

아래 구조는 3인 팀이 병렬 개발하면서도 계약 충돌을 줄이기 위한 기준안이다.

```text
docs/
  docs/
    ae_sentinel_recommended_architecture.md
    pnu-cloud-ai-pbl-proposal-parsed.md
    aws-system-repository-and-interface-contract.md
  packages/
    contracts/
      schemas/
      examples/

backend/
  application-api/               # API Gateway + Lambda handlers
    src/
      handlers/
      services/
      repositories/
      clients/
      models/
    tests/
  infra/
    application-account/

frontend/
  public/
  src/
    app/
    components/
    features/
  tests/

edge-iot/
  esp32-firmware/                # ESP32 sensor sampling, FFT, MQTT publish
  publisher-simulator/           # 발표/로컬 검증용 이벤트 발생기
  infra/
    iot-account/

ml/
  training-pipeline/             # SageMaker processing/training/registry
  endpoint-runtime/              # inference container/entrypoint spec
  prompt-assets/                 # Bedrock prompt templates
  infra/
    ml-account/

scripts/
  dev/
  validation/
```

## 3. 팀원-디렉토리 소유권

| 팀원 | 주 소유 디렉토리 | 책임 범위 |
| --- | --- | --- |
| 김태훈 | `frontend`, `backend/application-api`, `backend/infra/application-account` | React, Amplify, API Gateway, Lambda API, DynamoDB, Cognito, SNS, 실시간 UI |
| 김세엽 | `edge-iot/esp32-firmware`, `edge-iot/publisher-simulator`, `edge-iot/infra/iot-account` | 센서 샘플링, FFT/특징 추출, MQTT publish, IoT Core/Rule |
| 서강재 | `ml/training-pipeline`, `ml/endpoint-runtime`, `ml/prompt-assets`, `ml/infra/ml-account` | SageMaker 학습·배포, 모델 규격, 물리 엔진 입력 규격, Bedrock 프롬프트 |
| 공통 | `docs/packages/contracts`, `docs/docs` | JSON Schema, 예제 payload, 인터페이스 합의 문서 |

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

- `AWS IoT Rule`이 `Ingestion Lambda`를 동일 계정 invoke 한다 (Lambda Permission).
- 실시간 경로에는 `features` 중심 payload만 전달한다.
- 원시 센서 버퍼는 필요 시 `IoT 도메인 S3`에 보관하고, payload에는 `rawObjectKey`만 남긴다.

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

### 6.4. 압전(피에조) 센서 경로 — 진동수 전용 입력

실제 계류삭 센서는 **피에조 압전 센서로 구조 진동수(Hz)만 계측**한다. 이 입력을 기존 AE
특징량 계약에 맞추기 위해 별도 어댑터 Lambda(`app-ingest-piezo`, `handlers.piezo`)를 둔다.
예제: `packages/contracts/examples/iot-piezo-frequency-event.v1.json`.

- 센서는 MQTT 토픽 `aesentinel/<site>/piezo` 로 최소 리딩을 publish한다(별도 IoT Rule → 어댑터).

```json
{ "schemaVersion": "1.0.0", "eventType": "piezo.frequency.detected",
  "eventId": "evt-piezo-0001", "occurredAt": "2026-07-11T09:00:00.000Z",
  "payload": { "siteId": "yard-a", "berthId": "berth-02", "lineId": "mooring-05", "freqHz": 28.5 } }
```

- 어댑터는 **taut-cable 관계(장력 T ∝ f²)** 로 진동수를 정규화 장력비 `ρ∈[0,1]`(슬랙 기준
  `FREQ_SLACK_HZ`≈8Hz ~ 파단 임박 `FREQ_CRIT_HZ`≈32Hz)로 매핑하고, 이를 AE 특징량
  (`amplitudeDbAe`·`snrDb`·`hitCount`·`durationMs`·`signalType`)으로 단조 변환하여
  `ae-feature-event.v2`를 생성한 뒤 **동일한 검증→추론(SageMaker)→스냅백→알림 파이프라인**에 태운다.
- 실측 진동수와 장력비는 이벤트 `context`(`sensorKind:"piezo"`, `structuralFreqHz`, `loadRatio`)에
  보존되어, 대시보드 계류삭 상세가 "측정 진동수(Hz) · 추정 장력(%MBL)"로 우선 표시하고 AE 수치는
  "진동수 기반 추정"으로 함께 보여준다.
- 결과적으로 프론트·모델·SNS·WebSocket을 바꾸지 않고 진동수 전용 하드웨어를 그대로 수용한다.
  `FREQ_SLACK_HZ`/`FREQ_CRIT_HZ`는 배포 시 env로 조정하며, 라인별 캘리브레이션(슬랙/임계 진동수
  실측)은 후속 과제다.

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

### 8.4. 학습 데이터셋 (홋줄 AE — 논문 기반 → 항만 도메인 변환)

ML 모델은 **계류삭 AE 파단 실측 논문(Bashir et al. 2017, *Applied Acoustics* 121)**
을 근거로 한 합성 데이터셋으로 학습한다. 근거·평가·검증은 별도 문서에 있다:
[홋줄 AE 데이터셋 평가](./rope-ae-dataset-evaluation.md) ·
[항만 볼라드 AE 데이터셋 카드](./port-ae-dataset.md) ·
[데이터셋 신뢰성 검증 보고서](./dataset-reliability-report.md).

관측 가능한(배포 시 센서에서 얻는) 특징 스키마 — 항만 도메인 변환본 기준:

| 특징 | 단위/범위 | 설명 |
|---|---|---|
| `Amplitude_dB_AE` | dB re 1 μV | 접촉식 AE 수신 진폭 |
| `Freq_Low_kHz` / `Freq_High_kHz` | kHz | 신호 주파수 대역 하한/상한 |
| `Duration_ms` | ms | 신호 지속시간 |
| `Hit_Count` | 회 | 단일 버스트 내 충격 횟수 (위험 단계의 강한 피처) |
| `SNR_dB` | dB | 수신 진폭 − 배경 소음 플로어 |
| `Ambient_Noise_dB` | dB | 배경 소음 플로어(크레인·강우·바람 반영) |
| `Rain_mmh` / `Wind_mps` / `Crane_Active` | 환경 공변량 | 강우량·풍속·크레인 가동 |
| `Temperature_C` / `Humidity_pct` | °C / % | 온·습도 |
| `Rope_Diameter_mm` / `Rope_MBL_kN` | mm / kN | 계류삭 규격 |

분류 라벨(`DamageType`)과 위험도 매핑:

| 라벨 | 하중 단계 | 위험도 |
|---|---|---|
| 안전 (Safe) | < 50% MBL | `NORMAL` |
| 주의 (Caution) | 50–70% MBL | `CAUTION` |
| 위험 (Danger) | ≥ 70% MBL | `DANGER` |

> 모델 입력은 개별 이벤트가 아니라 **직전 N개 이벤트의 시계열 윈도 집계**를 사용한다
> (신호군 조성·이벤트율·진폭/히트 통계). 하중비·절대시간·시험기 좌표는 라벨 누수 및
> 배포 불가 정보이므로 학습 피처에서 제외한다.
>
> ⚠️ **마이그레이션 진행 중**: 현재 프론트 `LineDetail`의 AE 수치 표시는 구
> AE-Damage-Detection 스키마(`lib/acoustic.ts` 합성값)를 유지하고 있으며, 위 홋줄 AE
> 스키마로의 전환과 실 센서/ML 특징량 push 연동은 예정 작업이다. 이전 proxy 접근의
> 상세 스키마는 [보관 문서](./archive/README.md)에 있다.

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
| `GET` | `/v1/vessels` | 전체 선박 조회 |
| `POST` | `/v1/vessels` | 선박 생성 |
| `GET` | `/v1/vessels/{vesselId}` | 선박 상세 조회 |
| `PUT` | `/v1/vessels/{vesselId}` | 선박 수정 (부분 수정) |
| `DELETE` | `/v1/vessels/{vesselId}` | 선박 삭제 |
| `POST` | `/v1/vessels/{vesselId}/berthing` | 접안 처리 (해당 선석 계류삭 자동 귀속) |
| `POST` | `/v1/vessels/{vesselId}/departure` | 출항 처리 (계류삭 매핑 해제 · NORMAL 리셋) |
| `GET` | `/v1/bollards` | 전체 볼라드 조회 |
| `POST` | `/v1/bollards` | 볼라드 생성 |
| `GET` | `/v1/bollards/{bollardId}` | 볼라드 상세 조회 |
| `PUT` | `/v1/bollards/{bollardId}` | 볼라드 수정 (부분 수정) |
| `DELETE` | `/v1/bollards/{bollardId}` | 볼라드 삭제 |

> 위 표는 핵심 경로 요약이다. 계류삭·이벤트 생성/수정/삭제(CUD)와 경보 조회/삭제(RD)를
> 포함한 **전체 CRUD 표면**과 요청/응답 스키마는 `docs/packages/contracts/openapi/ae-sentinel-rest.v1.yaml`
> (배포된 Swagger UI `/docs`)에 정의되어 있으며 이 문서와 동기화된다.

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

### 9.4. 선박(Vessel) · 볼라드(Bollard) · 계류삭 연결

AE 센서는 안벽 볼라드(계선주)에 고정된다. 각 계류삭(`LineStatus`)은 영구 귀속처
`bollardId`(불변)와 접안 세션 동안만 붙는 `vesselId`(시간제)를 가진다. 대시보드 지도는
선박의 `lat`/`lon`으로 마커를 표시하고, 마커 색은 그 선박에 속한 계류삭 중 가장 높은
위험도를 따른다.

선박(Vessel) 레코드:

```json
{
  "vesselId": "vessel-01",
  "name": "HYUNDAI FORWARD",
  "siteId": "yard-a",
  "berthId": "berth-01",
  "imo": "9812345",
  "vesselType": "Container",
  "lengthOverallM": 300,
  "lat": 34.7842,
  "lon": 126.517,
  "status": "BERTHED",
  "createdAt": "2026-07-08T00:00:00.000Z",
  "updatedAt": "2026-07-08T00:00:00.000Z"
}
```

- `status` enum: `BERTHED | APPROACHING | DEPARTED`
- 계류삭 상태(`GET /v1/lines`) 응답에는 소속 선박을 가리키는 `vesselId` 필드가 포함된다.
- 선박 생성 시 `vesselId`를 생략하면 서버가 자동 생성한다.

볼라드(Bollard) 레코드 — 안벽에 고정된 영구 자산:

```json
{
  "bollardId": "bollard-01",
  "siteId": "yard-a",
  "berthId": "berth-01",
  "positionM": 12.0,
  "label": "B-01",
  "twinSlot": 1,
  "createdAt": "2026-07-08T00:00:00.000Z",
  "updatedAt": "2026-07-08T00:00:00.000Z"
}
```

계류삭(Line) 라이프사이클 규칙:

- `bollardId`는 센서의 영구 귀속처(불변), `vesselId`는 접안 세션 동안만 붙는 시간제 연결이다.
- 항만 안전 기준상 **하나의 볼라드에는 최대 2개 라인**만 붙일 수 있다. 초과 시
  `BOLLARD_FULL`(HTTP 409)을 반환한다.
- **미계선(`vesselId` 없음) 라인은 항상 `NORMAL`·`riskScore=0`**이다. 출항 처리 시 해제된
  라인은 자동으로 `NORMAL`로 리셋된다.
- 계류삭 생성 시 `lineId`를 생략하면 서버가 `mooring-<날짜>-<hex>`로 자동 생성한다.
  `siteId`/`berthId`는 소속 볼라드에서 파생할 수 있다.
- 접안(`/berthing`)은 해당 선석의 미배정 라인을 선박에 자동 귀속하고, 출항(`/departure`)은
  그 선박의 모든 라인을 해제한다.

## 10. `docs/packages/contracts`에 먼저 고정할 파일

1주차에 아래 스키마부터 먼저 고정하는 것을 권장한다.

- `schemas/common/event-envelope.v1.json`
- `schemas/iot/ae-feature-event.v1.json`
- `schemas/iot/validated-feature-event.v1.json`
- `schemas/ml/inference-request.v1.json`
- `schemas/ml/inference-response.v1.json`
- `schemas/app/line-status.v1.json`
- `schemas/app/event-detail.v1.json`
- `schemas/websocket/ws-risk-message.v1.json`

예제 payload는 `docs/packages/contracts/examples`에 저장한다.

## 11. 구현 우선순위

1. `docs/packages/contracts` 스키마와 example JSON부터 확정한다.
2. `edge-iot/publisher-simulator`로 IoT payload를 먼저 재현한다.
3. `backend/application-api`에서 `Ingestion -> Orchestrator -> SageMaker` 최소 경로를 붙인다.
4. `DynamoDB` 저장 스키마와 `GET /v1/lines`, `GET /v1/events`를 구현한다.
5. 이후 `WebSocket`, `SNS`, `Bedrock`, `Cognito`를 순차적으로 붙인다.

## 12. 지금 시점의 핵심 합의

- IoT 팀의 완료 기준은 `ae.feature.detected` 계약을 깨지 않고 `Ingestion Lambda`까지 전달하는 것이다.
- ML 팀의 완료 기준은 `inference-request.v1` 입력에 대해 `inference-response.v1`을 안정적으로 반환하는 것이다.
- 애플리케이션 팀의 완료 기준은 위 두 계약을 받아 저장, 알림, 조회, 시각화까지 end-to-end로 연결하는 것이다.
