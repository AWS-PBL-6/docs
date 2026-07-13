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
- 상세 절차는 [`../../docs/contracts/contracts-change-process.md`](../../docs/contracts/contracts-change-process.md)를 따른다.

## 우선 계약

- `schemas/common/event-envelope.v1.json`
- `schemas/iot/ae-feature-event.v2.json` ← **현행** (port-domain 접촉식 AE 이벤트)
- `schemas/ml/inference-request.v2.json` ← **현행** (윈도 집계 modelInput)
- `schemas/ml/inference-response.v1.json`
- `schemas/websocket/ws-risk-message.v1.json`

### v1 (보관)

- `schemas/iot/ae-feature-event.v1.json` — Kaggle proxy 시대 per-event 스키마
- `schemas/ml/inference-request.v1.json` — proxy per-event modelInput

> v2는 계류삭 AE 논문 기반 → 항만 도메인 모델로 전환하며 도입됐다. 센서는 각
> 계류삭의 선박 측 연결 고리에 부착되고 `source.deviceId`는 관리자 API에서 해당
> `payload.lineId`에 배정한 AWS IoT Thing ID와 반드시 일치해야 한다. per-event
> 특징(`amplitudeDbAe`·`snrDb`·`hitCount`·`signalType` + 환경 공변량)과, 모델 입력이
> **직전 W개 이벤트 윈도 집계**로 바뀐 점이 v1과의 차이다. 배경은
> [홋줄 AE 데이터셋 평가](../../docs/ml-data/rope-ae-dataset-evaluation.md),
> [계류삭 연결 고리 AE 데이터셋](../../docs/ml-data/port-ae-dataset.md) 참조. wire의
> `schemaVersion`은 PoC 동안 `1.0.0`을 유지한다(파일 major만 v2).

- 피에조 원시 진동수: `schemas/iot/piezo-frequency-event.v1.json`
- 피에조 원시 예제: `examples/iot-piezo-frequency-event.v1.json`
