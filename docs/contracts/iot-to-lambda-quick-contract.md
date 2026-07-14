# IoT → Lambda 전달 규격

> 피에조 압전센서 기준 · AWS 리전 `ap-northeast-2`

## 한 줄 흐름

`센서 MQTT publish` → `AWS IoT Rule` → `ae-sentinel-app-ingest-piezo Lambda`

센서가 Lambda를 직접 호출하지 않는다. 아래 topic과 JSON으로 발행하면 IoT Rule이 Lambda를 호출한다.

접속 전제: IoT Thing에 **활성 인증서**와 `iot:Connect`·해당 topic의 `iot:Publish` 정책을 연결한다. 장치에 `lambda:InvokeFunction` 권한은 필요 없다.

## 1. MQTT topic

```text
aesentinel/{sensorId}/piezo
```

예: `aesentinel/factorypulse_gateway_01/piezo`

## 2. MQTT 메시지(JSON)

```json
{
  "schemaVersion": "1.0.0",
  "eventType": "piezo.frequency.detected",
  "eventId": "evt-20260713-0001",
  "traceId": "trc-20260713-0001",
  "occurredAt": "2026-07-13T09:00:00.000Z",
  "sentAt": "2026-07-13T09:00:00.120Z",
  "source": {
    "account": "iot",
    "service": "esp32-piezo",
    "deviceId": "factorypulse_gateway_01"
  },
  "payload": {
    "freqHz": 28.5
  }
}
```

## 3. 반드시 맞아야 하는 값

```text
topic의 {sensorId}
       = source.deviceId
       → Backend가 LineStatus.sensorId로 현재 계류삭을 조회
```

`eventId`는 메시지마다 새 값, 시간은 UTC ISO 8601, `freqHz`는 `0`보다 큰 Hz 값으로 보낸다.
`topicSensorId`는 보내지 않는다. IoT Rule이 topic의 `{sensorId}`를 읽어 자동으로 추가한다.
`siteId`, `berthId`, `lineId`도 보내지 않는다. Lambda가 관리 화면의 최신 센서 배정에서 결정한다.

| 잘못된 경우 | Lambda 처리 결과 |
|---|---|
| 계류삭에 배정되지 않은 센서 | `SENSOR_NOT_BOUND` |
| topic과 `source.deviceId` 불일치 | `VALIDATION_ERROR` |
| 같은 `eventId` 재전송 | `DUPLICATE_EVENT` |
| 필수 필드·단위 오류 | `VALIDATION_ERROR` |

전체 필드 규칙은 [IoT JSON Schema v2](../../packages/contracts/schemas/iot/piezo-frequency-event.v2.json), 등록·검증 절차는 [센서 운영 가이드](../operations/sensor-event-piezo-guide.md)를 따른다.

## 4. 온라인 상태 heartbeat

```text
MQTT clientId = AWS IoT Thing 이름 = sensorId
topic: aesentinel/{sensorId}/heartbeat
주기: 30초 · 120초 동안 lifecycle/heartbeat/측정값이 없으면 오프라인
```

heartbeat JSON은 [센서 heartbeat 예제](../../packages/contracts/examples/iot-sensor-heartbeat.v1.json)를 따른다.
AWS IoT의 연결/해제 lifecycle 이벤트는 장치가 별도로 만들지 않으며 Backend가 자동 수신한다.
