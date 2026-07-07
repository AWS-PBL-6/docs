# Contracts Change Process

## 1. 목적

이 문서는 `AWS-PBL-6/docs` 저장소를 기준으로 `packages/contracts` 변경을 관리하는 절차를 정의한다.

적용 대상:

- `packages/contracts/schemas/**`
- `packages/contracts/examples/**`
- 계약 변경을 설명하는 아키텍처 문서

## 2. 기본 원칙

- contracts 의 canonical source of truth 는 `AWS-PBL-6/docs` 이다.
- 구현 저장소는 contracts 를 직접 정의하지 않고 참조하거나 동기화한다.
- breaking change 는 구현 코드보다 먼저 docs repo 에서 합의한다.
- schema 와 example 은 항상 함께 변경한다.

## 3. 변경 유형

### 3.1. 문서 보강

영향:

- 설명 문구 수정
- 예제 설명 추가
- 오타 수정

처리:

- docs repo PR 만으로 처리 가능

### 3.2. 비파괴 변경

예시:

- optional 필드 추가
- 설명 추가
- enum 확장 전 사전 합의
- 예제 payload 보강

처리:

- docs repo 에서 schema + example + 설명 문서를 함께 수정
- 관련 구현 저장소 PR 을 후속으로 생성

### 3.3. 파괴적 변경

예시:

- required 필드 추가
- 필드명 변경
- payload 구조 이동
- enum 축소/이름 변경
- 응답 구조 변경

처리:

- 기존 `v1` 파일을 덮어쓰지 않는다
- 새 major version 파일을 추가한다. 예: `ae-feature-event.v2.json`
- `schemaVersion` 도 새 major 기준으로 올린다
- migration note 를 문서화한다
- 구현 저장소는 docs PR 승인 후 순차 반영한다

## 4. 표준 변경 순서

1. 변경 필요성을 이슈나 PR 설명에서 명시한다.
2. `AWS-PBL-6/docs` 에서 schema, example, 문서를 먼저 수정한다.
3. PR 본문에 영향 받는 저장소를 명시한다.
4. docs PR 승인 후 각 구현 저장소에서 후속 PR 을 만든다.
5. 구현 반영이 끝나면 release note 또는 changelog 성격의 짧은 기록을 남긴다.

## 5. PR 최소 체크리스트

- schema 파일이 변경되었는가
- example payload 가 함께 갱신되었는가
- 관련 아키텍처 문서가 갱신되었는가
- breaking/non-breaking 여부가 명시되었는가
- 영향 받는 저장소가 적혔는가

## 6. 영향 저장소 매핑

| 계약 영역 | 우선 영향 저장소 |
| --- | --- |
| `schemas/iot/**` | `Edge-IoT`, `Backend`, `docs` |
| `schemas/ml/**` | `ML`, `Backend`, `docs` |
| `schemas/websocket/**` | `Backend`, `Frontend`, `docs` |
| `schemas/app/**` | `Backend`, `Frontend`, `docs` |
| `schemas/common/**` | 전 저장소 |

## 7. 버전 규칙

- 파일명 suffix 의 `v1`, `v2` 는 **major compatibility** 를 뜻한다.
- JSON 내부 `schemaVersion` 은 `1.0.0`, `1.0.1` 같은 세부 버전을 뜻한다.
- breaking change 는 새 파일로 올린다.
- non-breaking change 는 같은 major 파일 안에서 관리할 수 있다.

## 8. 권장 병합 순서

1. `docs`
2. `Backend`
3. `Edge-IoT` 또는 `ML`
4. `Frontend`

이 순서는 시스템 중심 오케스트레이터가 Backend 이기 때문에 권장된다.

## 9. 금지 사항

- 구현 저장소에서 docs 합의 없이 schema 를 독자 수정하는 행위
- example 없이 schema 만 바꾸는 행위
- 기존 major schema 를 조용히 깨는 행위
- `DANGER`, `CAUTION`, `NORMAL` 같은 공통 enum 을 단일 저장소에서 임의 변경하는 행위

## 10. 운영 팁

- 각 구현 저장소 README 에 docs repo 와 contracts 기준 경로를 명시한다.
- contract 관련 PR 제목은 `contract:` prefix 를 권장한다.
- merge 전에 관련 팀 최소 1명 이상 확인을 받는 편이 좋다.
