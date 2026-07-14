# 프로덕션 배포 · 센서 마이그레이션 보고서 (2026-07-13)

> 상태: 배포 완료 · 계정 `727847798739` · 리전 `ap-northeast-2`  
> 현장 잔여 작업: 미등록 물리 센서 3대 등록 및 실제 연결 고리 재부착 확인

## 배포 결과

| 영역 | 결과 | 실제 리소스 |
|---|---|---|
| Lambda/SAM | `UPDATE_COMPLETE` | CloudFormation `ae-sentinel-app` |
| REST | 실제 Lambda invoke 200 | `ae-sentinel-app-rest` |
| 피에조 인입 | 3개 이벤트 종단 검증 성공 | `ae-sentinel-app-ingest-piezo` |
| SageMaker Processing | `Completed` | `ae-sentinel-line-eye-processing-20260713-01` |
| SageMaker Training | `Completed`, billable 150초 | `ae-sentinel-line-eye-training-20260713-02` |
| Model Registry | Version 1 `Approved` | `ae-sentinel-line-eye-models/1` |
| 실시간 추론 | `InService` | endpoint `ae-sentinel-port-window` |
| Amplify | Job 14 `SUCCEED` | app `ddcrzjuu5yfnn`, branch `main` |
| Cognito | 프로덕션 callback/logout 추가 | `https://main.ddcrzjuu5yfnn.amplifyapp.com/` |

Amplify에서 내려받은 프로덕션 `index.html`은 Job 14의 새 번들을 참조했고,
JS 번들은 `index-CGOCKxRS.js`, CSS는 `index-xX9b3H3O.css`다.

### 프론트 센서값 누락·역행 원인 및 정합성 재정립

2026-07-14 운영 로그에서 고위험 측정은 RAG 안내 생성으로 완료가 늦어지는 동안 뒤의
정상 측정이 먼저 완료되는 병렬 Lambda 순서 역전이 확인됐다. 기존 Frontend는 WebSocket
도착 순서대로 현재값을 덮어써 최신 13 Hz 뒤에 과거 37 Hz가 다시 표시될 수 있었고,
수신값을 15초 뒤 삭제해 장치가 온라인이어도 `값 대기`로 되돌아갔다.

- Backend는 `lastMeasurementAt`을 DynamoDB 조건부 갱신해 더 최신 장치 시각만 수락한다.
- 추론 결과는 자신이 여전히 최신 측정일 때만 LineStatus에 반영한다. 과거 결과는
  EventRecord·경보 이력에는 보존하되 WebSocket의 `statusCurrent=false`로 표시한다.
- Frontend는 `occurredAt` 기준으로 최신 측정만 현재값에 반영하고 15초 강제 삭제를 없앴다.
  오래된 값은 `마지막 측정`과 경과시간으로 구분하며, 실제 disconnect일 때만 제거한다.
- `alert.sent`는 SNS/SMS 발송 감사 메시지일 뿐 센서 상태가 아니므로 LineStatus 갱신에서
  제외한다. 늦게 발송된 과거 DANGER 알림이 최신 NORMAL 상태를 덮지 않는다.
- heartbeat의 `updatedAt`은 측정 시각이 아니다. 상세창은 `occurredAt`과
  `lastMeasurementAt`만 비교해 추론 대기 중 heartbeat가 실측값을 숨기지 않게 한다.
- 이벤트 탭은 운영 요구대로 `CAUTION`·`DANGER`만 표시한다. `NORMAL` 측정은 모델 추세와
  최신값 복구를 위해 저장되지만 경고 이벤트 목록에는 노출하지 않는다.

운영 재현에서는 40 Hz(`07:44:06.119Z`) 뒤에 18 Hz(`07:44:07.110Z`)가 수신됐고,
18 Hz/NORMAL 판정이 먼저 완료됐다. 늦게 완료된 40 Hz/DANGER 프레임은
`statusCurrent=false`였으며 DynamoDB LineStatus를 되돌리지 않았다. 이후 최신 10 Hz 이벤트도
LineStatus와 정확히 일치했다. CloudFormation은 `UPDATE_COMPLETE`, Piezo Lambda는
`Active/Successful`, Amplify Job 14는 2026-07-14 16:59:37 KST에 `SUCCEED`했다.

### 상세 3D·0 Hz 기본값·실시간 동기화 보완

2026-07-14 13:28 KST에 Frontend를 Amplify main Job 11로 배포했다. 컨테이너선은
렌더링 때 적재 배열이 바뀌지 않는 고정 다단 적재로 변경했고, 계류삭·이벤트 상세의
Three.js 홋줄은 배포 후 지연 청크가 사라지는 문제를 막기 위해 기본 번들에 포함했다.
오프라인 실센서는 과거 EventRecord의 1 Hz를 현재값으로 재사용하지 않고 0.0 Hz를
표시한다. WebSocket 측정 직후 늦게 끝난 REST 응답이 최신 상태를 덮지 않도록 타임스탬프
병합을 적용했으며, 연결 해제 시 화면의 현재 리딩을 제거하고 30초 주기로 상태를 보정한다.

### IoT 피에조 v2 Lambda 재배포

2026-07-13 16:33 KST에 `ae-sentinel-app`을 다시 배포해 CloudFormation
`UPDATE_COMPLETE`를 확인했다. `ae-sentinel-app-ingest-piezo`에 `lineId`·`siteId`·`berthId`
없이 `factorypulse_gateway_01`, `freqHz=8.1`만 전달한 실제 invoke가 200/`status=OK`를 반환했고,
Backend가 최신 배정인 `mooring-20260713-e7ab9659`, `yard-a`, `berth-02`를 자동으로 채웠다.
결과는 `NORMAL`, `riskScore=0`, `alertSent=false`였다. 검증 이벤트는 삭제했으며
`lastEventId`와 `deviceOnline`도 조건부 원복해 운영 데이터에는 남기지 않았다.

### 센서 실시간 연결 상태 배포

2026-07-13 17:01 KST에 `ae-sentinel-app-sensor-presence` Lambda와 아래 트리거를 배포했다.

- `$aws/events/presence/connected/+`, `$aws/events/presence/disconnected/+` IoT Rule
- `aesentinel/+/heartbeat` IoT Rule(장치 권장 주기 30초)
- EventBridge 1분 Schedule, heartbeat timeout 120초

실제 `factorypulse_gateway_01`에서 lifecycle connect/disconnect가 Lambda로 전달됐고,
피에조 측정 `1.0 Hz` 3건도 수신되어 `lastSeenAt`이 갱신됐다. 마지막 실제 disconnect
`2026-07-13T08:04:17.699Z`에서 `deviceOnline=false` 전환을 확인했다. 테스트 heartbeat는
Rule→Lambda→DynamoDB→REST를 통과해 `CONNECTED`와 `lastSeenAt`을 반환했으며 별도
EventRecord를 만들지 않는다. Frontend는 Amplify main Job 6으로 `lastSeenAt` 표시와
WebSocket 온라인/오프라인 갱신을 배포했다.

### 비동기 실시간 센서값 갱신

2026-07-13 17:29 KST에 원시 피에조 값이 SageMaker 추론을 기다리던 병목을 분리했다.
Backend는 유효한 값을 받으면 먼저 `sensor.reading.received`로 `freqHz`, `loadRatio`,
`deviceOnline`, `lastSeenAt`을 push하고, 추론 완료 후 `risk.*` 메시지로 위험도를 다시
push한다. Frontend는 원시값을 즉시 표시하며 WebSocket 재연결·브라우저 온라인 복귀 시
REST 최신 상태를 재동기화한다.

실제 프로덕션 WebSocket을 연결한 채 IoT Core에 `10.5 Hz`를 발행해 같은 연결에서
`sensor.reading.received`와 `risk.caution.detected`를 순서대로 수신했다. Lambda 로그 기준
warm 실행 전체는 328 ms였고 두 WebSocket 메시지 간격은 약 0.22초였다. 검증 이벤트 3건은
다른 신규 이벤트가 없다는 조건부 트랜잭션으로 삭제했으며, 최신 운영 이벤트
`evt-efe2f3c2-bacd-4caf-8414-3b4b2d867ffa`와 `deviceOnline=false` 상태로 복원했다.
Frontend는 Amplify main Job 8(`SUCCEED`, 17:30:51 KST 완료)로 배포했다.
이후 관제 현황 행에서도 최신 `freqHz`, 수신 경과시간, 온라인 상태가 보이도록 보완해
Job 9(`SUCCEED`, 17:46:17 KST 완료)로 배포했다. 페이지 최초 진입 시 최근 EventRecord로
값을 복원하고, 이후 `sensor.reading.received`로 같은 표시를 계속 갱신한다.

### 오프라인 위험색·저주파 오판정 교정

2026-07-13 18:00 KST에 오프라인 계류삭의 과거 위험도를 활성 관제 상태로 표시하던
문제를 수정했다. 오프라인/PENDING 계류삭은 회색 `관제 대기`로 표시되고 위험 KPI,
선박 위험색, 경보 배너에서 제외된다. 과거 판정은 상세 화면에서만 `마지막 판정`으로
구분해 보여준다.

Backend에는 `piezo-frequency-policy-v1`을 적용해 피에조 최종 판정과 `riskScore`가
문서화된 부하비 구간을 따르도록 했다. 잘못 저장돼 있던 `1.0 Hz / CAUTION 0.64` 이벤트와
라인 최신 상태는 조건부 DynamoDB 트랜잭션으로 `NORMAL 0.00`으로 교정했다. Lambda/SAM은
CloudFormation `UPDATE_COMPLETE`, Frontend는 Amplify Job 10(`SUCCEED`, 18:00:33 KST)이다.

## SageMaker 모델 증적

- 데이터: `port_mooring_eye_ae_dataset.csv`, 55개 세션
- 모델: `ae-port-window-xgb-v1-20260713`
- OOF 정확도: `0.8695`
- OOF macro-F1: `0.8707`
- 위험 재현율: `0.9699`
- 위험 정밀도: `0.9993`
- S3 모델:
  `s3://ae-sentinel-sagemaker-727847798739/line-eye/training/ae-sentinel-line-eye-training-20260713-02/output/model.tar.gz`

검증용 endpoint에서 JSON 계약 invoke 후 Registry 버전 1을 승인했다. 프로덕션
endpoint에서도 같은 smoke request가 `DANGER`, `riskScore=0.997`,
`modelVersion=ae-port-window-xgb-v1-20260713`을 반환했다. 이후 Piezo Lambda에
18 Hz대 이벤트 3개를 순차 입력해 warming-up 2회 후 실제 SageMaker 모델 응답까지
확인했다. 생성된 테스트 이벤트 3개는 삭제하고 라인 위험 상태를 `NORMAL`로 복원했다.

첫 Training 시도 `ae-sentinel-line-eye-training-20260713-01`은 Script Mode의
`sagemaker_submit_directory`가 환경변수로만 전달되어 `train.py`를 찾지 못해 실패했다.
하이퍼파라미터 전달 방식으로 수정한 `-02`가 정상 완료됐으며 실패 Job은 감사 이력으로
남아 있다.

## 센서 데이터 마이그레이션

변경 전에 DynamoDB 온디맨드 백업을 생성했다.

- Lines backup: `01783923564106-93d73835`
- Bollards backup: `01783923564322-3a25153d`

자동 확정 가능한 항목만 이관했다. 배포 직후 관리자 화면에서 새 계류삭이 생성되고
등록 센서가 다시 배정되어, 최종 상태는 최신 관리자 작업을 소스 오브 트루스로 삼았다.

| 최종 Line | 현재 Bollard | 센서 상태 | 결과 |
|---|---|---|---|
| `mooring-20260713-f968ecc7` | `bollard-07` | `factorypulse_gateway_01` | 17:07 KST 확인, `DEVICE` 단일 배정 |
| `mooring-01` | `bollard-02` | `PENDING` | 과거 `AE-DEV-02`는 IoT Thing 없음 |
| `mooring-02` | `bollard-05` | `PENDING` | 과거 `asdasd`는 IoT Thing 없음 |

초기 이관 대상이던 `mooring-20260713-0da7701a`, `mooring-03`, `mooring-04`는 배포 후
관리자 작업에서 삭제됐다. 후속 관리자 작업으로 계류삭 ID가 여러 차례 변경됐으며
구체 ID를 장치 설정에 고정하지 않고 최신 Lines 테이블을 소스 오브 트루스로 삼는다.
이전 라인을 복원하지 않아 실제 센서는 정확히 한 라인만 소유한다. 과거
`AE-DEV-08`과 `bollard-11`의 VIRTUAL 값은 참고용 백업에만 남는다.

모든 Bollard 레코드에서 `sensorMode`, `sensorId`, `sensorBoundAt`, `deviceOnline`을
제거했다. 남은 미등록 센서 라인은 `PENDING`, `deviceOnline=false`,
`riskLevel=NORMAL`로 초기화해 과거 수치가 실시간 상태로 오인되지 않게 했다.

`factorypulse_gateway_01`은 IoT Registry에 실제 존재해 소프트웨어 귀속을 완료했으며,
Thing 속성을 `sensorKind=PIEZO`, `displayName=Mooring_Eye_Piezo_01`,
`berthId=berth-02`로 정리했다. 단, 실제 장비가 볼라드에서 해당 계류삭의 선박 측
아이·스플라이스 인근으로 물리적으로 이동·고정됐는지는 현장에서 확인해야 한다.

## 현장 체크리스트

- [ ] `factorypulse_gateway_01`을 관리 화면에서 현재 배정된 계류삭 연결 고리에 실제 재부착하고 고정 상태 확인
- [ ] `AE-DEV-02`, `AE-DEV-08`을 AWS IoT Thing으로 등록하거나 폐기 판단
- [ ] 테스트 문자열 `asdasd`는 실제 장비가 아니면 영구 폐기
- [ ] 각 실센서의 `source.deviceId`와 MQTT topic sensorId가 같고, Backend에서 현재 배정 `lineId`가 조회되는지 확인
- [ ] 계류삭 상세에서 실제 `freqHz` 수신과 `deviceOnline=true` 확인

## 롤백

- 데이터: 위 DynamoDB 온디맨드 백업으로 새 테이블 복원 후 검토
- 모델: 이전 endpoint config `ae-sentinel-port-window-cfg`로 `UpdateEndpoint`
- 프론트: Amplify main의 이전 성공 Job 4 아티팩트로 재배포
- 백엔드: CloudFormation 변경 세트/이전 Lambda 버전 기준으로 SAM 재배포
