# 피에조 무입력·지연값·이벤트 표시 정책

> 기준일: 2026-07-14 (텔레메트리/ML 게이팅 분리 개정 반영)

## 확인된 원인

- Backend는 지금까지 `freqHz > 0`이면 모두 유효 입력으로 받았다. 따라서 장치가 무입력 상태에서
  발행한 `1/4/6 Hz`도 실제 측정 이벤트처럼 저장·표시됐다.
- 온라인 상태(`lastSeenAt`)와 유효 측정 시각이 분리되지 않아 과거 `1 Hz` EventRecord가 현재값처럼
  계속 보일 수 있었다.
- `occurredAt`의 지연/역순 검사가 없어 MQTT 오프라인 큐나 재전송의 늦은 값이 화면을 갱신할 수 있었다.

## 적용 정책

| 항목 | 기본값 | 처리 |
|---|---:|---|
| 활성 주파수 하한 | `1.0 Hz` | 미만은 `TELEMETRY_ONLY` — 원시 측정은 `sensor.reading.received`로 실시간 발행하되 ML 추론·이벤트 생성은 건너뜀 |
| 최대 지연 | `30초` | 초과는 `IGNORED/STALE_READING` |
| 미래 시각 허용 | `5초` | 초과는 `IGNORED/FUTURE_READING` |
| 역순 측정 | 마지막 유효 측정 이하 | `IGNORED/OUT_OF_ORDER_READING` |
| 화면 현재값 TTL | `15초` | 새 유효값이 없으면 “온라인 · 값 대기”로 전환 |

시간 유효성(지연·미래·역순) 검사가 먼저 적용되고, 활성 하한은 마지막에 판정한다.
즉 지연·역순인 유휴값은 텔레메트리로도 발행되지 않는다.

### 2026-07-14 개정: 텔레메트리와 ML 게이팅 분리

기존 정책은 8 Hz 미만을 완전히 무시해 센서가 유휴 상태인 동안 대시보드 Hz가
갱신되지 않았다(24시간 546건 전량 무시 관측). 유휴 대역 값도 실제 현재 측정값이므로
원시 텔레메트리는 항상 WebSocket으로 발행하고, ML 이벤트 스팸 방지 목적의
하한은 추론·이벤트 생성에만 적용한다. 프론트엔드 AE 특징량·장력비 추정도
백엔드 이벤트를 기다리지 않고 수신한 `freqHz`에서 직접 계산한다
(`frontend/src/lib/piezo.ts`의 `piezoAeEstimate`, T ∝ f² 동일 보정 상수).

환경변수는 `PIEZO_MIN_ACTIVE_HZ`, `PIEZO_MAX_EVENT_AGE_SECONDS`,
`PIEZO_MAX_FUTURE_SKEW_SECONDS`다.

### 2026-07-15 개정: 시연용 주파수 대역 확정

시연 하드웨어의 정수 Hz 판독에 맞춰 위험도는 측정 주파수를 직접 컷한다
(백엔드 `inference_service` 피에조 정책 v2 = 프론트 `piezoBand` 동일):

| 측정 주파수 | 판정 |
|---|---|
| `< 1 Hz` | 유휴 대역 — `TELEMETRY_ONLY`, 추론 없음 |
| `1 ~ 3 Hz 미만` | `NORMAL` |
| `3 ~ 6 Hz 미만` | `CAUTION` |
| `6 Hz 이상` | `DANGER` (SNS 이메일 + SMS 발송) |

보정 상수도 같은 범위로 재설정: `PIEZO_FREQ_SLACK_HZ=1.0`,
`PIEZO_FREQ_CRIT_HZ=6.0` (loadRatio는 1 Hz→0, 6 Hz→1). 대역 컷은
`PIEZO_NORMAL_MAX_HZ`(3.0)·`PIEZO_CAUTION_MAX_HZ`(6.0)로 배포별 튜닝 가능하며,
현장 기준선 측정 후 센서별 캘리브레이션 값으로 바꿔야 한다.

## 이벤트 탭

정상 측정은 ML 윈도·추세·감사와 현재 센서값 복원을 위해 DynamoDB에 유지한다. 운영자가 조치할
“이벤트” 목록은 `riskLevel=ABNORMAL`로 `CAUTION/DANGER`만 조회한다. 정상 원시값은 계류삭
상세와 로그/분석 API에서 확인한다.

## 장치 측 권장

- FFT peak prominence/진폭 임계값을 함께 만족할 때만 `freqHz` 발행
- 무입력은 `freqHz=1` 같은 대체값 대신 heartbeat만 발행
- 각 측정에 고유 `eventId`와 실제 측정 UTC `occurredAt` 포함
- 오프라인 큐 재전송 시 원 측정시각 유지
