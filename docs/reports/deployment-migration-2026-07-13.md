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
| Amplify | Job 5 `SUCCEED` | app `ddcrzjuu5yfnn`, branch `main` |
| Cognito | 프로덕션 callback/logout 추가 | `https://main.ddcrzjuu5yfnn.amplifyapp.com/` |

Amplify에서 내려받은 `index.html`은 로컬 프로덕션 빌드와 byte 단위로 일치했고,
JS 번들은 `index-BgObrW_0.js`, CSS는 `index-BcdxdGEd.css`다.

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
| `mooring-20260713-86204c88` | `bollard-07` | `factorypulse_gateway_01` | IoT Thing 확인, `DEVICE` 단일 배정 |
| `mooring-01` | `bollard-02` | `PENDING` | 과거 `AE-DEV-02`는 IoT Thing 없음 |
| `mooring-02` | `bollard-05` | `PENDING` | 과거 `asdasd`는 IoT Thing 없음 |

초기 이관 대상이던 `mooring-20260713-0da7701a`, `mooring-03`, `mooring-04`는 배포 후
관리자 작업에서 삭제됐다. 이를 복원하지 않았고, 잠시 복원되어 중복됐던 이전 라인은
조건부 삭제해 최종적으로 실제 센서는 정확히 한 라인만 소유한다. 과거
`AE-DEV-08`과 `bollard-11`의 VIRTUAL 값은 참고용 백업에만 남는다.

모든 Bollard 레코드에서 `sensorMode`, `sensorId`, `sensorBoundAt`, `deviceOnline`을
제거했다. 남은 미등록 센서 라인은 `PENDING`, `deviceOnline=false`,
`riskLevel=NORMAL`로 초기화해 과거 수치가 실시간 상태로 오인되지 않게 했다.

`factorypulse_gateway_01`은 IoT Registry에 실제 존재해 소프트웨어 귀속을 완료했으며,
Thing 속성을 `sensorKind=PIEZO`, `displayName=Mooring_Eye_Piezo_01`,
`berthId=berth-02`로 정리했다. 단, 실제 장비가 볼라드에서 해당 계류삭의 선박 측
아이·스플라이스 인근으로 물리적으로 이동·고정됐는지는 현장에서 확인해야 한다.

## 현장 체크리스트

- [ ] `factorypulse_gateway_01`을 `mooring-20260713-86204c88` 연결 고리에 실제 재부착하고 고정 상태 확인
- [ ] `AE-DEV-02`, `AE-DEV-08`을 AWS IoT Thing으로 등록하거나 폐기 판단
- [ ] 테스트 문자열 `asdasd`는 실제 장비가 아니면 영구 폐기
- [ ] 각 실센서의 `source.deviceId`, MQTT topic sensorId, 배정 `lineId` 일치 확인
- [ ] 계류삭 상세에서 실제 `freqHz` 수신과 `deviceOnline=true` 확인

## 롤백

- 데이터: 위 DynamoDB 온디맨드 백업으로 새 테이블 복원 후 검토
- 모델: 이전 endpoint config `ae-sentinel-port-window-cfg`로 `UpdateEndpoint`
- 프론트: Amplify main의 이전 성공 Job 4 아티팩트로 재배포
- 백엔드: CloudFormation 변경 세트/이전 Lambda 버전 기준으로 SAM 재배포
