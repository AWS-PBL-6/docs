# AWS 계정 복원 보고서 (2026-11-16)

> 상태: AWS 리소스 복원 완료(실장치 연결·SMS 수신자 등록·운영 데이터 재등록 대기), 2026-11-19 점검 반영
>
> 계정/리전: `814663755627` / `ap-northeast-2`, AWS CLI 프로필 `ae-sentinel-v2`
>
> 현행 운영 가이드: [AWS Application 스택 가이드](../operations/aws-application-account-guide.md)

## 1. 배경

2026-11-16 기존 계정 `727847798739`에서 AE-Sentinel의 Lambda 7개, DynamoDB 테이블 9개, SNS 토픽, S3 버킷이 삭제된 것을 확인했습니다.
CloudFormation 스택 `ae-sentinel-app`은 남아 있었지만 실제 리소스와 일치하지 않았습니다.
새 계정을 받아 저장소의 현재 코드 기준으로 전체 스택을 다시 배포했습니다.

## 2. 복원 결과

| 영역 | 결과 | 새 리소스 |
|---|---|---|
| SAM 스택 | `CREATE_COMPLETE` | `ae-sentinel-app` |
| REST API | `/docs` 200, 인증 없는 `/v1/lines` 401 | `https://7vrzqs3ekb.execute-api.ap-northeast-2.amazonaws.com` |
| SageMaker | `InService`, 스모크 요청 `DANGER`·riskScore `0.998` | endpoint `ae-sentinel-port-window` (서버리스 3,072 MB, 최대 동시성 5) |
| 모델 아티팩트 | 로컬 `line-eye-latest`로 재패키징 | `s3://ae-sentinel-sagemaker-814663755627/line-eye/models/ae-port-window-xgb-v1-20260713/` |
| Bedrock KB | 문서 8건 인제스트 성공(실패 0건), 벡터 2,577개 | KB `CPD8VIYV1U`, 데이터 소스 `A5UWKAXHIS` |
| RAG 종단 | `ae-sentinel-app-response-guidance`에 DANGER 샘플 입력 시 HTTP 200, 출처 `KB`, 신뢰도 `HIGH`, 인용 4개 | `ml/agent-response-layer/test/event-danger-line5.json` |
| 벡터 저장소 | 1,024차원 `float32`, cosine, AES256 | `ae-sentinel-kb-vectors-814663755627` / `ae-sentinel-kb-index` |
| KB 원본 버킷 | 메타데이터 사이드카 8건 업로드 | `s3://ae-sentinel-kb-814663755627/approved/korean/` |
| Amplify | Job 1 `SUCCEED`, SPA 경로 200 | app `d1cf48ghmi5y5c`, `https://main.d1cf48ghmi5y5c.amplifyapp.com/` |
| Cognito | 관리자 계정 1개 생성(`FORCE_CHANGE_PASSWORD`) | Hosted UI `ae-sentinel-814663755627` |
| DynamoDB | 볼라드 12건 복원 | `ae-sentinel-app-bollards` |
| SNS | 이메일 구독 확인 완료(2026-11-19 확인), SMS 번호 1개 샌드박스 인증 | `ae-sentinel-app-alerts` |
| IoT | Thing·인증서·정책 재발급, heartbeat Rule→Lambda 실행 확인 | Thing `factorypulse_gateway_01`, endpoint `a1jo5mo5y43cwq-ats.iot.ap-northeast-2.amazonaws.com` |

스모크 요청은 `ml/infra/ml-account/smoke-request.json`입니다.
2026-07-13 결과(`0.997`)와의 차이는 서빙 컨테이너의 XGBoost 2.1.4 사용 때문이며 판정은 같습니다.

## 3. 데이터 복원 범위

- DynamoDB 백업은 2026-07-13 센서 스키마 마이그레이션 직전 스냅샷(lines 5건, bollards 12건)만 남아 있었습니다.
- 계류삭 5건은 현재 스키마의 `sensorMode`·`sensorId`가 없고 존재하지 않는 선박·이벤트를 참조하여 복원하지 않았습니다.
- 볼라드 12건은 현재 `BollardRecord` 필드(`bollardId`, `siteId`, `berthId`, `positionM`, `label`, `createdAt`, `updatedAt`)만 남기고 복원했습니다.
- events, alerts, vessels, settings, sms-recipients 테이블은 백업이 없어 빈 상태로 시작합니다.
- KB 메타데이터 사이드카 8건은 기존 벡터 인덱스의 메타데이터에서 재생성했습니다.

## 4. KB 문서 출처

| 문서 | 출처 | 2026-07 색인본과의 비교 |
|---|---|---|
| 항만하역 통합안전매뉴얼, 공용부두·부두운영회사 표준안 2건 | 팀 Notion 첨부 | sha256 일치 |
| 항만하역사업장별 자체안전관리계획서 표준(안) | 팀 Notion 첨부 | 7월 해시 기록 없음 |
| 부산항 안전기준 매뉴얼 | 팀 Notion 첨부 원본에 `ocrmypdf -l kor+eng`와 `gs /ebook` 적용 | 66.7MB → 23.9MB, 한글 단어 수 7월 대비 약 76% |
| KOMSA 가이드, 수협 매뉴얼 | 메타데이터 `sourceUrl`의 공식 원출처 | sha256 일치 |
| 선박설비기준 별표 21의2 | 국가법령정보센터 별표 PDF | 해시 상이, 7월 색인 텍스트와 99.4% 일치 |

- 7월 색인본은 공용부두 하역사 표준안과 일반화물 부두운영회사 표준안의 파일 내용이 서로 바뀐 상태였습니다. 이로 인해 두 문서의 인용 제목이 반대로 표시되었습니다. 이번 복원에서는 첫 페이지 내용 기준으로 파일을 배치하고 사이드카의 `sourceSha256`을 실제 해시로 수정했습니다.
- 부산항 매뉴얼은 텍스트 레이어가 없는 디자인 PDF입니다. 400dpi 재래스터화에서도 인식 단어 수가 개선되지 않아 기본 OCR 결과를 사용했습니다.

## 5. 리소스 ID 변경

| 항목 | 기존 계정 | 새 계정 |
|---|---|---|
| 계정 | `727847798739` | `814663755627` |
| REST API | `kfethf9pug` | `7vrzqs3ekb` |
| Knowledge Base | `LWEXZE8ZFE` | `CPD8VIYV1U` |
| Amplify app | `ddcrzjuu5yfnn` | `d1cf48ghmi5y5c` |
| Cognito Hosted UI | `ae-sentinel-727847798739` | `ae-sentinel-814663755627` |
| IoT endpoint | 계정별 상이 | `a1jo5mo5y43cwq-ats` |

코드 반영 파일은 `backend/infra/application-account/samconfig.toml`, OpenAPI 스펙 2개, `edge-iot/publish_*.py`입니다.
`frontend/.env`는 커밋 대상이 아니므로 로컬 파일만 갱신했습니다.

## 6. 복원 이후 변경

| 날짜 | 변경 | 커밋 |
|---|---|---|
| 2026-11-18 | 허용 이메일 자체 회원가입. Pre sign-up Lambda `ae-sentinel-app-presignup`이 SSM `/ae-sentinel/signup-allowed-emails`를 조회함 | Backend `a2b21dd`, Frontend `b016478` |
| 2026-11-19 | 로그인 화면 비밀번호 찾기, 기존 세션이 남은 상태의 로그인 오류(`There is already a signed in user.`) 수정 | Frontend `73fa19c` |

운영 절차는 [AWS Application 스택 가이드](../operations/aws-application-account-guide.md)의 "사용자 계정 (Cognito)" 절이 기준입니다.

## 7. 2026-11-19 점검 결과

AWS CLI로 새 계정을 직접 조회한 결과입니다.

| 항목 | 결과 |
|---|---|
| SAM 스택 | `UPDATE_COMPLETE` |
| SageMaker endpoint | `InService` |
| Bedrock KB | `ACTIVE` |
| Amplify | job 5 `SUCCEED` |
| SNS 이메일 구독 | 확인 완료 |
| Cognito 사용자 | 2명(자체 가입 `CONFIRMED` 1명, 관리자 `FORCE_CHANGE_PASSWORD` 1명) |
| DynamoDB 건수 | bollards 12, vessels 1, lines 2, events 2, alerts 1, settings 0, sms-recipients 0 |
| 계류삭 센서 배정 | 2건 중 1건 `DEVICE`(`factorypulse_gateway_01`), 1건 `PENDING` |
| IoT 데이터 수신 | `ae-sentinel-app-ingest`, `ae-sentinel-app-ingest-piezo` 실행 0회(로그 그룹 없음) |

- vessels, lines, events, alerts 데이터는 복원 이후 관리 화면에서 새로 등록한 데이터입니다.
- 위험 이벤트 발생 시 이메일·SMS가 실제로 수신되는지는 아직 검증하지 않았습니다.

## 8. 미완료 항목

| 항목 | 필요 조치 | 담당 |
|---|---|---|
| 실장치 연결 | 게이트웨이 PC에서 Edge-IoT `main`의 `publish_sensor.py`(토픽 `aesentinel/{sensorId}/piezo`)를 사용하고 `certs/`를 새 인증서로 교체 | Edge 담당 |
| SMS 수신자 등록 | 관리 화면 SMS 수신자 메뉴에서 번호 등록. 샌드박스 상태라 SNS에서 인증한 번호만 수신 가능 | 운영자 |
| 선박·계류삭 등록 | 남은 선박과 계류삭을 등록하고 `PENDING` 계류삭에 센서 배정 | 운영자 |
| 관리자 비밀번호 설정 | 로그인 화면의 "비밀번호 찾기"로 새 비밀번호 설정 | 관리자 |
| 알림 종단 검증 | 실장치 연결과 SMS 수신자 등록 후 위험 이벤트에서 이메일·SMS 수신 확인 | 운영자 |
| 기존 계정 정리 | 남은 SageMaker endpoint, KB, S3 Vectors, Amplify, Cognito, CFN 스택 삭제 여부 결정 | 운영자 |

- 새 계정의 SNS SMS는 샌드박스 상태이며 월 지출 한도는 기본값 1 USD입니다.
- IoT 정책 `factorypulse_gateway_policy`는 기존과 같이 리소스 `*`를 허용합니다. 장치 수가 늘면 client ID와 토픽 단위로 범위를 줄이십시오.
