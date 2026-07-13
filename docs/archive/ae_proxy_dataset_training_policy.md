> ⚠️ **보관됨(SUPERSEDED) · proxy 데이터셋 학습·임계값 정책 (2026-07-09)**
> 이 문서는 **Kaggle 항공기 복합재 AE 데이터셋(AE-Damage-Detection)** 기반의 초기 proxy 접근을 기록한 것으로, 더 이상 현행 방법이 아니다. 현재는 계류삭 AE 파단 실측 논문(Bashir et al. 2017)을 근거로 한 합성 데이터셋과 그 항만 도메인 변환본을 사용한다.
> 현행 문서: [홋줄 AE 데이터셋 평가](../ml-data/rope-ae-dataset-evaluation.md) · [계류삭 연결 고리 AE 데이터셋](../ml-data/port-ae-dataset.md) · [데이터셋 신뢰성 검증 보고서](../ml-data/dataset-reliability-report.md)

---

# AE Proxy Dataset 학습/임계값/발표 대응 보고서

작성일: 2026-07-09  
대상 프로젝트: AE-Sentinel  
관련 문서:

- [AE-Sentinel 권장 AWS 전체 아키텍처](../architecture/ae_sentinel_recommended_architecture.md)
- [AE-Sentinel 저장소 구조 및 인터페이스 계약](../contracts/aws-system-repository-and-interface-contract.md)
- [부산대학교 클라우드 AI PBL 프로젝트 기획서](../reports/pnu-cloud-ai-pbl-proposal-parsed.md)

---

## 1. 목적

본 문서는 Kaggle `Acoustic Emission Dataset`를 이용해 AE-Sentinel의 1차 ML PoC를 수행할 때,
다음 세 가지를 동시에 만족시키기 위한 기준 문서다.

1. 모델의 본질을 `섬유파단(Fiber Breakage) 유사 신호 검출`로 유지한다.
2. 서비스는 `정상(NORMAL) / 경고(CAUTION) / 위험(DANGER)` 3단계 경보를 제공한다.
3. 발표/질의응답에서 "왜 항공기 복합재 데이터셋으로 선박 계류삭 위험감지를 했는가"에 대해
   과장 없이 방어 가능한 설명을 제공한다.

---

## 2. 현 상태와 제약

### 2.1. 데이터셋 설명

Kaggle 데이터셋 설명상 이 데이터는 복합재 항공기 부품의 AE 센서 데이터를 기반으로 하며,
손상 라벨은 다음 네 가지다.

- `0`: No Damage
- `1`: Matrix Cracking
- `2`: Fiber Breakage
- `3`: Delamination

### 2.2. 로컬 CSV 확인 결과

`/Users/mac/Downloads/AE_Damage_Detection_Dataset.csv`를 기준으로 확인한 결과는 다음과 같다.

- 총 행 수: `1000`
- 결측치: `0`
- 완전 중복 행: `0`
- 클래스 분포:
  - `0`: `418`
  - `1`: `288`
  - `2`: `200`
  - `3`: `94`

### 2.3. 프로젝트 스키마 제약

현재 실시간 추론 스키마는 다음 필드 중심이다.

- `peakAmplitude`
- `energy`
- `dominantFrequencyHz`
- `riseTimeMs`
- `durationMs`
- `tensionEstimateKn`
- `temperatureC`
- `humidityPct`

관련 계약:

- [iot/ae-feature-event.v1.json](../../packages/contracts/schemas/iot/ae-feature-event.v1.json)
- [ml/inference-request.v1.json](../../packages/contracts/schemas/ml/inference-request.v1.json)
- [ml/inference-response.v1.json](../../packages/contracts/schemas/ml/inference-response.v1.json)

즉, 학습 feature도 가능한 한 이 런타임 스키마와 맞춰야 한다.  
특히 `Load_Condition`과 `Impact_Energy_J`는 현재 서비스 입력 계약에 직접 존재하지 않으므로,
학습에 강하게 의존하면 train/serve mismatch가 커진다.

---

## 3. 최종 권장 전략

### 3.1. 핵심 원칙

가장 손이 덜 가고, 동시에 발표 명분이 가장 깔끔한 방식은 다음이다.

1. **모델은 이진 분류로 유지한다.**
2. **핵심 positive는 `Damage_Type == 2 (Fiber Breakage)`로 한정한다.**
3. **`1(Matrix Cracking)`과 `3(Delamination)`은 모델의 핵심 학습 라벨에 넣지 않는다.**
4. **서비스의 3단계 경보는 모델 score와 시간축 규칙으로 만든다.**

### 3.2. 왜 이렇게 하는가

이유는 명확하다.

- `2`는 "섬유파단 유사 AE 패턴"이라는 여러분의 핵심 주장과 가장 직접적으로 맞닿아 있다.
- `1`과 `3`을 학습 positive에 넣으면 모델이 `섬유파단 검출기`가 아니라
  `항공 복합재 일반 손상 검출기`로 해석되기 쉽다.
- 반대로 `1`과 `3`을 학습 negative에 넣어도 의미가 흐려진다.
  정상도 아니고, 그렇다고 여러분이 직접 찾고 싶은 핵심 파단 정답도 아니기 때문이다.

따라서 **핵심 학습은 `0` vs `2`**,  
**`1`과 `3`은 shadow/reference set으로 두는 것**이 가장 방어 가능하다.

---

## 4. 학습 데이터 구성표

### 4.1. 최종 권장 구성

| 구분 | 포함 라벨 | 행 수 | 용도 | 사용 여부 |
|---|---:|---:|---|---|
| Core Negative | `0` (No Damage) | `418` | 정상 기준 학습 | 사용 |
| Core Positive | `2` (Fiber Breakage) | `200` | 핵심 위험 학습 | 사용 |
| Shadow Abnormal A | `1` (Matrix Cracking) | `288` | 경고 구간 검증 참고 | 학습 제외 |
| Shadow Abnormal B | `3` (Delamination) | `94` | 경고/고위험 spill-over 검증 참고 | 학습 제외 |

### 4.2. 학습/검증/평가 구조

| 단계 | 데이터 | 목적 |
|---|---|---|
| 모델 학습 | `0` + `2` | 정상 vs 섬유파단 유사 이진 모델 학습 |
| 핵심 성능 평가 | `0` + `2` | Recall, FNR, Precision, F1, PR-AUC 평가 |
| 운영 정책 검증 | `1` + `3` | 경고 구간에 얼마나 잘 걸리는지 그림자 평가 |
| 최종 서비스 점검 | 전체 `0/1/2/3` score 분포 | 정상/경고/위험 threshold sanity check |

### 4.3. split 권장안

데이터가 많지 않으므로 고정 `70/15/15`보다 아래 방식이 낫다.

1. `0`과 `2`만 대상으로 `Stratified 5-Fold CV` 수행
2. 각 fold의 out-of-fold 확률(score) 수집
3. 이 OOF score로 핵심 threshold를 추정
4. 같은 학습된 설정으로 `1`과 `3`에 대해 shadow score 분석

이 방법은 작은 데이터셋에서 표본 낭비가 적고, 구현도 과도하게 복잡하지 않다.

---

## 5. 전처리 정책

### 5.1. 유지할 feature

현행 서비스 스키마와 가장 잘 맞는 1차 feature는 아래와 같다.

| 원본 컬럼 | 유지 여부 | 이유 |
|---|---|---|
| `Peak_Amplitude_dB` | 유지 | 런타임 `peakAmplitude`와 대응 |
| `Energy_Release_J` | 유지 | 런타임 `energy`와 대응 |
| `Signal_Duration_ms` | 유지 | 런타임 `durationMs`와 대응하며, 단독 상관은 낮지만 v1 구현에서는 런타임 정합성과 추후 파생 feature를 위해 포함 |
| `Rise_Time_ms` | 유지 | 런타임 `riseTimeMs`와 대응 |
| `Frequency_Spectrum_Hz` | 유지 | 런타임 `dominantFrequencyHz`와 대응 |
| `Temperature_C` | 유지 | 런타임 `temperatureC`와 대응 |
| `Humidity_%` | 유지 | 런타임 `humidityPct`와 대응 |

### 5.2. 1차 제외 권장 feature

| 원본 컬럼 | 제외 이유 |
|---|---|
| `Load_Condition` | 런타임 추론 입력 계약에 없음 |
| `Impact_Energy_J` | `tensionEstimateKn`와 의미가 다르며 직접 치환하기 어려움 |
| `Fiber_Type` | 항공 복합재 재질 정보가 강하고 rope 도메인 설명력이 약함 |
| `Resin_Type` | 항공 복합재 고유 속성에 가깝고 서비스 입력에도 없음 |
| `Thickness_mm` | rope 현장 추론에 직접 대응되지 않음 |

### 5.3. 중요한 원칙

- `Impact_Energy_J`를 `tensionEstimateKn`로 이름만 바꿔 쓰면 안 된다.
- `Load_Condition`을 유지한 모델은 현장 서비스에 바로 연결하기 어렵다.
- 현재 PoC에서는 **train/serve parity**를 우선한다.
- `Signal_Duration_ms`는 단독 상관이 낮아도 곧바로 삭제하지 않는다. v1 구현에서는 런타임 정합성과
  추후 파생 feature(`energy/duration`, `rise/duration`) 가능성을 이유로 포함한다.
- proxy dataset 기반 v1 구현에서는 **학습 범위를 벗어난 입력값을 feature별 min/max로 clip**해
  out-of-domain 외삽이 과도하게 폭주하지 않도록 제한한다.

### 5.4. 추천 파생 feature

파생 feature는 많을수록 좋은 것이 아니라, **현장에서도 동일하게 계산 가능한 것만** 사용해야 한다.

1차 후보:

- `log_energy = log(1 + Energy_Release_J)`
- `energy_per_duration = Energy_Release_J / Signal_Duration_ms`
- `rise_ratio = Rise_Time_ms / Signal_Duration_ms`
- `freq_x_rise = Frequency_Spectrum_Hz * Rise_Time_ms`
- `temp_x_humidity = Temperature_C * Humidity_%`

단, 첫 실험에서는 원본 7개 feature baseline을 먼저 만들고,
그 다음에 파생 feature를 하나씩 추가해 비교하는 것이 낫다.

---

## 6. 권장 학습 절차

### 6.1. 모델 선택

가장 적은 수고로 시작하려면 아래 순서를 추천한다.

1. `Logistic Regression` baseline
2. `CatBoostClassifier` main model

권장 이유:

- Logistic Regression: 가장 단순한 기준선
- CatBoost: 작은 tabular 데이터에서 전처리 부담이 상대적으로 적고, 확률 score 활용이 쉽다

### 6.2. 불균형 처리

핵심 학습셋 `0 vs 2`의 비율은 `418 : 200`으로 약 `2.09 : 1`이다.  
이는 극단적 불균형은 아니므로, 1차 실험에서는 아래 순서가 적절하다.

1. 무보정 baseline
2. `class_weight` 또는 `scale_pos_weight`
3. threshold 조정

권장하지 않는 시작점:

- 초기부터 무조건 `SMOTE`
- 초기부터 무조건 `ADASYN`

이유:

- 샘플 수가 작고 도메인 갭이 큰 상황에서 synthetic sample이 오히려 잘못된 minority 신호를 만들 수 있다.
- oversampling은 나중에 보조 실험으로 두는 편이 안전하다.

### 6.3. 성능 평가 지표

핵심 평가는 `0 vs 2` 기준으로 한다.

- `Recall`
- `False Negative Rate`
- `Precision`
- `F1`
- `PR-AUC`
- `Confusion Matrix`

이 프로젝트는 안전 시스템이므로 `Accuracy`를 주 지표로 쓰지 않는다.

---

## 7. Threshold 정책표

### 7.1. 기본 철학

현재 권장 구조는 **단일 확률 하나만 쓰는 방식보다, 하이브리드 score + 정책 레이어**다.

모델/런타임은 최소한 아래 정보를 만든다.

- `breakageScore`: `0 vs 2` core 분류기의 섬유파단 유사 확률
- `dangerSimilarity`: 과거 `Damage_Type=2` exemplar와의 유사도
- `normalSimilarity`: 과거 `Damage_Type=0` exemplar와의 유사도
- `oodScore`: 현재 입력이 학습 manifold 밖에 있는지 보는 점수
- `hybridScore`: 위 요소를 합성한 서비스용 위험 score

즉:

- **모델 역할**: 섬유파단 유사성에 대한 핵심 score와 보조 진단값 생성
- **정책 레이어 역할**: `OOD`, 시간축, hysteresis를 반영해 현장형 경보 단계 결정

PoC v1의 예시 결합식은 다음과 같다.

`hybridScore = (0.6 * breakageScore + 0.4 * dangerSimilarity) * (1 - 0.3 * oodScore)`

### 7.2. 1차 권장 정책

| 단계 | 조건 | 의미 | 시스템 동작 |
|---|---|---|---|
| `NORMAL` | `hybridScore < T_warn` | 정상 범위 | 상태 저장만 수행 |
| `CAUTION` | `T_warn <= hybridScore < T_danger` | 위험 전조 또는 애매한 비정상 | 대시보드 경고, 현장 재확인 |
| `DANGER` | `hybridScore >= T_danger` | 섬유파단 유사 신호 강함 | 즉시 위험 경보 후보 |

단, 실제 서비스에서는 아래 보정이 함께 들어간다.

- `OOD`가 높으면 단발 `DANGER`를 우선 `CAUTION`으로 낮춘다.
- 동일 라인에서 high score가 반복되면 `CAUTION -> DANGER`로 승격한다.
- `highConfidenceDanger`를 넘는 경우만 즉시 `DANGER`를 허용한다.

### 7.3. threshold 산정 방법

#### `T_danger`

`0 vs 2` 검증 `hybridScore`에서 다음 조건을 우선 만족시키는 threshold로 정한다.

1. `Recall(class 2) >= 0.90`
2. 위 조건을 만족하는 threshold 중 `Precision` 또는 `F1`이 가장 좋은 값 선택

#### `T_warn`

`T_warn`는 모델이 직접 학습한 클래스가 아니라 운영 경보용 구간이다.  
따라서 아래 방식으로 정한다.

1. `0` 클래스 검증 `hybridScore`의 상위 백분위(예: `95th percentile`)를 계산
2. 그 값을 `T_warn`의 초기 후보로 둔다
3. `1`과 `3` shadow set `hybridScore`를 넣어 보았을 때, 상당수가 `T_warn` 이상으로 올라오는지 확인
4. 정상 오탐이 과도하지 않은 범위에서 미세 조정한다

핵심은:

- `T_danger`는 `0 vs 2`로 정한다
- `T_warn`는 `0` 배경 점수와 `1/3` shadow 분포를 같이 보고 정한다

### 7.4. 초기 예시값

아래 수치는 확정값이 아니라 PoC 출발점 예시다.  
2026-07-09 현재 로컬 artifact 예시는 `T_warn=0.2195`, `T_danger=0.3195`이며,
이는 proxy dataset 기준 결과일 뿐 rope 실데이터 기준으로 다시 보정해야 한다.

| 파라미터 | 초기값 예시 | 비고 |
|---|---:|---|
| `T_warn` | `0.35` | 정상 상위 tail 위로 설정 |
| `T_danger` | `0.70` | high recall 목표 기반 |
| `T_warn_release` | `0.25` | 경고 해제용 하향 hysteresis |
| `T_danger_release` | `0.55` | 위험 해제용 하향 hysteresis |
| `highConfidenceDanger` | `0.85` | 즉시 위험 확정용 상한 |

### 7.5. 시간축 안정화 규칙

한 window의 단발 score로 바로 경보를 내면 현장 경보가 튈 수 있다.  
따라서 아래 규칙을 함께 적용한다.

| 레벨 | 권장 규칙 |
|---|---|
| `CAUTION` 진입 | 최근 `5`개 window 중 `2`개 이상이 `T_warn` 초과 또는 EMA score가 `T_warn` 초과 |
| `DANGER` 진입 | 최근 `5`개 window 중 `3`개 이상이 `T_danger` 초과 또는 연속 `2`개 high score와 EMA 확인 |
| `CAUTION` 해제 | 최근 score가 `T_warn_release` 아래로 일정 횟수 연속 유지 |
| `DANGER` 해제 | 최근 score가 `T_danger_release` 아래로 일정 횟수 연속 유지 |

권장 이유:

- 일시적 노이즈 억제
- 위험 단계의 과잉 on/off 방지
- 대시보드와 SNS 경보의 안정성 향상

---

## 8. `1`과 `3`의 역할 정의

### 8.1. 사용 원칙

`1(Matrix Cracking)`과 `3(Delamination)`은 **핵심 학습 정답이 아니다.**

이 둘의 역할은 다음과 같다.

- `1`: 경고 구간에 잘 포착되는지 확인하는 shadow abnormal
- `3`: 일부가 고위험으로 spill-over 되는지 보는 shadow abnormal

### 8.2. 왜 학습에 넣지 않는가

`1`과 `3`을 학습 positive로 넣으면 다음 문제가 생긴다.

- 모델이 `섬유파단 검출기`가 아니라 `복합재 손상 일반 검출기`로 보인다
- 발표 시 "항공기 손상 분류를 선박 rope 파단으로 확장 해석했다"는 공격을 받기 쉽다

반대로 `1`과 `3`을 학습 negative로 넣으면 다음 문제가 생긴다.

- 비정상 손상을 정상과 같은 쪽으로 학습시키는 셈이 된다
- score 해석이 부자연스러워진다

따라서 가장 논리적인 선택은 **학습 제외 + shadow 평가**다.

---

## 9. 발표용 답변 문안

### 9.1. 20초 버전

> 저희는 이 데이터를 선박 rope 파단 정답 데이터로 본 것이 아니라, 공개된 AE proxy dataset으로 사용했습니다.  
> 핵심 모델도 복합재 전체 손상 분류기가 아니라 `Fiber Breakage`만을 positive로 두는
> 이진 검출기로 설계했고, 나머지 손상 클래스는 warning threshold를 검증하는 참고군으로만 사용했습니다.

### 9.2. 1분 버전

> 이 데이터셋은 항공기 복합재 데이터라서, 저희도 이를 선박 계류삭 실데이터와 동일하다고 주장하지는 않습니다.  
> 대신 공통점은 AE 센서 모달리티와 섬유 기반 재료의 미세 손상 시 나타나는 고주파 탄성파 패턴입니다.  
> 그래서 저희는 원본 4클래스를 그대로 운영 라벨로 쓰지 않고, `Fiber Breakage`만을 핵심 위험 타깃으로 재정의했습니다.  
> 즉 모델 자체는 `정상 vs 섬유파단 유사 신호`만 학습하고, `Matrix Cracking`과 `Delamination`은
> 직접적인 rope 파단 정답이 아니라 shadow abnormal로 분리했습니다.  
> 이후 서비스 단계에서만 모델 score와 연속 관측 규칙을 이용해 `정상/경고/위험` 3단계 경보로 변환합니다.  
> 따라서 저희 주장은 '선박 rope 실환경 완성 모델'이 아니라, 'AE 기반 섬유파단 유사 신호 검출 파이프라인의 PoC'입니다.

### 9.3. 예상 질문과 답변

#### Q1. 왜 항공기 복합재 데이터셋을 썼나요?

> 공개된 rope AE 라벨 데이터가 제한적인 상황에서, AE 기반 손상 신호를 실시간 수집/분류/경보하는
> 파이프라인을 먼저 검증할 proxy dataset이 필요했습니다.  
> 이 데이터셋은 AE feature와 손상 라벨이 명확하게 주어져 있어 PoC용 출발점으로 적합했습니다.

#### Q2. 그러면 이 모델이 항공기 손상 분류기 아닌가요?

> 아니고, 그 점을 피하려고 라벨을 그대로 쓰지 않았습니다.  
> 핵심 모델은 `Fiber Breakage`만 positive로 두는 이진 모델이며, 나머지 클래스는 운영 경보 검증용 shadow set으로만 활용했습니다.

#### Q3. 그런데 서비스는 왜 `정상/경고/위험` 3단계인가요?

> 모델은 breakage risk score 하나를 내고, 서비스는 그 score와 시간축 규칙으로 3단계 경보를 만듭니다.  
> 즉 `경고`는 별도 학습 클래스가 아니라 운영 정책상 만든 중간 위험 구간입니다.

#### Q4. 도메인 갭은 어떻게 처리하나요?

> 현재 단계에서는 PoC임을 명확히 밝히고, 이후 선박 계류삭 실데이터를 수집해 전이학습 또는 재학습하는 것이 필수라고 설명합니다.  
> 이 문서의 정책도 그 전까지의 임시 운영 기준입니다.

---

## 10. 구현 체크리스트

### 10.1. 바로 할 일

1. `0`과 `2`만 남긴 core dataset 생성
2. 런타임 대응 7개 feature baseline 학습셋 생성
3. Logistic Regression baseline
4. CatBoost main model
5. `0 vs 2` OOF score로 `T_danger` 선정
6. `1/3` shadow score로 `T_warn` sanity check
7. EMA + 연속 window 규칙을 서비스 정책에 반영

### 10.2. 나중에 할 일

1. `tensionEstimateKn` 실측 데이터 확보 후 feature 추가
2. raw waveform 저장 경로 구축
3. rope 현장 데이터 기반 재학습 또는 transfer learning
4. oversampling/augmentation은 그 이후 보조 실험으로 검토

### 10.3. 데이터 증강 로드맵

현재 단계에서 우선순위는 증강보다 실데이터 확보다. 다만 이후 증강을 검토할 경우 아래 순서를 따른다.

1. **금지**: 초기부터 tabular row를 무작정 복제하거나 임의 수치로 SMOTE를 과신하지 않는다.
2. **1순위**: rope 실측 raw waveform 확보 후 waveform 레벨 증강
3. **2순위**: 센서 현실 범위 내 feature jitter
4. **3순위**: 실데이터 확보 후 class-conditional augmentation

권장 방식:

- waveform이 생기면 `noise injection`, `gain scaling`, `time shift`, `band-limited marine noise mixing`
- feature 공간에서는 센서 허용 범위 내 작은 jitter만 허용
- `1`과 `3`은 여전히 shadow set으로 유지하고, rope 실데이터가 쌓이기 전까지 핵심 positive에 합치지 않는다

---

## 11. 최종 결론

이 프로젝트의 1차 ML PoC는 아래 문장으로 요약할 수 있다.

> **우리는 항공기 복합재 전체 손상 분류기를 만드는 것이 아니라, 공개 AE proxy dataset에서
> `Fiber Breakage`를 핵심 위험 타깃으로 재정의한 이진 검출기를 만들고, 그 score를 이용해
> 선박 계류삭용 `정상/경고/위험` 운영 경보 정책을 구성한다.**

이 방식이 현재 데이터 제약, 서비스 스키마, 발표 명분을 동시에 가장 잘 만족한다.

---

## 12. 참고 자료

### 내부 문서

- [AE-Sentinel 권장 AWS 전체 아키텍처](../architecture/ae_sentinel_recommended_architecture.md)
- [AE-Sentinel 저장소 구조 및 인터페이스 계약](../contracts/aws-system-repository-and-interface-contract.md)
- [부산대학교 클라우드 AI PBL 프로젝트 기획서](../reports/pnu-cloud-ai-pbl-proposal-parsed.md)

### 외부 자료

- Kaggle Acoustic Emission Dataset  
  https://www.kaggle.com/datasets/ziya07/acoustic-emission-dataset

- Bing Wang et al., *Non-destructive testing and evaluation of composite materials/structures: A state-of-the-art review*  
  https://arxiv.org/abs/2002.12201

- Emmanuel Ramasso et al., *Multisensor acoustic emission data set reflecting supervised untightening of bolts in a jointed vibrating structure (ORION-AE)*  
  https://arxiv.org/abs/2111.06322

- Emmanuel Ramasso et al., *On the Condition Monitoring of Bolted Joints through Acoustic Emission and Deep Transfer Learning*  
  https://arxiv.org/abs/2405.20887

- Anju Rani et al., *Imagery Dataset for Condition Monitoring of Synthetic Fibre Ropes*  
  https://arxiv.org/abs/2309.17058

- Tuomas Jalonen et al., *Real-Time Damage Detection in Fiber Lifting Ropes Using Lightweight Convolutional Neural Networks*  
  https://arxiv.org/abs/2302.11947

- Ahmad B. Hassanat et al., *Stop Oversampling for Class Imbalance Learning: A Critical Review*  
  https://arxiv.org/abs/2202.03579
