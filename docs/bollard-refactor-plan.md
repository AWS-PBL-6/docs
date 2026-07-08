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

## 남기는 것 (이번 범위 밖)

- 계선 자동 감지(장력 온셋)는 edge-iot 펌웨어/시뮬레이터 몫 — 백엔드는 이벤트만 수신
- AIS/Port-MIS 연동은 후속 (접안 API가 그 자리를 대신하는 수동 트리거)
- 접안 이력(세션 로그) 테이블은 필요해지면 `BerthingRecord`로 확장
- 초음파(AE) 수치는 현재 프론트 결정론적 합성값이며, 실제 센서/ML 파이프라인이 라인별
  최신 특징량을 push하면 그 값으로 대체한다(표시 스키마는 데이터셋과 동일하게 유지).
