# Backend ↔ Response Guidance Agent 계약

> 상태: 현행(Canonical) · 버전 `1.0.0` · 기준일 2026-07-14  
> 기계 검증: `ml/agent-response-layer/contracts/schemas/`

## 한 줄 원칙

Backend가 `CAUTION/DANGER`를 확정한 뒤 Agent를 동기 호출한다. Agent는 승인된 PDF/문서만
RAG 검색해 안내 JSON을 반환하며, 위험도 재판정·물리 계산·저장·발송을 하지 않는다.

```text
IoT → Backend → SageMaker/정책 → Backend ──Invoke──▶ Agent/Bedrock KB
                                Backend ◀──guidance──┘
                                  ├─ EventRecord.responseGuidance 저장
                                  ├─ WebSocket 경고창
                                  ├─ SNS Topic 이메일
                                  └─ DynamoDB opt-in 수신자 SMS
```

## 기존 제안 JSON과 달랐던 점

| 제안 필드/문장 | 현재 프로젝트 사실 | 교정 |
|---|---|---|
| `rms`, `peakAmplitude`, `energy`, `dominantFrequencyHz`, `riseTimeMs` | 현행 port-domain/piezo 경로의 EventRecord에 없음 | 현행 AE proxy 필드로 변경 |
| `materialType`, `tensionEstimateKn`, `temperatureC`, `humidityPct` | 현재 센서가 측정·전송하지 않음 | 보내지 않고 Agent도 추정 금지 |
| “합성섬유”, “MBL 70% 초과” | 입력 근거가 없으면 단정 불가 | 실측 `structuralFreqHz`와 파생 `loadRatio`만 설명 |
| `alertDispatched` | 발송은 Backend 책임 | Agent 응답은 항상 `false` |
| 가상 터미널 규정 인용 | 운영 근거가 아님 | 승인 메타데이터가 붙은 실제 PDF만 검색 |

피에조 원본 사실값은 `context.structuralFreqHz` 하나다. `loadRatio`와 `features`의 AE 값은
Backend가 보정식으로 만든 파생값이므로 안내문에서도 실측과 파생을 구분한다.

## 교정된 입력 예시

```json
{
  "schemaVersion": "1.0.0",
  "requestId": "req-20260714-0005",
  "traceId": "trc-20260714-0005",
  "eventId": "evt-20260714-0005",
  "lineId": "mooring-05",
  "siteId": "yard-a",
  "berthId": "berth-01",
  "bollardId": "bollard-05",
  "vesselId": "vessel-02",
  "riskLevel": "DANGER",
  "riskScore": 0.9215,
  "classProbabilities": { "NORMAL": 0.0, "CAUTION": 0.0, "DANGER": 1.0 },
  "modelVersion": "ae-port-window-xgb-v1-20260713+piezo-frequency-policy-v1",
  "features": {
    "amplitudeDbAe": 91.99,
    "freqLowKhz": 25.37,
    "freqHighKhz": 198.33,
    "durationMs": 3.55,
    "hitCount": 26,
    "snrDb": 57.6,
    "signalType": "High amplitude"
  },
  "context": {
    "ambientNoiseDb": 40.0,
    "rainMmh": 0.0,
    "windMps": 0.0,
    "craneActive": 0,
    "sensorKind": "piezo",
    "structuralFreqHz": 30.8,
    "loadRatio": 0.9215
  }
}
```

## 교정된 출력 예시

```json
{
  "schemaVersion": "1.0.0",
  "type": "response.guidance.generated",
  "traceId": "trc-20260714-0005",
  "eventId": "evt-20260714-0005",
  "generatedAt": "2026-07-14T09:00:01.480Z",
  "lineId": "mooring-05",
  "vesselId": "vessel-02",
  "siteId": "yard-a",
  "riskLevel": "DANGER",
  "riskScore": 0.9215,
  "source": "KB",
  "alertDispatched": false,
  "guidance": {
    "riskLevel": "DANGER",
    "headline": "5번 계류삭 위험. 데드존 통제·작업 중단·인원 이탈 확인",
    "situation": "피에조 센서에서 구조 진동수 30.8 Hz가 수신되었고, 현재 보정식의 파생 loadRatio는 0.9215입니다. Backend가 확정한 DANGER 판정에 따라 즉시 안전조치가 필요합니다.",
    "immediateActions": [
      "스냅백 위험구역 내 인원 이탈 완료 확인",
      "해당 선석 작업 즉시 중단",
      "위험구역 접근 통제"
    ],
    "followupActions": [
      "선박 측에 계류 재조정 요청",
      "인접 라인 연쇄 파단 가능성 점검",
      "비상 보고 체계 가동"
    ],
    "citations": [
      "s3://<approved-kb>/general/<approved-source>.pdf"
    ],
    "confidence": "HIGH"
  }
}
```

## 오류·발송 규칙

- Agent/RAG 오류: Backend가 `confidence=LOW`, `fallback-template` 안내를 생성해 무응답을 막는다.
- Agent가 입력 위험도를 바꾸면 응답을 폐기하고 fallback한다.
- WebSocket 팝업은 `risk.caution.detected`와 `risk.danger.detected`만 연다.
- SNS Topic은 구독 이메일로, 직접 SMS는 `enabled=true && smsOptIn=true`이며 사이트/선석이
  일치하는 DynamoDB 수신자에게만 보낸다.
- 같은 `eventId`는 한 번만 발송한다.
