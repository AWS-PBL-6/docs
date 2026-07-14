# 피에조 무입력·지연값·이벤트 표시 정책

> 기준일: 2026-07-14

## 확인된 원인

- Backend는 지금까지 `freqHz > 0`이면 모두 유효 입력으로 받았다. 따라서 장치가 무입력 상태에서
  발행한 `1/4/6 Hz`도 실제 측정 이벤트처럼 저장·표시됐다.
- 온라인 상태(`lastSeenAt`)와 유효 측정 시각이 분리되지 않아 과거 `1 Hz` EventRecord가 현재값처럼
  계속 보일 수 있었다.
- `occurredAt`의 지연/역순 검사가 없어 MQTT 오프라인 큐나 재전송의 늦은 값이 화면을 갱신할 수 있었다.

## 적용 정책

| 항목 | 기본값 | 처리 |
|---|---:|---|
| 활성 주파수 하한 | `8.0 Hz` | 미만은 `IGNORED/BELOW_ACTIVE_FREQUENCY`, 추론·이벤트·WS 측정 생성 안 함 |
| 최대 지연 | `30초` | 초과는 `IGNORED/STALE_READING` |
| 미래 시각 허용 | `5초` | 초과는 `IGNORED/FUTURE_READING` |
| 역순 측정 | 마지막 유효 측정 이하 | `IGNORED/OUT_OF_ORDER_READING` |
| 화면 현재값 TTL | `15초` | 새 유효값이 없으면 “온라인 · 값 대기”로 전환 |

환경변수는 `PIEZO_MIN_ACTIVE_HZ`, `PIEZO_MAX_EVENT_AGE_SECONDS`,
`PIEZO_MAX_FUTURE_SKEW_SECONDS`다. 8 Hz는 현재 PoC slack 보정 기준이므로 현장 기준선 측정 후
센서별 캘리브레이션 값으로 바꿔야 한다.

## 이벤트 탭

정상 측정은 ML 윈도·추세·감사와 현재 센서값 복원을 위해 DynamoDB에 유지한다. 운영자가 조치할
“이벤트” 목록은 `riskLevel=ABNORMAL`로 `CAUTION/DANGER`만 조회한다. 정상 원시값은 계류삭
상세와 로그/분석 API에서 확인한다.

## 장치 측 권장

- FFT peak prominence/진폭 임계값을 함께 만족할 때만 `freqHz` 발행
- 무입력은 `freqHz=1` 같은 대체값 대신 heartbeat만 발행
- 각 측정에 고유 `eventId`와 실제 측정 UTC `occurredAt` 포함
- 오프라인 큐 재전송 시 원 측정시각 유지
