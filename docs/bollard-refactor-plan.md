# 볼라드 중심 모델 리팩토링 — 실행 계획

> 작성일: 2026-07-08 · 근거: [센서 부착 위치 결정](sensor-placement-decision.md)
> **상태: 완료 (2026-07-08)** — 아래 전 항목이 백엔드/프론트/인프라에 반영·배포되었다.

센서를 안벽(지면) 볼라드 고정형으로 전환하는 데 따른 백엔드/프론트 작업 목록.
`Bollard = 영구 자산`, `Line.bollardId = 불변 귀속`, `Line.vesselId = 접안 세션 동안만
붙는 시간제 연결`이 핵심 원칙이다.

## 1. Backend — 모델 (application-api)

- [x] `models/records.py`
  - `BollardRecord` 신설: `bollardId`, `siteId`, `berthId`, `positionM`(안벽 서쪽 끝
    기준 미터), `label`, `twinSlot`, `createdAt/updatedAt`
  - `LineStatus.bollardId` 추가 (센서의 영구 귀속처)
  - `LineStatus.vesselId` 의미 재정의: 접안 세션 동안만 유효한 시간제 연결
  - `VesselRecord` docstring에서 "선박 볼라드에 센서" 전제 제거
- [x] `common/errors.py`: `BOLLARD_NOT_FOUND`, `BOLLARD_FULL`

## 2. Backend — 저장소

- [x] `repositories/app_repository.py`
  - InMemory/Dynamo 양쪽에 bollard CRUD (`put/get/list/delete_bollard`)
  - 시드 데이터: 볼라드 시드 + 라인에 `bollardId` 부여
  - `_bollard_from_dict` 재수화, `_line_from_dict`에 `bollardId`
  - `summary()`에 `totalBollards`
- [x] `common/config.py`: `BOLLARDS_TABLE` env

## 3. Backend — 서비스 / REST

- [x] `services/command_service.py`
  - bollard create/update/delete
  - line create/update가 `bollardId` 수용
  - **볼라드당 최대 2개 라인** 제한(`MAX_LINES_PER_BOLLARD=2`, 초과 시 `BOLLARD_FULL` 409)
  - **미계선(vesselId 없음) 라인은 항상 `NORMAL`**(create/update에서 강제, 출항 시 리셋)
  - **`lineId` 생략 시 자동 생성**(`new_id("mooring")` → `mooring-<날짜>-<hex>`)
  - `berth_vessel(vesselId, body)`: 접안 처리 — 지정 `bollardIds` 또는 해당 선석의
    미배정 라인 자동 귀속 + 선박 상태 `BERTHED`
  - `depart_vessel(vesselId)`: 출항 처리 — 라인 `vesselId` 해제 + 상태 `DEPARTED`
- [x] `services/query_service.py`: bollard 목록/조회
- [x] `handlers/rest_api.py` 라우트
  - `GET/POST /v1/bollards`, `GET/PUT/DELETE /v1/bollards/{bollardId}`
  - `POST /v1/vessels/{vesselId}/berthing` (접안 = 1클릭 매핑)
  - `POST /v1/vessels/{vesselId}/departure` (출항 = 매핑 해제)

## 4. Backend — 문서 / 인프라

- [x] `apidocs/openapi.yaml`: bollards·berthing·departure·`BOLLARD_FULL`·`lineId` 자동 생성 반영
  (canonical `docs/packages/contracts/openapi/ae-sentinel-rest.v1.yaml`와 동기화)
- [x] `backend/infra/application-account/template.yaml`: `BollardsTable` +
  Lambda env `BOLLARDS_TABLE` (배포 완료)

## 5. Frontend 연동

- [x] `lib/types.ts`: `Bollard` 타입(+`twinSlot`), `LineStatus.bollardId`
- [x] `lib/api.ts`: `listBollards`/`createBollard`/`updateBollard`/`deleteBollard`,
  `berthVessel`, `departVessel`, `LineInput.bollardId`
- [x] `twin/layout.ts`: **실제 볼라드가 안벽 계선주(quay post)**가 되어 라인이 자신의
  `bollardId` 포스트로 수렴한다. 선박은 자신이 매인 볼라드 평균 X 앞에 배치되고,
  선체 겹침 방지 분리 패스로 서로 겹치지 않게 밀어낸다.
- [x] `components/LineForm.tsx`: 볼라드 선택(용량 n/2 표시·만석 비활성), Site/Berth는
  볼라드에서 자동 파생, `lineId`는 선택(미입력 시 자동 생성)
- [x] `features/VesselsManage.tsx`: 선박 펼침 안에서 계류삭 추가/수정/삭제 + 접안/출항 액션
- [x] `twin/demo.ts`: 데모 라인에 `bollardId` 반영(볼라드 공유 예시 포함)

## 6. 트윈 상호작용 · UI (계획 이후 확정)

- [x] 대시보드 디지털 트윈에서 **볼라드를 클릭하면 볼라드 상세**(사이드 패널: 정보 +
  수정/삭제(우측 상단 아이콘 버튼) + 빈 볼라드는 "볼라드 센서 매핑"), **계류삭을 클릭하면
  계류삭 상세**(실시간 상태 + 3D 홋줄 + 매핑된 라인은 초음파(AE) 상세 수치)를 표시
- [x] 볼라드 위 마우스오버 시 `라벨 · 연결수/최대(2)` 툴팁, 연결 라인 중 최고 위험도 링 표시
- [x] 볼라드 관리 탭 제거 → 대시보드 클릭 기반으로 통합, 대시보드에서 볼라드 추가 가능
- [x] 아이콘 라이브러리(lucide-react)로 토글/액션 표기 정리
- [x] **매핑된 계류삭 초음파(AE) 수치** 표시 — `lib/acoustic.ts`가 AE-Damage-Detection
  데이터셋 스키마(진폭 dB·에너지 J·지속/상승 ms·주파수 Hz·충격 에너지 J·온도/습도·섬유/수지/
  두께·손상유형 0~3)를 라인별 결정론적으로 생성해 `LineDetail`에 표시

## 7. 검증

- [x] 백엔드: 로컬 리포지토리 시나리오 — 접안 → 라인 자동 귀속 → 출항 → 해제,
  볼라드 최대 2개 초과 거부, 미계선 NORMAL, `lineId` 자동 생성
- [x] 프론트: `tsc -b && vite build` 통과, dev 서버 `/demo` 확인

## 8. 감사(리뷰) 후 수정 — 2026-07-09

구현 전체를 재검토(시나리오 테스트 + 화면 검증)하여 아래 결함을 수정했다.

- [x] **볼라드 삭제 가드**: 라인이 귀속된 볼라드 삭제 시 dangling `bollardId` 참조가
  남던 문제 → `BOLLARD_IN_USE`(409)로 거부 (errors.py, command_service, OpenAPI 양 사본,
  BollardDetail 삭제 버튼 비활성+사유 툴팁)
- [x] **`berth_vessel` 원자성**: 라인을 순차 커밋하다 충돌(409) 시 일부만 귀속된 상태로
  중단되던 문제 → 검증 패스 후 일괄 커밋(2-pass)
- [x] **`twinSlot` 미구현(죽은 필드)**: 모델·API에만 있고 읽는 곳이 없었음 →
  `twin/layout.ts`가 twinSlot(1-based)을 트윈 포스트 슬롯에 고정 배치, 나머지는
  (선석, 위치) 순으로 빈 슬롯 채움
- [x] **볼라드 라벨 불일치**: HUD/씬 태그가 raw id(`bollard-01`)와 라벨(`B-01`)을
  혼용 → `bollardByLine`·계류삭 태그가 포스트 라벨로 통일
- [x] **홋줄 교차**: 페어리드 순서가 lineId 정렬이라 볼라드 위치와 어긋나면 로프가
  교차 → 볼라드 X 좌표 순으로 페어리드 배정
- [x] `MAX_LINES_PER_BOLLARD` 상수를 import 블록 밖으로 이동 (style)

검증: 백엔드 py_compile + 시나리오(접안 원자성/삭제 가드/정원 초과/출항 리셋) 통과,
`tsc -b && vite build` 통과, `/demo` 평시·위험 페이즈 스크린샷 확인.

## 9. 센서 연동 vs 계류삭 연결 분리 — 2026-07-09

볼라드 상세에서 "센서 매핑"과 "계류삭 매핑"의 경계가 모호하다는 피드백에 따라
두 개념을 명확히 분리했다.

**개념 모델**

| | 센서 연동 (온보딩) | 계류삭 연결 (운영) |
|---|---|---|
| 시점 | 볼라드 추가 시 **첫 1회** | 수시 |
| 선택지 | 실센서(DEVICE, 장비 ID 필수) 또는 가상 모드(VIRTUAL) | 미연결 계류삭 연결 / 연결 해제 |
| 변경 | 불가 — 유일한 예외는 VIRTUAL→DEVICE(실물 설치 시) | 자유 — 단 계선 중(vesselId 有) 라인은 잠금 |

**백엔드**

- [x] `BollardRecord.sensorMode`(PENDING→DEVICE\|VIRTUAL) + `sensorId` + `sensorBoundAt`
- [x] `POST /v1/bollards/{id}/sensor` — 1회 연동(재연동 `SENSOR_ALREADY_BOUND` 409,
  VIRTUAL→DEVICE 만 허용); 생성 시 `sensorMode` 동시 지정도 가능
- [x] 미연동(PENDING) 볼라드에 계류삭 연결 거부 — `BOLLARD_NOT_PROVISIONED` 409
- [x] 계선 중 라인의 볼라드 변경/해제 거부 — `LINE_MOORED` 409 (출항 처리 먼저)
- [x] 시드: berth-01 DEVICE(ae-000x) · 07-09 VIRTUAL · 10-12 PENDING, OpenAPI 양 사본 반영

**프론트**

- [x] `BollardForm`: 생성 시 연동 라디오(가상 모드 기본 / 실센서 ID / 나중에)
- [x] `BollardDetail`: 센서 상태 카드(미연동→[실센서 연동]/[가상 모드로 시작],
  가상→[실센서로 전환]) + "계류삭 연결" 섹션(미연결 라인만 목록, 행별 연결 해제
  버튼 — 계선 중이면 비활성+사유)
- [x] `LineForm`: 미연동 볼라드 선택 불가("센서 미연동"), 계선 중 볼라드 변경 잠금
- [x] 트윈: 미연동 포스트는 어두운 링, 툴팁에 `연결 n/2 · 가상|센서 미연동` 표시;
  볼라드 클릭 히트 타깃 확대(r2.4→4.5m); `BOLLARD_SPACING` 34→60m(선박이 자기
  볼라드 군집 위에 앉도록 — 겹침 방지 패스의 드리프트 제거)

검증: 백엔드 시나리오(PENDING 연결 거부 → 연동 → 연결, DEVICE 재연동 409,
VIRTUAL→DEVICE 전환, 계선 중 재매핑 409) 통과, 빌드 통과, `/demo` 볼라드 클릭 →
상세 패널(가상 모드 카드 + 연결 계류삭) 스크린샷 확인.

## 10. 디지털 트윈 낮/밤 자동 전환 — 2026-07-09

현재 시각에 따라 씬 환경이 자동으로 낮/밤 상태로 전환된다.

- `twin/daylight.ts`: 시각 → **일광 계수**(0=밤, 1=낮) — 새벽 05:30~07:30,
  해질녘 18:30~20:30 사이 smoothstep 전환, 30초마다 갱신(스텝 ~1% 미만)
- 일광 계수 하나로 전체 환경을 보간: 하늘/바다 셰이더(uniform), 안개 색·거리,
  ambient/hemisphere/directional 조명(달빛 저각 → 햇빛 고각), 별·달(밤) ↔ 해·글리터(낮),
  도시 실루엣·산·안벽·크레인 색, 야간 조명(마스트·크레인 캡·선박 창문·항해등·
  부산항대교 케이블·도시 비콘)은 밤에만 점등
- 위험/주의/정상 상태색, 위험반경, HUD는 낮/밤 무관하게 동일 (관제 정보 우선)
- 시연·테스트: `?tod=13`(낮) `?tod=19.5`(황혼) `?tod=23`(밤) URL 파라미터로 시각 고정
- 검증: 세 시각 스크린샷 + 빌드 통과

## 남기는 것 (이번 범위 밖)

- 계선 자동 감지(장력 온셋)는 edge-iot 펌웨어/시뮬레이터 몫 — 백엔드는 이벤트만 수신
- AIS/Port-MIS 연동은 후속 (접안 API가 그 자리를 대신하는 수동 트리거)
- 접안 이력(세션 로그) 테이블은 필요해지면 `BerthingRecord`로 확장
- 초음파(AE) 수치는 현재 프론트 결정론적 합성값이며, 실제 센서/ML 파이프라인이 라인별
  최신 특징량을 push하면 그 값으로 대체한다(표시 스키마는 데이터셋과 동일하게 유지).
