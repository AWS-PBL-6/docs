# AE-Sentinel 문서 허브

문서는 목적별 폴더로 구분한다. 처음 합류한 팀원은 먼저
[팀 협업 인터페이스 계약](docs/contracts/team-collaboration-interfaces.md)을 읽고,
담당 영역의 운영 문서와 기계 검증 계약(`packages/contracts/`)을 확인한다.

## 문서 구조

| 폴더 | 용도 | 시작 문서 |
|---|---|---|
| [`docs/architecture/`](docs/architecture/README.md) | 현행 AWS·도메인·UI 설계와 의사결정 | [권장 AWS 아키텍처](docs/architecture/ae_sentinel_recommended_architecture.md) |
| [`docs/contracts/`](docs/contracts/README.md) | IoT·Backend·ML·Frontend 팀 경계와 변경 절차 | [팀 협업 인터페이스 계약](docs/contracts/team-collaboration-interfaces.md) |
| [`docs/operations/`](docs/operations/README.md) | AWS 스택 및 센서 온보딩·이벤트 운영 | [센서·이벤트 운영 가이드](docs/operations/sensor-event-piezo-guide.md) |
| [`docs/ml-data/`](docs/ml-data/README.md) | 계류삭 AE 데이터셋·학습 근거·검증 결과 | [연결 고리 AE 데이터셋 카드](docs/ml-data/port-ae-dataset.md) |
| [`docs/reports/`](docs/reports/README.md) | 날짜가 고정된 진행 보고와 원본 기획 기록 | [보고서 색인](docs/reports/README.md) |
| [`docs/archive/`](docs/archive/README.md) | 폐기되거나 대체된 설계·실험 | [보관 문서 색인](docs/archive/README.md) |
| [`packages/contracts/`](packages/contracts/README.md) | OpenAPI·JSON Schema·예제 payload | [계약 패키지 안내](packages/contracts/README.md) |

## 현행 핵심 결정

- 피에조 압전센서는 볼라드가 아니라 **각 계류삭의 선박 측 연결 고리(아이·스플라이스 인근)**에 한 대씩 부착한다.
- `sensorId`는 `lineId`에 1:1 배정하며 `bollardId`는 현재 안벽 연결 위치만 나타낸다.
- 원시 IoT topic은 `aesentinel/{sensorId}/piezo`, 특징 이벤트는 `aesentinel/{sensorId}/features`를 사용한다.
- 프론트 관리자는 AWS IoT Thing 목록에서 센서를 클릭해 계류삭에 적용한다. 센서 ID 수기 입력은 기본 흐름이 아니다.
