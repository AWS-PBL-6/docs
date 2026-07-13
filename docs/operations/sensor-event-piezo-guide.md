# 센서 온보딩 · 이벤트 처리 · 피에조 주파수 운영 가이드

> 분류: 현행 운영 가이드 · IoT/Backend

## 1. 연결 가능한 센서 목록을 현재 시스템이 판단하는 방법

센서 인벤토리의 소스 오브 트루스는 **AWS IoT Core Thing Registry**다.
백엔드 `GET /v1/sensors`가 실제 등록된 피에조 압전센서 목록을 조회하고,
`GET /v1/lines`의 현재 배정 정보만 조합해 반환한다.

- `sensorKind=PIEZO`
- `source=AWS_IOT_CORE`
- `registrationStatus=REGISTERED`
- `available=true|false` 및 현재 배정 계류삭
- Fleet indexing 사용 시 `connectivityStatus=CONNECTED|DISCONNECTED`
- Fleet indexing 미사용 시 `UNKNOWN` (등록됨을 온라인으로 오인하지 않음)

프론트는 관리자가 센서 ID를 직접 입력하지 못하게 하고, 이 목록에서 장비를
클릭한 뒤 적용하도록 한다. 이전 볼라드 레코드에 남은 수동/목업 센서 ID는
마이그레이션 참고값일 뿐 연결 가능한 장비 목록에 표시되지 않는다. 백엔드는 동일 `sensorId`를 다른
계류삭에 다시 묶는 요청도 `409 SENSOR_IN_USE`로 거부한다.

센서는 각 계류삭의 선박 측 연결 고리(아이·스플라이스 인근)에 한 대씩 부착한다.
볼라드는 안벽 연결 위치만 나타내며 센서를 소유하지 않는다.

## 2. 이벤트 발생 조건과 현재 상태 의미

### 2.1 발생 흐름

이벤트는 다음 흐름으로 생성된다.

1. IoT Core 가 AE 특징 이벤트 또는 피에조 주파수 이벤트를 수신
2. 피에조면 `ingest-piezo` 가 진동수(`freqHz`)를 AE 특징량으로 변환
3. `inference_service.run()` 이 최근 윈도우를 묶어 SageMaker(또는 로컬 모델) 추론
4. 결과를 `EventRecord` 로 저장하고, `LineStatus` 최신 상태를 갱신
5. WebSocket 으로 대시보드에 즉시 push
6. 최종 위험도가 `DANGER` 이면 스냅백 구역 계산 + SNS 경보 발송

### 2.2 현재 위험도 판정 기준

현재 기본 temporal policy 는 다음과 같다.

- `warnThreshold`: `0.35`
- `dangerThreshold`: `0.70`
- `highConfidenceDanger`: `0.85`
- 최근 5개 이벤트 윈도우 기준 EMA/히트 수를 함께 반영

따라서 최종 이벤트 상태는 **단일 센서값 1회**만으로 정해지지 않고,
모델 점수 + 최근 이벤트 추세를 함께 본다.

### 2.3 `미확인` 과 `확인(Ack)` 의 현재 의미

현재 백엔드에는 이벤트 확인 여부와 별도로 처리 상태가 저장된다.

- `acknowledged = false`: 아직 운영자가 확인하지 않음
- `acknowledged = true`: 운영자가 경보를 확인함
- `workflowStatus = NEW`: 신규
- `workflowStatus = ACKNOWLEDGED`: 확인
- `workflowStatus = IN_PROGRESS`: 조치 중
- `workflowStatus = RESOLVED`: 처리 완료
- `workflowStatus = FALSE_POSITIVE`: 오탐 종결

중요한 점:

`Ack` 는 처리 완료가 아니라 확인 기록이다. 조치 중/처리 완료/오탐 종결은
`workflowStatus`, `workflowNote`, `handledBy` 로 별도 관리한다.

## 3. SNS / SMS 연동 방안

현재 백엔드는 `DANGER` 이벤트에서 `alert_service.dispatch()` 를 통해 기존 SNS topic
발행을 유지하고, `/v1/sms-recipients` 테이블의 활성/동의 수신자에게 직접 SMS 를
fan-out 한다.

현 시점 AWS 공식 문서 기준:

- Amazon SNS SMS 는 AWS End User Messaging SMS 를 통해 실제 문자 전송을 수행한다.
- SNS 토픽에 전화번호 구독자를 붙여 다수 번호로 fan-out 할 수 있다.
- SMS 는 수신자 동의(opt-in), 발송 제한, 수신거부(opt-out) 관리가 필요하다.

공식 문서:

- https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- https://docs.aws.amazon.com/sns/latest/dg/channels-sms-best-practices.html

### 3.1 현재 구현

- `GET/POST /v1/sms-recipients`
- `GET/PUT/DELETE /v1/sms-recipients/{recipientId}`
- `enabled=true` 이고 `smsOptIn=true` 인 수신자만 대상
- `siteId`/`berthId` 가 비어 있으면 전체 대상
- 값이 있으면 이벤트의 `siteId`/`berthId` 와 일치할 때만 발송
- 발송 성공 시 `lastNotifiedAt` 갱신

기존 SNS topic 발행도 유지되므로 이메일 구독자나 다른 SNS 구독자는 계속 동작한다.

### 3.2 이후 확장안

SNS 토픽은 유지하고, 별도 `AlertFanout` Lambda 를 구독시켜 수신 대상을 관리한다.

권장 구조:

1. `DANGER` 이벤트 발생
2. 기존 SNS topic 발행
3. `AlertFanout` Lambda 가 topic 구독
4. 수신자 테이블(DynamoDB)에서 선석/조/역할 기준 대상 조회
5. 동의(opt-in)된 번호에만 SMS 발송
6. 발송 결과/실패 사유/재시도 이력 저장

이 방식의 장점:

- 선석별, 야드별, 교대조별 라우팅 가능
- 수신거부/동의 이력 관리 가능
- 프론트에서 “누가 어떤 경보를 받는지” 관리 UI 확장 가능

## 4. 피에조(압전) 센서 주파수 기준선

현재 코드 기준 기본 보정값은 아래와 같다.

- `PIEZO_FREQ_SLACK_HZ = 8.0`
- `PIEZO_FREQ_CRIT_HZ = 32.0`

피에조 어댑터는 taut-cable 관계 `T ∝ f²` 를 사용해 진동수를 장력비로 바꾼다.

```text
rho = (f^2 - 8^2) / (32^2 - 8^2)
```

프론트에 표시한 운영 가이드 밴드는 이 장력비를 3구간으로 나눈 값이다.

- 정상: `< 19.5 Hz`
- 주의: `19.5 ~ 26.4 Hz`
- 위험: `>= 26.4 Hz`
- 임계 상단 참고선: `32.0 Hz`

주의:

- 이 값은 **센서 운영 가이드선**이다.
- 실제 이벤트의 `NORMAL/CAUTION/DANGER` 는 이 값만으로 확정되지 않는다.
- 최종 판정은 ML 추론 + 최근 5개 이벤트의 temporal policy 를 함께 반영한다.

참고로 현재 backend 의 `dangerThreshold=0.70`, `highConfidenceDanger=0.85` 를
주파수 축으로 단순 환산하면 대략 아래 수준이다.

- DANGER 승격이 본격화되는 구간: 약 `27.1 Hz`
- 고신뢰 DANGER 직행 구간: 약 `29.6 Hz`

하지만 이 값 역시 모델 응답과 윈도우 추세에 따라 달라질 수 있으므로,
운영 문서에서는 다음처럼 구분해 쓰는 것이 안전하다.

- **센서 가이드선**: `19.5 / 26.4 / 32.0 Hz`
- **최종 이벤트 판정**: ML + temporal policy 결과

## 5. 프론트 반영 포인트

이번 프론트 수정은 다음을 목표로 한다.

- 볼라드 상세에서는 연결 계류삭과 안벽 위치만 관리
- 계류삭 생성/상세에서 현재 연동 중인 실센서 상태 표시
- AWS IoT 피에조 센서 목록 클릭 → 선택 센서 적용(ID 직접 입력 제거)
- 중복 `sensorId` 사전 차단
- 이벤트 `source.deviceId`와 계류삭 배정 `sensorId` 불일치 차단
- 실센서 이벤트가 없을 때 목업 수치 대신 수신 대기 표시
- 이벤트 상세에서 `Ack` 의미를 명시
- 피에조 계류삭 상세에서 주파수 정상/주의/위험 기준선 표시
