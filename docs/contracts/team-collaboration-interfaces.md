# AE-Sentinel 팀 협업 인터페이스 계약

> 분류: 현행 협업 계약 · 팀 시작 문서

> 기준일: 2026-07-13  
> 상태: 현행(Canonical)  
> 변경 핵심: 센서 소유권 `Bollard → Mooring Line`

이 문서는 IoT·Backend·ML·Frontend·운영 담당자가 서로 넘겨주는 데이터와 책임을 한곳에서
확인하기 위한 시작점이다. 필드의 기계 검증 기준은 [`docs/packages/contracts/`](../../packages/contracts/README.md)의 JSON
Schema와 OpenAPI가 우선하며, 이 문서는 그 계약을 실제 구현 순서로 설명한다.

## 1. 한 문장 원칙

피에조 압전센서는 볼라드가 아니라 **각 계류삭의 선박 측 연결 고리(아이·스플라이스
인근)에 한 대씩 부착**하며, AWS IoT Thing ID인 `sensorId`는 하나의 `lineId`에만
전용 배정한다.

```text
AWS IoT Thing(sensorId) 1 ── 1 Mooring Line(lineId)
                                  ├── N:1 Bollard(bollardId, 안벽 연결 지점)
                                  └── N:1 Vessel(vesselId, 접안 세션 동안만)
```

- `sensorId`: 물리 센서의 영구 식별자이자 AWS IoT Thing 이름
- `lineId`: 센서가 실제 측정하는 계류삭 식별자
- `bollardId`: 계류삭이 현재 연결된 안벽 위치. 센서 소유자가 아니다.
- `vesselId`: 현재 접안 세션의 선박. 출항 시 해제된다.
- 한 볼라드에는 최대 2개 계류삭을 연결할 수 있지만, 두 계류삭은 서로 다른 센서를 쓴다.

## 2. 팀별 책임과 소스 오브 트루스

| 경계 | 제공 팀 | 소비 팀 | 소스 오브 트루스 | 책임 |
|---|---|---|---|---|
| 센서 등록/인증 | IoT | Backend·Frontend | AWS IoT Thing Registry | Thing 생성, 인증서·정책, `sensorId` 유지 |
| 센서↔계류삭 배정 | Backend | IoT·Frontend | Lines DynamoDB | 중복 배정 차단, 연동 시각 기록 |
| 진동수 원시 이벤트 | IoT | Backend | MQTT 계약 | `deviceId`, `freqHz`, 시간 전송 |
| 검증·추론 오케스트레이션 | Backend | ML·Frontend | Event/Line 테이블 | 신원 검증, 윈도 조립, 상태 저장 |
| 모델 입력/출력 | Backend | ML | inference v2/v1 스키마 | 입력 순서·응답 enum 유지 |
| 실시간 관제 메시지 | Backend | Frontend | WebSocket v1 스키마 | 위험 갱신·경보 push |
| 센서 선택 UX | Backend | Frontend | `GET /v1/sensors` | ID 직접 입력 금지, 목록 클릭 적용 |

## 3. 관리자 REST 계약

정확한 필드/응답 envelope는
[`ae-sentinel-rest.v1.yaml`](../../packages/contracts/openapi/ae-sentinel-rest.v1.yaml)을 따른다.

### 3.1 센서 인벤토리 조회

```http
GET /v1/sensors
Authorization: Bearer <Cognito ID token>
```

AWS IoT Thing Registry에 실제 존재하는 피에조 센서만 반환한다.

```json
{
  "sensorId": "piezo-mooring-05",
  "thingName": "piezo-mooring-05",
  "displayName": "5번 계류삭 피에조",
  "sensorKind": "PIEZO",
  "source": "AWS_IOT_CORE",
  "registrationStatus": "REGISTERED",
  "connectivityStatus": "CONNECTED",
  "lastSeenAt": "2026-07-13T09:00:00.000Z",
  "assignedLineId": null,
  "assignedBollardId": null,
  "assignedVesselId": null,
  "assignedBerthId": null,
  "available": true
}
```

- `available=true`: 새 계류삭에 선택 가능
- `available=false`: `assignedLineId`가 이미 소유함
- 배정 센서는 Backend lifecycle/heartbeat 상태가 `CONNECTED|DISCONNECTED`의 기준이다.
- `UNKNOWN`: 아직 lifecycle/heartbeat/측정 이력이 없는 센서이며 “오프라인”과 같지 않다.
- 프론트는 `sensorId` 텍스트 입력창을 제공하지 않고 이 응답을 목록으로 표시한다.

### 3.2 계류삭 생성과 동시에 센서 배정

```http
POST /v1/lines
Content-Type: application/json
```

```json
{
  "lineId": "mooring-05",
  "siteId": "busan",
  "berthId": "PNC-1",
  "bollardId": "bollard-05",
  "vesselId": "vessel-02",
  "sensorMode": "DEVICE",
  "sensorId": "piezo-mooring-05"
}
```

센서 설치가 아직 끝나지 않았다면 `sensorMode=PENDING`으로 생성한다. 이 상태에서는
이벤트를 수신하지 않으며 관제 화면도 측정값 대신 “센서 미연동”을 표시한다.

### 3.3 기존 계류삭에 센서 클릭 적용

```http
POST /v1/lines/mooring-05/sensor
Content-Type: application/json

{
  "sensorMode": "DEVICE",
  "sensorId": "piezo-mooring-05"
}
```

허용 상태 전이는 다음뿐이다.

```text
PENDING ──→ DEVICE
   └──────→ VIRTUAL ──→ DEVICE
```

`DEVICE→DEVICE`, 다른 계류삭이 쓰는 `sensorId`, `PENDING`으로 되돌리기는 허용하지 않는다.
계류삭을 삭제하면 그 센서는 다시 `available=true`가 된다.

### 3.4 LineStatus 핵심 응답

```json
{
  "lineId": "mooring-05",
  "siteId": "busan",
  "berthId": "PNC-1",
  "sensorMode": "DEVICE",
  "sensorId": "piezo-mooring-05",
  "sensorBoundAt": "2026-07-13T09:00:00.000Z",
  "deviceOnline": true,
  "lastSeenAt": "2026-07-13T09:00:30.000Z",
  "bollardId": "bollard-05",
  "vesselId": "vessel-02",
  "riskLevel": "NORMAL",
  "riskScore": 0.12
}
```

## 4. IoT → Backend MQTT 계약

### 4.1 토픽

```text
aesentinel/{sensorId}/piezo
```

예시:

```text
aesentinel/piezo-mooring-05/piezo
```

IoT Rule은 `topic(2)`를 `topicSensorId`로 주입한다. 메시지의 `source.deviceId`와 토픽의
`{sensorId}`가 같아야 하며, Backend는 이 값으로 현재 `LineStatus.sensorId` 배정을 조회한다.

### 4.2 피에조 원시 이벤트

스키마:
[`piezo-frequency-event.v2.json`](../../packages/contracts/schemas/iot/piezo-frequency-event.v2.json)  
예제:
[`iot-piezo-frequency-event.v2.json`](../../packages/contracts/examples/iot-piezo-frequency-event.v2.json)

```json
{
  "schemaVersion": "1.0.0",
  "eventType": "piezo.frequency.detected",
  "eventId": "evt-piezo-20260713-0001",
  "traceId": "trc-piezo-20260713-0001",
  "occurredAt": "2026-07-13T09:00:00.000Z",
  "sentAt": "2026-07-13T09:00:00.120Z",
  "source": {
    "account": "iot",
    "service": "esp32-piezo",
    "deviceId": "piezo-mooring-05"
  },
  "payload": {
    "freqHz": 28.5,
    "ambientNoiseDb": 40.0,
    "rainMmh": 0.0,
    "windMps": 3.2,
    "craneActive": 0
  }
}
```

필수 필드:

- envelope: `schemaVersion`, `eventType`, `eventId`, `traceId`, `occurredAt`, `sentAt`, `source`, `payload`
- identity: `source.deviceId`(MQTT topic의 `{sensorId}`와 동일)
- measurement: `payload.freqHz` (`> 0`, 단위 Hz)

센서는 위험도나 추정 장력을 보내지 않는다. 원시 `freqHz`만 사실값이고, 장력비·AE proxy는
Backend 피에조 어댑터가 동일 보정 설정으로 계산한다.

`lineId`, `siteId`, `berthId`는 센서 계약이 아니다. 관리자가 센서를 재배정해도 펌웨어를
바꾸지 않도록 Backend가 Lines 테이블의 최신 배정에서 세 값을 채운다.

### 4.3 온라인·오프라인 계약

```text
MQTT clientId = AWS IoT Thing 이름 = sensorId
heartbeat topic = aesentinel/{sensorId}/heartbeat
heartbeat 주기 = 30초 / offline timeout = 120초 / sweep = 1분
```

- AWS IoT `$aws/events/presence/connected/{clientId}`: 즉시 `deviceOnline=true`
- AWS IoT `$aws/events/presence/disconnected/{clientId}`: 즉시 `deviceOnline=false`
- heartbeat 또는 정상 측정 수신: `deviceOnline=true`, `lastSeenAt` 갱신
- 120초 무응답: scheduled Lambda가 `deviceOnline=false` 처리
- 모든 변경은 `line.status.updated` 또는 `device.offline` WebSocket으로 Frontend에 push

heartbeat 스키마와 예제:
[`sensor-heartbeat.v1.json`](../../packages/contracts/schemas/iot/sensor-heartbeat.v1.json),
[`iot-sensor-heartbeat.v1.json`](../../packages/contracts/examples/iot-sensor-heartbeat.v1.json)

### 4.4 Backend 신원 검증 순서

1. topic의 `{sensorId}`와 `source.deviceId`가 같은지 확인
2. `LineStatus.sensorId`로 현재 배정 계류삭 조회
3. 조회한 라인의 `lineId`·`siteId`·`berthId`를 내부 이벤트에 주입
4. event schema와 수치 범위 검증
5. `eventId` 중복 확인
6. 검증 성공 후에만 윈도 조립·ML 추론·상태 저장

| 오류 코드 | HTTP/처리 | 발생 조건 | IoT 팀 조치 |
|---|---:|---|---|
| `SENSOR_NOT_BOUND` | 409/REJECTED | 센서에 배정된 계류삭이 없음 | 관리 화면에서 센서를 계류삭에 배정 |
| `DUPLICATE_EVENT` | 409/REJECTED | 같은 `eventId` 재수신 | 정상적인 멱등 처리, 새 ID로 재측정 |
| `VALIDATION_ERROR` | 400/REJECTED | 센서 ID 불일치 또는 필드/단위/범위 오류 | topic·펌웨어 payload 수정 |

## 5. Backend 내부 변환 → ML 계약

피에조 원시 이벤트는 Backend에서 `ae.feature.detected` v2로 변환된다.

- 실제 측정값 보존: `context.sensorKind=piezo`, `context.structuralFreqHz`
- 파생값: `context.loadRatio`
- 현재 PoC 변환: `T ∝ f²`, 기본 보정 `slack=1 Hz`, `critical=6 Hz` (2026-07-15 시연 대역: 1–2 NORMAL / 3–5 CAUTION / 6+ DANGER)
- 모델 입력: 한 이벤트가 아니라 같은 `lineId`의 최근 이벤트 윈도 17개 집계 피처

계약 파일:

- [`ae-feature-event.v2.json`](../../packages/contracts/schemas/iot/ae-feature-event.v2.json)
- [`inference-request.v2.json`](../../packages/contracts/schemas/ml/inference-request.v2.json)
- [`inference-response.v1.json`](../../packages/contracts/schemas/ml/inference-response.v1.json)

ML 응답 필수값:

```json
{
  "riskLevel": "CAUTION",
  "riskScore": 0.58,
  "classProbabilities": {
    "NORMAL": 0.30,
    "CAUTION": 0.62,
    "DANGER": 0.08
  },
  "modelVersion": "port-window-xgb-v1",
  "inferenceMs": 12.4
}
```

`riskLevel`은 `NORMAL|CAUTION|DANGER`, 확률은 합계 1, `riskScore`는 0~1을 유지한다.

## 6. Backend → Frontend 실시간 계약

WebSocket 스키마:
[`ws-risk-message.v1.json`](../../packages/contracts/schemas/websocket/ws-risk-message.v1.json)

주요 타입:

- `line.status.updated`
- `sensor.reading.received` — 추론을 기다리지 않고 원시 피에조 값 즉시 전달
- `risk.caution.detected`
- `risk.danger.detected`
- `snapback.zone.updated`
- `alert.sent`
- `device.offline`

```json
{
  "type": "risk.danger.detected",
  "version": "1.0.0",
  "traceId": "trc-piezo-20260713-0001",
  "sentAt": "2026-07-13T09:00:01.000Z",
  "data": {
    "lineId": "mooring-05",
    "riskLevel": "DANGER",
    "riskScore": 0.91,
    "lastEventId": "evt-piezo-20260713-0001",
    "occurredAt": "2026-07-13T09:00:00.000Z",
    "deviceOnline": true,
    "lastSeenAt": "2026-07-13T09:00:01.000Z",
    "measurement": {
      "sensorKind": "PIEZO",
      "eventId": "evt-piezo-20260713-0001",
      "sensorId": "factorypulse_gateway_01",
      "freqHz": 28.5,
      "loadRatio": 0.7794
    }
  }
}
```

Backend는 먼저 `sensor.reading.received`로 원시값·온라인 상태를 보내고, 추론이
끝나면 `risk.*` 메시지로 판정값을 보낸다. Frontend는 `lineId`로 LineStatus를 즉시
갱신하고, `lastEventId`로 영속 이벤트를 비동기 조회한다. `measurement`는 화면에 즉시 표시할 원시 피에조 값이며,
재연결 시 REST 최신 상태로 누락 구간을 보정한다. 센서 소유권을 `bollardId`로
추론하면 안 된다.

## 7. Backend → 알림/대응 계층 계약

`CAUTION/DANGER` 확정 시 Backend가 Agent 안내를 받아 SNS 이메일·등록 SMS·WebSocket을 발행한다. 대응 가이드/Bedrock 입력에는
`lineId`, `bollardId`, `vesselId`, `siteId`, 위험도와 근거 수치를 함께 전달한다.
여기서 `bollardId`는 현장 위치 안내용이고, 센서 식별은 항상 `sensorId/lineId`로 한다.

## 8. 관리자 운영 순서

```text
1. AWS IoT Core에 센서 Thing 등록
2. 안벽 볼라드 위치 등록
3. 선박/계류삭 등록
4. 계류삭 연결 고리에 물리 센서 부착
5. 계류삭 화면에서 IoT 센서 목록 → 클릭 → 선택 센서 적용
6. 센서 펌웨어의 Thing 이름/topic/`source.deviceId` 확인
7. 테스트 이벤트 발행
8. 계류삭 상세에서 실제 freqHz 수신 확인
```

실센서가 `DEVICE`인데 이벤트가 없으면 Frontend는 목업 수치를 표시하지 않고 “수신 대기”를
표시한다. 합성값은 `VIRTUAL` 또는 공개 데모에서만 사용한다.

## 9. 기존 볼라드 센서 데이터 마이그레이션

기존 `Bollard.sensorId`는 자동으로 계류삭에 이전하지 않는다. 한 볼라드에 두 계류삭이 연결될
수 있어 어떤 로프를 측정했는지 결정할 수 없기 때문이다.

1. 기존 볼라드 센서 ID를 참고 목록으로 내보낸다.
2. 실제 센서를 계류삭 연결 고리로 옮겨 부착한다.
3. 해당 `lineId`에서 `/v1/lines/{lineId}/sensor`로 다시 배정한다.
4. 한 볼라드에 계류삭이 두 개면 센서도 두 개 준비한다.
5. topic 센서 ID로 현재 배정 `lineId`가 조회되는지 테스트한 후 기존 볼라드 센서 필드를 폐기한다.

## 10. 변경·검증 체크리스트

계약을 변경하는 팀은 한 PR에서 다음을 함께 수정한다.

- [ ] JSON Schema
- [ ] 정상/오류 예제 payload
- [ ] OpenAPI
- [ ] Backend validator와 오류 코드
- [ ] Frontend TypeScript 타입
- [ ] IoT 펌웨어/시뮬레이터 예제
- [ ] 이 문서의 흐름/표
- [ ] 기존 버전과 호환 또는 major 버전 증가 여부

필수 통합 테스트:

- [ ] 센서 목록 클릭 적용 성공
- [ ] 같은 센서를 두 계류삭에 배정하면 `SENSOR_IN_USE`
- [ ] 올바른 `deviceId+freqHz` 이벤트가 현재 배정 계류삭에 저장·추론됨
- [ ] 미배정 센서 이벤트가 `SENSOR_NOT_BOUND`
- [ ] topic과 `source.deviceId` 불일치가 `VALIDATION_ERROR`
- [ ] 실센서 무이벤트 상태에서 Frontend가 목업 수치를 표시하지 않음
- [ ] CAUTION/DANGER 이벤트가 Agent·WebSocket·SNS 이메일·등록 SMS까지 같은 `lineId`를 유지

Breaking change 절차는 [계약 변경 프로세스](contracts-change-process.md)를 따른다.
