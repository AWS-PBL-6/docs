> ⚠️ **보관됨(SUPERSEDED) · proxy breakage 모델 구현 결과 (2026-07-09)**
> 이 문서는 **Kaggle 항공기 복합재 AE 데이터셋(AE-Damage-Detection)** 기반의 초기 proxy 접근을 기록한 것으로, 더 이상 현행 방법이 아니다. 현재는 계류삭 AE 파단 실측 논문(Bashir et al. 2017)을 근거로 한 합성 데이터셋과 그 항만 도메인 변환본을 사용한다.
> 현행 문서: [홋줄 AE 데이터셋 평가](../ml-data/rope-ae-dataset-evaluation.md) · [계류삭 연결 고리 AE 데이터셋](../ml-data/port-ae-dataset.md) · [데이터셋 신뢰성 검증 보고서](../ml-data/dataset-reliability-report.md)

---

# AE Proxy Breakage Model 구현 결과 보고서

작성일: 2026-07-09  
대상 프로젝트: AE-Sentinel

관련 문서:

- [AE Proxy Dataset 학습/임계값/발표 대응 보고서](./ae_proxy_dataset_training_policy.md)
- [AE-Sentinel 권장 AWS 전체 아키텍처](../architecture/ae_sentinel_recommended_architecture.md)
- [AE-Sentinel 저장소 구조 및 인터페이스 계약](../contracts/aws-system-repository-and-interface-contract.md)

관련 구현:

- [학습 파이프라인](../../../ml/training-pipeline/proxy_breakage_pipeline.py)
- [로컬 추론 런타임](../../../ml/endpoint-runtime/local_model_runtime.py)
- [로컬 백엔드 연동](../../../backend/application-api/src/clients/sagemaker_client.py)
- [학습 결과 요약 JSON](../../../ml/training-pipeline/artifacts/latest/training_summary.json)
- [학습 결과 요약 리포트](../../../ml/training-pipeline/artifacts/latest/training_report.md)

---

## 1. 목적

본 문서는 현재까지 구현한 `0 vs 2` 중심의 AE proxy breakage 모델이 실제로 어떤 상태인지,
다음 관점에서 정리하기 위한 결과 보고서다.

1. 무엇을 구현했는가
2. 어떤 데이터/정책으로 학습했는가
3. 현재 성능은 어느 수준인가
4. 바로 다음 단계에서 무엇을 해야 하는가

---

## 2. 구현 범위

이번 단계에서 구현한 항목은 다음과 같다.

### 2.1. 학습 파이프라인

`ml/training-pipeline/proxy_breakage_pipeline.py`를 추가했다.

주요 기능:

- Kaggle CSV 로드
- core dataset 구성: `Damage_Type 0` vs `2`
- shadow dataset 구성: `Damage_Type 1`, `3`
- baseline 7개 feature와 engineered feature set 비교
- weighted logistic regression 학습
- core exemplar 저장
- class별 최근접 이웃 기반 similarity score 구성
- leave-one-out distance 기반 OOD 기준치 산출
- hybrid score 기준 5-fold CV 평가
- hybrid weight / OOD penalty 후보 탐색
- `CatBoost` / `LightGBM` / `XGBoost` 비교 훅
- `T_warn`, `T_danger` threshold 산출
- 학습 artifact와 요약 리포트 생성

### 2.2. 로컬 추론 런타임

`ml/endpoint-runtime/local_model_runtime.py`를 추가했다.

주요 기능:

- `model_artifact.json` 로드
- `inference-request.v1` 형식 입력 수신
- `riskScore`, `riskLevel`, `classProbabilities` 반환
- `breakageScore`, `dangerSimilarity`, `normalSimilarity`, `oodScore` 계산 가능
- 학습 범위를 벗어난 입력값에 대해 feature별 `min/max clip` 적용
- OOD가 높으면 단발 `DANGER`를 `CAUTION`으로 낮추는 guardrail 적용

### 2.3. 백엔드 연동

`backend/application-api`에 `LOCAL_ML_MODEL_PATH` 설정 경로를 추가했다.

동작 우선순위:

1. `SAGEMAKER_ENDPOINT`가 있으면 실제 SageMaker 호출
2. 없고 `LOCAL_ML_MODEL_PATH`가 있으면 학습 artifact 사용
3. 둘 다 없으면 기존 deterministic mock 사용

### 2.4. 시간축 정책 연동

`backend/application-api/src/services/inference_service.py`에 시간축 안정화 규칙을 넣었다.

핵심 정책:

- 단발 `score >= T_danger`라도 바로 `DANGER`로 올리지 않음
- 반복 high score 또는 EMA 추세가 확인되면 `CAUTION -> DANGER` 승격
- `OOD`가 높으면 먼저 `CAUTION`으로 보수 처리
- `highConfidenceDanger`를 넘는 경우만 즉시 `DANGER` 허용

### 2.5. 로컬 데모 시뮬레이터

`backend/application-api/scripts/local_demo.py`는 더 이상 임의의 hand-crafted scale을 쓰지 않는다.

- `LOCAL_ML_MODEL_PATH`가 있으면 artifact의 demo profile을 자동 사용
- 없으면 저장소의 `artifacts/latest/model_artifact.json`을 자동 탐색
- AWS 관련 env는 local demo 안에서 비활성화해 항상 로컬 경로로 실행

---

## 3. 실험 설정

### 3.1. 데이터셋

입력 CSV:

- `/Users/mac/Downloads/AE_Damage_Detection_Dataset.csv`

전체 클래스 분포:

| Damage_Type | 의미 | 행 수 |
|---:|---|---:|
| 0 | No Damage | 418 |
| 1 | Matrix Cracking | 288 |
| 2 | Fiber Breakage | 200 |
| 3 | Delamination | 94 |

### 3.2. 학습 대상

이번 v1 모델의 핵심 목적은 `섬유파단 유사 신호 검출`이다.

따라서:

- core negative: `0`
- core positive: `2`
- shadow abnormal: `1`, `3`

### 3.3. 사용 feature

이번 단계에서는 baseline 7개 raw feature와 engineered feature set을 모두 비교했다.

raw 7개:

- `peakAmplitude`
- `energy`
- `durationMs`
- `riseTimeMs`
- `dominantFrequencyHz`
- `temperatureC`
- `humidityPct`

engineered 후보:

- `logEnergy`
- `energyPerDuration`
- `riseRatio`
- `freqXRise`
- `tempXHumidity`

`Signal_Duration_ms`는 단독 상관이 낮았지만, 이번 구현에서는 다음 이유로 raw feature로 포함했다.

- 런타임 스키마 정합성
- 추후 `energy/duration`, `rise/duration` 파생 feature 확장 가능성

### 3.4. 모델 및 평가

- champion core 분류기: dependency-free weighted logistic regression
- runtime fusion:
  - `hybridScore = (0.6 * breakageScore + 0.4 * dangerSimilarity) * (1 - 0.3 * oodScore)`
- 평가: `Stratified 5-Fold CV`
- threshold 목표:
  - `T_warn`: negative score 95 percentile
  - `T_danger`: `Recall >= 0.90` 우선

추가로 이번 단계에서 실제로 선택된 설정은 다음과 같다.

- selected feature set: `engineered`
- selected hybrid params:
  - `weightBreakage=0.6`
  - `weightDangerSimilarity=0.4`
  - `oodPenalty=0.3`

---

## 4. 결과 요약

### 4.1. core CV 결과 (`0 vs 2`, threshold=`0.5`)

| 지표 | 값 |
|---|---:|
| Accuracy | `0.5469` |
| Precision | `0.3347` |
| Recall | `0.4050` |
| F1 | `0.3665` |
| FNR | `0.5950` |
| ROC-AUC | `0.4985` |
| PR-AUC | `0.3464` |

### 4.2. threshold 결과

| 항목 | 값 |
|---|---:|
| `T_warn` | `0.2743` |
| `T_danger` | `0.3743` |
| `T_warn_release` | `0.1743` |
| `T_danger_release` | `0.2743` |
| `highConfidenceDanger` | `0.4743` |
| `OOD caution` | `0.50` |
| `OOD block danger` | `0.75` |

`T_danger` 적용 시 결과:

| 지표 | 값 |
|---|---:|
| Precision | `0.3256` |
| Recall | `0.9850` |
| F1 | `0.4894` |
| FNR | `0.0150` |

### 4.3. shadow set score 분포

| 클래스 | mean score | `>= T_warn` 비율 | `>= T_danger` 비율 |
|---|---:|---:|---:|
| `1` Matrix Cracking | `0.4850` | `1.0000` | `0.9965` |
| `3` Delamination | `0.4890` | `1.0000` | `0.9894` |

### 4.4. hybrid explain 예시

현재 local demo는 artifact 기반 demo profile을 사용한다.

- hand-crafted scale mismatch는 제거됐다
- `normal`/`caution` profile은 현재 proxy dataset 한계 때문에 둘 다 `CAUTION`으로 시작한다
- `danger` profile은 첫 이벤트부터 `DANGER`로 들어가며 snap-back zone과 alert 경로까지 확인된다

즉, 현재 엔진은 단발 score만으로 위험을 확정하기보다,
`OOD + 시간축 확인`을 우선하는 보수형 동작을 유지하되,
artifact 안쪽 분포를 쓰는 local demo로 바뀌었다.

---

## 5. 해석

### 5.1. 구현은 성공했다

다음 관점에서는 현재 작업이 완료되었다고 볼 수 있다.

- `0 vs 2` core 학습 파이프라인 구축
- shadow set 분리 평가 구현
- similarity/OOD 기반 hybrid artifact 구현
- threshold 자동 산출 구현
- artifact 생성 구현
- `application-api`가 local artifact를 읽는 경로 구현
- 시간축 승격 규칙 구현

즉, **정책 문서대로 파이프라인은 실제 코드로 연결되었다.**

### 5.2. 핵심 분류 성능은 아직 좋지 않다

핵심 문제는 분류력이 낮다는 점이다.

- 현재 champion hybrid 기준 `ROC-AUC 0.4985`는 여전히 사실상 랜덤 수준이다.
- `PR-AUC 0.3464`도 충분히 강한 분리 성능으로 보기 어렵다.
- `threshold=0.5` 기준 `Recall 0.405`로 위험 검출도 약하다.

즉, 현재 모델은 "돌아가는 PoC"이지만 "신뢰 가능한 breakage detector"는 아니다.

### 5.3. booster 비교에서는 `XGBoost`와 `CatBoost`가 hybrid baseline보다 약간 낫다

이번 단계에서 실제 booster 비교도 수행했다.

- `CatBoost`: `F1 0.4122`, `ROC-AUC 0.5518`, `PR-AUC 0.3479`
- `LightGBM`: `F1 0.3718`, `ROC-AUC 0.5274`, `PR-AUC 0.3292`
- `XGBoost`: `F1 0.4147`, `ROC-AUC 0.5469`, `PR-AUC 0.3556`

해석은 다음과 같다.

- 현재 pure-Python hybrid baseline보다 `CatBoost`와 `XGBoost`가 조금 낫다
- 하지만 이 역시 strong classifier라고 보기엔 부족하다
- 즉, 알고리즘을 바꿔도 proxy dataset 한계는 그대로 남는다

### 5.4. engineered feature와 튜닝을 넣어도 본질적 한계는 남아 있다

이번 단계에서 baseline vs engineered, hybrid weight 탐색까지 추가했지만,
분류력이 본질적으로 좋아졌다고 보긴 어렵다.

- selected champion은 `engineered`였지만 `ROC-AUC 0.4985` 수준이다
- shadow danger 비율은 약간 내려갔지만 여전히 거의 전부 danger 영역이다

즉, 파이프라인은 더 정교해졌지만 dataset 한계는 그대로다.

### 5.5. hybrid 엔진은 운영 guardrail로는 의미가 있다

분류력 자체는 약하지만, 하이브리드 엔진은 운영상 두 가지 개선을 제공한다.

- `dangerSimilarity`와 `normalSimilarity`로 "어느 쪽 exemplar와 더 닮았는가"를 같이 볼 수 있다.
- `OOD`가 높을 때 단발 `DANGER`를 막고, 시간축 반복에서만 승격시켜 과잉 경보를 줄일 수 있다.

즉, 지금 단계에서 하이브리드는 **정확도 향상 엔진**이라기보다
**설명 가능성과 보수적 운영 제어를 위한 엔진**에 가깝다.

### 5.6. high-recall threshold는 오탐을 크게 늘린다

`Recall >= 0.90`를 맞추기 위해 `T_danger=0.3743`를 쓰더라도,
`Precision 0.3256` 수준에 머문다.

이 말은:

- 위험을 놓치는 건 줄일 수 있지만
- 정상이나 비핵심 손상도 위험으로 많이 끌어올린다는 뜻이다

### 5.7. shadow set이 거의 전부 danger로 들어온다

`Damage_Type 1`과 `3`이 사실상 전부 `T_danger` 이상으로 들어갔다.

이 결과는 두 가지 해석이 가능하다.

1. model이 실제로 `2`만 분리하지 못하고 `1/3`까지 한 묶음으로 본다
2. dataset 자체에서 `2`와 `1/3`의 feature separation이 약하다

둘 다 현재 proxy dataset의 한계를 보여준다.

### 5.8. local demo 스케일 mismatch는 해결했다

현재 local demo는 artifact profile을 사용하므로 과거의 극단적인 OOD mismatch 문제는 제거됐다.

다만 그것이 모델 신뢰성이 높아졌다는 뜻은 아니다.

- 이제 데모는 "proxy 학습 분포 안에서 backend policy가 어떻게 반응하는가"를 보여준다
- 그러나 proxy dataset 자체가 rope 실데이터를 대체하진 못한다

---

## 6. 검증 수행 내역

이번 단계에서 실제 수행한 검증은 다음과 같다.

### 6.1. 문법 검증

- `python3 -m py_compile`로 신규 스크립트 문법 확인

### 6.2. 학습 artifact 생성

- 실제 CSV로 학습/평가 실행
- 생성 위치:
  - `ml/training-pipeline/artifacts/latest/model_artifact.json`
  - `ml/training-pipeline/artifacts/latest/training_summary.json`
  - `ml/training-pipeline/artifacts/latest/training_report.md`

### 6.3. 로컬 런타임 검증

- `local_model_runtime.py`가 `inference-request.v1` 입력을 받아
  `inference-response.v1` 형식으로 반환하는지 확인

### 6.4. 백엔드 연동 검증

- `LOCAL_ML_MODEL_PATH`를 통해 `application-api`가 artifact를 읽는지 확인
- in-memory mock 저장소 경로로 전체 local demo 흐름 실행 확인

### 6.5. 시간축 승격 검증

- 같은 라인에 동일한 danger 패턴을 반복 주입했을 때
  tuned policy 기준으로 첫 이벤트부터 `DANGER`가 가능함을 확인
- `DANGER` 승격 시 snap-back zone 계산과 alert dispatch 경로까지 동작 확인

---

## 7. 현재 결론

현재 결과를 한 문장으로 요약하면 다음과 같다.

> **`0 vs 2` 중심의 hybrid proxy breakage 학습/추론 파이프라인 구현은 완료되었지만, 현재 Kaggle proxy dataset만으로는 `Fiber Breakage`를 안정적으로 분리하는 성능이 충분하지 않다.**

즉:

- 시스템 구현 측면: 진전 있음
- 모델 신뢰성 측면: 아직 부족함

---

## 8. 다음 단계 권장안

### 8.1. 1순위

`CatBoost` 또는 `XGBoost`를 실제 서빙 후보로 올려서
현재 hybrid logistic baseline과 동일 정책 레이어에서 재평가하는 것이 좋다.

이유:

- booster 비교 결과가 현재 baseline보다 조금 낫다
- 특히 `XGBoost`는 `F1 0.4147`, `PR-AUC 0.3556`으로 현재 baseline보다 우세하다

### 8.2. 2순위

모델 자체는 다음 순서로 확장하는 것이 좋다.

1. booster 모델의 shadow spill-over 분석 추가
2. booster 출력에 현재 `OOD + 시간축` 정책을 그대로 얹은 비교
3. rope 실측 feature 분포로 threshold 재보정
4. 필요하면 그 이후에만 증강 검토

### 8.3. 3순위

가장 중요한 것은 rope 실데이터 확보다.

이유:

- 현재 proxy dataset은 항공 복합재 데이터다
- `2`와 `1/3` 분리가 약하다
- 현장 입력 분포와도 차이가 크다

따라서 성능 개선의 핵심은 알고리즘보다 **실제 rope AE 데이터 확보**다.

---

## 9. 데이터 증강에 대한 판단

현재 단계에서는 **증강보다 실데이터 확보가 우선**이다.

### 9.1. 지금 당장 권장하지 않는 것

- 단순 row duplication
- 무작정 SMOTE
- 임의의 feature noise 추가

이유:

- 현재 separation이 약한 상태에서 synthetic tabular sample을 늘리면
  decision boundary를 더 왜곡할 수 있다
- 특히 `1/3`이 이미 거의 danger로 올라오는 구조에서는
  잘못된 minority 보강이 오히려 모델 본질을 더 흐릴 수 있다

### 9.2. 이후 권장 순서

1. rope raw waveform 확보
2. waveform 레벨 증강
3. 제한된 feature jitter
4. 실데이터 기반 class-conditional augmentation

권장 waveform 증강:

- `noise injection`
- `gain scaling`
- `time shift`
- `band-limited marine noise mixing`

권장 feature-space 증강:

- 센서 허용 범위 내 bounded jitter
- 물리적으로 불가능한 조합을 만들지 않는 작은 범위의 perturbation

---

## 10. 최종 평가

이번 결과는 실패가 아니라, **어디까지 구현됐고 어디서부터 데이터 한계가 시작되는지 명확히 드러난 상태**다.

의미 있는 점:

- 모델 목표를 `0 vs 2`로 좁힌 구조가 실제 코드로 정착됨
- 학습/평가/threshold/artifact/backend 연결이 한 번에 이어짐
- `0 vs 2 score + danger/normal similarity + OOD + 시간축 규칙`이 하나의 엔진으로 묶임
- 발표 시 "우리는 어떤 모델을 만들었고 아직 무엇이 부족한가"를 구체적으로 설명할 수 있게 됨

남은 핵심 과제:

- 실데이터 확보
- demo 입력 재정의
- policy layer 안정화
- 이후에만 증강 검토

현재 시점에서 가장 정확한 표현은 다음과 같다.

> **AE-Sentinel의 v1 hybrid proxy breakage 파이프라인은 구현 완료 상태이며, 모델 자체는 아직 연구/보정 단계다.**
