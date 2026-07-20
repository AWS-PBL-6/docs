# 저비용 구조 정량 분석 — 실 AE 계측 장비 도입 기준

> 작성일: 2026-07-16 (개정: 기준선을 데모 키트가 아닌 **실 AE 계측 장비 도입**으로
> 설정) · 용도: 기대효과 "나) 저비용 구조" 절의 정량 근거.
> 모든 단가에 출처를 병기하고 계산식을 노출함. AWS 단가는 2026-07 시점 공식 요금
> 페이지 기준(대표 리전 표기 시 서울 리전은 소폭 높을 수 있음 — §6 유의사항).
> 환율 가정: **1 USD = 1,400원, 1 EUR = 1,600원**.

## 1. 기준 시나리오 — 실 AE 계측 노드 도입 사양

현 PoC의 진동 프록시 키트가 아니라, **kHz 대역 음향방출(AE)을 실측하는 엣지 AE
노드**를 상용 도입 사양으로 가정함. 저비용 피에조 다이어프램 기반 AE 계측의 학술적
타당성은 기존 연구로 확인되어 있음
([Low-Cost AE Sensor 연구](https://www.researchgate.net/publication/340722569_A_Low-Cost_Acoustic_Emission_Sensor_based_on_Piezoelectric_Diaphragm)).

| 항목 | 가정값 | 근거 |
| --- | --- | --- |
| 적용 범위 | 1개 선석 (안벽 볼라드 8기) | 일반화물 선석의 통상 볼라드 배치를 단순화한 기준 배치 |
| 계측 노드 | 측점(계류삭)당 1식 × 8식 — 저가 공진형 AE 센서 + 고속 ADC(1~5 MSPS) + MCU(엣지 FFT·특징 추출) + Wi-Fi 직결 | 본 시스템 데이터 모델 + AE 실측 요건(수십~수백 kHz 대역) |
| 발행 방식 | 엣지에서 AE 특징(진폭·에너지·히트수·지속시간 등)을 추출해 기존 계약(`ae-feature-event.v2`)으로 발행 — **클라우드 파이프라인 변경 없음** | 계측·분석 경계가 이벤트 계약으로 추상화된 현행 아키텍처 |
| 월 메시지 수 | 8 × (86,400초/2) × 30일 = **10,368,000건** (특징 요약 2초 주기 상한 — AE 히트 이벤트 기반 발행 시 이보다 감소) | 계산식 · 보수적 상한 |
| 활성 이벤트 비율 | 5% (월 518,400건 ML 추론) | 유휴 시간 추론 생략 — 운영 정책 반영 |
| 경보 발송 | 월 50건 (SMS) | 시연·데모 운영 수준 가정 |

---

## 2. 초기 도입 비용 (실 AE 계측 하드웨어)

### 가) 기준 사양 — 저가 엣지 AE 노드 (상용 도입 기준선)

| 구성품 (측점당) | 산정 단가(개산) | 수량 | 소계 | 근거 |
| --- | ---: | ---: | ---: | --- |
| 저가 공진형/다이어프램 AE 센서 | $100 (약 14만 원) | 8 | 112만 원 | [저가 AE 센서 연구](https://www.researchgate.net/publication/340722569_A_Low-Cost_Acoustic_Emission_Sensor_based_on_Piezoelectric_Diaphragm) 접근 — 산업용(하단 표) 대비 약 1/10급 가정 |
| 고속 ADC(1~5 MSPS) + MCU (ESP32/STM32, 엣지 특징 추출) | $50 (약 7만 원) | 8 | 56만 원 | 범용 부품 시세 개산 |
| 방수 하우징 · 케이블 · 설치 자재 | $50 (약 7만 원) | 8 | 56만 원 | 개산 |
| **합계 (1선석 8측점)** | 측점당 약 $200 (28만 원) | | **약 224만 원** | |

- 별도 게이트웨이 불필요(Wi-Fi 직결) · **안벽 공사·훅 교체·전원 인프라 공사 없음**
  — 기존 볼라드에 부착만 (비교군과의 핵심 구조 차이, §4)
- 도입 시 모델은 실측 AE 특징으로 재학습 — 파이프라인(Processing→Training→
  Registry→Endpoint)은 운영 검증 완료, 재학습 과금 시간 150초(회당 수백 원)

### 나) 상향 옵션 — 인증급 산업 표준 AE 구성

감도·인증이 요구되는 고객용 상향 경로. 본 분석의 기준선이 아닌 옵션으로 병기.

| 구성품 (채널당) | 단가 | 근거 |
| --- | ---: | --- |
| 150 kHz 공진 AE 센서 (R15a급) | 약 $900 | 실거래 시세 $827~995 ([eBay 리스팅](https://www.ebay.com/shop/acoustic-emission-sensor?_nkw=acoustic+emission+sensor) · [제품 페이지](https://www.physicalacoustics.com/by-product/sensors/R15a-150-kHz-General-Purpose-AE-Sensor), 제조사 정가는 견적) |
| 프리앰프 + DAQ·신호처리 몫 | 채널 총액 $4,000 상한 | [NI 커뮤니티 견적 사례](https://forums.ni.com/t5/Multifunction-DAQ/Acoustic-Emission-DAQ-need/td-p/3760440): "센서+컨디셔너+DAQ < $4,000/채널" |
| **8채널 합계** | **$32,000 ≈ 약 4,500만 원** | |

---

## 3. 월 운영 비용 (AWS 종량 과금) — 계산식 노출

엣지에서 특징만 발행하므로 클라우드 비용 구조는 계측 하드웨어와 **독립적** —
AE 전환 후에도 동일한 상한이 적용됨.

| # | 서비스 | 공식 단가 | 계산식 | 월 비용 |
| --- | --- | --- | --- | ---: |
| 1 | IoT Core 메시지 | $1.00/백만 건 (5KB 단위, 특징 페이로드 <1KB=1건) — [요금](https://aws.amazon.com/iot-core/pricing/) | 10.368M × $1.0/M | $10.4 |
| 2 | IoT Core 규칙 엔진 | 규칙 실행+액션 각 $0.15/백만 — [요금](https://aws.amazon.com/iot-core/pricing/) | 10.368M × 2 × $0.15/M | $3.1 |
| 3 | Lambda 요청 | $0.20/백만 요청 — [요금](https://aws.amazon.com/lambda/pricing/) | 10.368M × $0.2/M | $2.1 |
| 4 | Lambda 컴퓨트 | $0.0000166667/GB-초 — [요금](https://aws.amazon.com/lambda/pricing/) | 유휴 95%: 9.85M×0.05초×0.5GB=246K GB-s → $4.1 / 활성 5%: 518K×1.5초×0.5GB=389K GB-s → $6.5 | $10.6 |
| 5 | SageMaker Serverless 추론 | 약 $0.000020/GB-초 (1GB 기준) — [요금](https://aws.amazon.com/sagemaker/pricing/) | 518K건 × 1.5초 × 3GB × $0.00002 | $46.7 |
| 6 | DynamoDB 온디맨드 쓰기 | $0.625/백만 WRU (2024-11 50% 인하) — [요금](https://aws.amazon.com/dynamodb/pricing/on-demand/) | 약 11M 쓰기 × $0.625/M + 읽기 소액 | $8.0 |
| 7 | API Gateway WebSocket | $1.00/백만 메시지 + 연결분 — [요금](https://aws.amazon.com/api-gateway/pricing/) | 실시간 push 약 11M × $1.0/M + 연결분 | $12.0 |
| 8 | SNS SMS (한국) | 통신사별 상이, 건당 약 $0.07~0.08 — [SMS 요금](https://aws.amazon.com/sns/sms-pricing/) | 50건 × $0.075 (+이메일: 10만 건까지 $2 미만) | $5.0 |
| 9 | S3 — AE 원파형 보관(재학습용 옵션) | $0.025/GB·월 — [요금](https://aws.amazon.com/s3/pricing/) | 히트당 수 KB × 월 52만 히트 ≈ 수 GB | $1.0 |
| 10 | Amplify 호스팅 · Cognito | 정적 서빙 $0.15/GB 등 — [요금](https://aws.amazon.com/amplify/pricing/) | 소규모 관제 인원 | $5.0 |
| | **월 합계** | | | **약 $104 ≈ 15만 원 (상한 17만 원)** |

**보수성 확인** — 위 추정이 상한인 이유:

- 5번(최대 항목)의 추론 1.5초는 **웜 상태 왕복 관측치** — 실제 모델 연산은 0.1초
  미만이므로 실과금은 이보다 낮음
- AE 노드는 히트 이벤트 기반 발행이 자연스러워 2초 주기 연속 발행(10.4M건/월)
  자체가 상한 — 실 발행량은 계류 활동 시간에 비례해 감소
- 유휴 시간 ML 추론 생략 정책이 배포되어 있음(피에조 인입 Lambda `TELEMETRY_ONLY`)
- **실측 교차검증**: 본 프로젝트 개발·검증 기간의 실제 AWS 청구액도 상기 추정과
  자릿수 일치 (PoC 트래픽에서 월 수십 달러대)

**연간 운영비 약 180만~200만 원/선석. 재학습 비용은 SageMaker Training 과금 시간
150초(ml.m5.large, 실배포 기록 2026-07-13)로 회당 수백 원.**

### 5년 총소유비용(TCO)

> **기준(저가 엣지 AE)**: 하드웨어 224만 원 + 운영 200만 원/년 × 5년 =
> **약 1,220만 원 / 선석**
> **옵션(인증급 AE)**: 4,500만 원 + 1,000만 원 = 약 5,500만 원 / 선석

---

## 4. 비교군 대비 정량 비교 — 출처 병기

### 가) 진공 자동계류 (Cavotec MoorMaster)

| 근거 | 내용 |
| --- | --- |
| 실계약 1 | 2개 선석 EUR 10.3M 수주 → **선석당 약 EUR 5.15M** ([Cavotec 보도자료](https://press.cavotec.com/pressreleases/cavotec-s-moormaster-automated-mooring-technology-registers-two-major-breakthroughs-1440150)) |
| 실계약 2 | 모로코 컨테이너 터미널 1건 EUR 5M ([Marine Log](https://www.marinelog.com/news/cavotec-wins-a-further-automated-mooring-system-order/)) |
| 환산 | EUR 5.15M × 1,600원 = **약 82억 원/선석** |

→ 기준 사양 5년 TCO(약 1,220만 원) 대비 **약 1/670**, 인증급 구성 기준으로도 약
1/150. 단, 기능 범위가 다름(자동 계류 vs 파단 조기경보) — §6 유의사항.

### 나) 하중 감시 계선훅 (Trelleborg SmartHook / Straatman QRH+LMS)

| 근거 | 내용 |
| --- | --- |
| 구성 | QRH 피벗 로드셀 + 컨트롤러 + 모니터링 SW — 훅 유닛 교체 또는 개조 공사 수반 ([Trelleborg SmartHook](https://www.trelleborg.com/en/marine-and-infrastructure/products-solutions-and-services/marine/docking-and-mooring/mooring-load-monitoring/smarthook-load-monitoring-system), [Straatman](https://straatmanmooringsystems.com/mooring-systems/quick-release-mooring-hook/)) |
| 단가 | 공개 정가 없음(프로젝트 견적 기반) — 금액 비교 대신 **구조 비교**로 서술 |

→ 구조 차이: ① 훅 교체·안벽 공사 수반 vs **부착식(공사 0)**, ② QRH 미설치 재래
부두 적용 불가 vs 기존 볼라드 그대로, ③ 하중(현재 장력)만 측정 vs **kHz AE 파단
전조 패턴 실측**.

### 다) 산업용 AE 전용 계측 시스템 (상시 계측 서버형)

같은 "AE 실측"이라는 기능 기준에서의 직접 비교군.

| 항목 | 전용 계측 시스템 | AE-Sentinel (기준 사양) |
| --- | --- | --- |
| 측점당 하드웨어 | 채널당 약 $4,000 ≈ 550만 원 ([NI 견적 사례](https://forums.ni.com/t5/Multifunction-DAQ/Acoustic-Emission-DAQ-need/td-p/3760440)) — 8채널 약 4,500만 원 | 측점당 약 28만 원 — 8식 약 224만 원 (**약 1/20**) |
| 분석 인프라 | 상시 가동 계측 서버 + 분석 SW (다채널 상용 SHM: Physical Acoustics [Sensor Highway/DaisAE](https://www.physicalacoustics.com/by-product/daisae-structural-health-monitoring-system/) 계열, 견적) | 서버리스 — 유휴 컴퓨트 0, 월 약 15만 원 종량 과금 |
| 경보·관제 | 별도 구축 필요 | 3D 관제·이메일/SMS·RAG 지침 포함 |

→ 비용 격차의 원천: ① 고정밀 인증급 센서 대신 **엣지 특징 추출로 충분한 저가
센서** 채택(판정은 클라우드 ML), ② 상시 계측 서버를 **서버리스로 대체**.

### 라) 전력 소모 비교 (가정 근거 명시)

| 구성 | 산정 근거 | 상시 전력 | 연간 전력량 |
| --- | --- | ---: | ---: |
| AE-Sentinel 엣지 AE 노드 8식 | 센서·프리앰프 약 0.5W + MCU/ADC·Wi-Fi 약 1W = 측점당 약 1.5W 가정 | 12W | 약 105 kWh |
| 전용 계측 장비 + 온프레미스 서버 | 다채널 DAQ + 1U급 상시 서버의 통상 소비전력 100~300W 중 중간값 가정 | 200W | 약 1,752 kWh |

→ 연간 전력량 **약 1/17** (계산: 12W×8,760h=105kWh · 200W×8,760h=1,752kWh).
클라우드 측은 서버리스라 유휴 컴퓨트가 없어 사용량 비례 소비만 발생.

---

## 5. 결론 문장 (보고서 반영용)

> 실 AE 계측 노드 도입 기준(1선석·측점 8식) 초기 하드웨어 약 224만 원(측점당 약
> 28만 원, 개산), 월 운영 약 15만~17만 원(AWS 공식 요금표 단가 × 사용량 계산식
> 합산, 추정 상한)으로 5년 총소유비용은 선석당 약 1,220만 원임. 공개 계약 사례
> 기준 선석당 약 82억 원(EUR 5.15M)인 진공 자동계류 대비 **약 1/670**, 동일하게
> AE를 실측하는 전용 계측 시스템(채널당 약 550만 원 + 상시 계측 서버) 대비 측점당
> **약 1/20**이며, 훅 교체 공사가 전제인 하중 감시 계선훅과 달리 기존 볼라드에
> 부착만으로 도입됨. 비용 격차의 원천은 엣지 특징 추출 + 클라우드 ML 판정 구조로
> 고정밀 계측기와 상시 서버를 대체한 데 있으며, 상시 서버가 없는 종량 과금 구조로
> 유휴 시간 비용과 전력(연간 약 1/17)을 동시에 제거함.

---

## 6. 산정 유의사항 (질의응답 대비)

1. **개산 단가**: 기준 사양의 측점당 $200은 부품 시세 기반 개산 — 양산 설계·인증
   수준에 따라 변동. 인증급 요구 시 §2-나 옵션(채널당 $4,000 상한, 8채널 4,500만
   원)으로 상향해도 자동계류 대비 약 1/150 유지
2. **리전 단가**: AWS 표기 단가는 대표(버지니아) 기준 포함 — 서울 리전은 서비스별
   0~20% 높을 수 있으나 보수적 사용량 가정이 이를 상쇄, "월 십수만 원대" 결론 유지
3. **MoorMaster 비교의 성격**: 자동 계류와 파단 조기경보는 기능이 다름 — "계류
   안전에 투입되는 선석당 설비 예산" 관점이며 동일 기능 대체재 주장이 아님
4. **SmartHook류**: 공개 정가 없음 — 공사 수반 여부라는 구조 차이로만 비교할 것
5. **저가 AE 센서의 한계**: 인증급(R15a) 대비 감도·재현성 열위 가능 — "연구·운영용
   실측"으로 포지셔닝하고 인증 요구 고객에는 상향 옵션 제시
6. **SMS 비용**: 한국행 통신사별 단가 상이(약 $0.07~0.08/건, 공식 CSV). 경보량
   증가 시 선형 증가, 프로덕션 전환 시 샌드박스 해제·한도 상향 필요
7. **모델 재학습**: AE 실측 전환 시 실측 특징으로 재학습 필요 — 파이프라인 운영
   검증 완료 상태(회당 과금 150초)라 절차·비용 리스크 낮음

---

## 부록 — 현행 PoC 검증용 키트 비용 (참고)

본 분석의 기준선은 실 AE 도입 사양이며, 아래는 파이프라인 검증에 사용한 현행 PoC
키트의 실비용(참고용).

| 구성품 | 판매가 범위 | 수량 | 소계 | 출처 |
| --- | --- | ---: | ---: | --- |
| 피에조 압전 소자 | 약 500~2,000원 | 8 | 1.6만 원 | 국내 부품몰 판매가 |
| Arduino Uno 호환보드 | 약 7,000~20,000원 | 8 | 12만 원 | 다나와 가격비교 |
| HC-06 블루투스 모듈 | 약 4,000~9,000원 | 8 | 6.4만 원 | [다나와 HC-06](https://prod.danawa.com/info/?pcode=15497894) |
| 배선·LED·케이스 | 일괄 | 8 | 4만 원 | 개산 |
| **합계** | | | **약 24만 원** | |

→ "24만 원으로 파이프라인 전 구간을 실기기 검증하고, 224만 원으로 실측 사양에
도달하는" 단계적 투자 구조 자체가 본 프로젝트의 저비용 전략임.

---

**출처 일람**: [AWS IoT Core 요금](https://aws.amazon.com/iot-core/pricing/) ·
[AWS Lambda 요금](https://aws.amazon.com/lambda/pricing/) ·
[Amazon SageMaker 요금](https://aws.amazon.com/sagemaker/pricing/) ·
[DynamoDB 온디맨드 요금](https://aws.amazon.com/dynamodb/pricing/on-demand/) ·
[API Gateway 요금](https://aws.amazon.com/api-gateway/pricing/) ·
[SNS SMS 요금](https://aws.amazon.com/sns/sms-pricing/) ·
[S3 요금](https://aws.amazon.com/s3/pricing/) ·
[Amplify 요금](https://aws.amazon.com/amplify/pricing/) ·
[R15a AE 센서 제품 페이지](https://www.physicalacoustics.com/by-product/sensors/R15a-150-kHz-General-Purpose-AE-Sensor) ·
[AE 센서 실거래 시세(eBay)](https://www.ebay.com/shop/acoustic-emission-sensor?_nkw=acoustic+emission+sensor) ·
[NI 커뮤니티 — AE DAQ 구성 비용](https://forums.ni.com/t5/Multifunction-DAQ/Acoustic-Emission-DAQ-need/td-p/3760440) ·
[Physical Acoustics SHM 시스템](https://www.physicalacoustics.com/by-product/daisae-structural-health-monitoring-system/) ·
[저가 피에조 다이어프램 AE 센서 연구](https://www.researchgate.net/publication/340722569_A_Low-Cost_Acoustic_Emission_Sensor_based_on_Piezoelectric_Diaphragm) ·
[Cavotec 보도자료](https://press.cavotec.com/pressreleases/cavotec-s-moormaster-automated-mooring-technology-registers-two-major-breakthroughs-1440150) ·
[Marine Log — Cavotec 수주](https://www.marinelog.com/news/cavotec-wins-a-further-automated-mooring-system-order/) ·
[Trelleborg SmartHook](https://www.trelleborg.com/en/marine-and-infrastructure/products-solutions-and-services/marine/docking-and-mooring/mooring-load-monitoring/smarthook-load-monitoring-system) ·
[다나와 HC-06 가격비교](https://prod.danawa.com/info/?pcode=15497894)
