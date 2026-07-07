# AE-Sentinel Contracts

이 디렉토리는 `IoT -> Backend -> ML -> Frontend` 경계의 단일 계약 소스다.

canonical source of truth 는 `AWS-PBL-6/docs` 저장소다.

## 원칙

- `schemas/`에는 검증 가능한 JSON Schema를 둔다.
- `examples/`에는 실제 예시 payload를 둔다.
- 스키마 변경 시 `schemaVersion` 또는 `version`을 함께 올린다.
- 팀 간 합의 없는 필드 추가/삭제는 금지한다.

## 변경 절차

- 계약 변경은 먼저 `docs` repo 에서 합의한다.
- schema, example, 관련 문서를 함께 수정한다.
- breaking change 는 기존 `v1` 파일을 덮어쓰지 말고 새 major 파일을 추가한다.
- 상세 절차는 `docs/contracts-change-process.md` 를 따른다.

## 우선 계약

- `schemas/common/event-envelope.v1.json`
- `schemas/iot/ae-feature-event.v1.json`
- `schemas/ml/inference-request.v1.json`
- `schemas/ml/inference-response.v1.json`
- `schemas/websocket/ws-risk-message.v1.json`
