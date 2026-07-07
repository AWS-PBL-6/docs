# AE-Sentinel Contracts

이 디렉토리는 `IoT -> Backend -> ML -> Frontend` 경계의 단일 계약 소스다.

## 원칙

- `schemas/`에는 검증 가능한 JSON Schema를 둔다.
- `examples/`에는 실제 예시 payload를 둔다.
- 스키마 변경 시 `schemaVersion` 또는 `version`을 함께 올린다.
- 팀 간 합의 없는 필드 추가/삭제는 금지한다.

## 우선 계약

- `schemas/common/event-envelope.v1.json`
- `schemas/iot/ae-feature-event.v1.json`
- `schemas/ml/inference-request.v1.json`
- `schemas/ml/inference-response.v1.json`
- `schemas/websocket/ws-risk-message.v1.json`
