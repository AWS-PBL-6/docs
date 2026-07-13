> ⚠️ **보관됨(SUPERSEDED) · proxy 하이브리드 엔진 연구 일지 (2026-07-09)**
> 이 문서는 **Kaggle 항공기 복합재 AE 데이터셋(AE-Damage-Detection)** 기반의 초기 proxy 접근을 기록한 것으로, 더 이상 현행 방법이 아니다. 현재는 계류삭 AE 파단 실측 논문(Bashir et al. 2017)을 근거로 한 합성 데이터셋과 그 항만 도메인 변환본을 사용한다.
> 현행 문서: [홋줄 AE 데이터셋 평가](../ml-data/rope-ae-dataset-evaluation.md) · [계류삭 연결 고리 AE 데이터셋](../ml-data/port-ae-dataset.md) · [데이터셋 신뢰성 검증 보고서](../ml-data/dataset-reliability-report.md)

---

# AE Proxy Hybrid 연구 일지

작성일: 2026-07-09  
대상 프로젝트: AE-Sentinel

관련 문서:

- [AE Proxy Dataset 학습/임계값/발표 대응 보고서](./ae_proxy_dataset_training_policy.md)
- [AE Proxy Breakage Model 구현 결과 보고서](./ae_proxy_training_result_report.md)
- [학습 파이프라인](../../../ml/training-pipeline/proxy_breakage_pipeline.py)
- [로컬 추론 런타임](../../../ml/endpoint-runtime/local_model_runtime.py)
- [로컬 데모 스크립트](../../../backend/application-api/scripts/local_demo.py)
- [학습 결과 요약 JSON](../../../ml/training-pipeline/artifacts/latest/training_summary.json)

---

## 1. 문서 목적

이 문서는 `0 vs 2` 중심 proxy breakage 모델에서
**하이브리드 엔진 도입 이후부터 현재까지** 어떤 연구/보정 작업을 했는지를
시간순으로 정리한 연구 일지다.

정리 범위는 다음과 같다.

1. 왜 단순 이진 score에서 하이브리드 엔진으로 확장했는가
2. 어떤 구현을 실제 코드로 넣었는가
3. 어떤 실험을 했고 무엇이 개선되었는가
4. 현재 남아 있는 한계는 무엇인가

---

## 2. 출발점

초기 proxy 모델의 목적은 명확했다.

- 학습 핵심 클래스: `Damage_Type 0` vs `Damage_Type 2`
- 목표: `Fiber Breakage` 유사 신호를 좁게 잡는 breakage detector

하지만 초기 단순 score 구조에는 두 문제가 있었다.

1. **설명력이 약했다.**  
   왜 위험으로 나왔는지 설명하기 어려웠다.

2. **운영 안정성이 약했다.**  
   단발 score만으로 `NORMAL / CAUTION / DANGER`를 나누면
   OOD 입력이나 일시적 스파이크에 취약했다.

이 문제 때문에 모델 출력 하나만 쓰는 구조에서,
**score + similarity + OOD + 시간축 정책**을 함께 쓰는 하이브리드 엔진으로 확장했다.

---

## 3. 연구 진행 일지

### 3.1. 1단계: 하이브리드 엔진 도입

먼저 core classifier는 그대로 유지했다.

- core classifier: weighted logistic regression
- core label policy: `0` vs `2`

그 위에 다음 요소를 추가했다.

- `breakageScore`
- `dangerSimilarity`
- `normalSimilarity`
- `oodScore`
- `hybridScore`

현재 결합식:

```text
hybridScore = (0.6 * breakageScore + 0.4 * dangerSimilarity) * (1 - 0.3 * oodScore)
```

의도는 단순하다.

- logistic score는 전체적인 breakage 방향성을 본다
- similarity는 실제 `0`/`2` exemplar와 얼마나 닮았는지 본다
- OOD는 학습 manifold 밖의 입력에 패널티를 준다

즉, 하이브리드 엔진은 정확도만 높이기 위한 장치라기보다
**설명 가능성과 보수적 운영 제어**를 위한 구조다.

### 3.2. 2단계: exemplar 기반 similarity와 OOD 추가

학습 artifact 안에 core example을 저장하고,
추론 시 최근접 이웃 거리 기반 similarity를 계산하도록 구현했다.

- `coreExamples` 저장
- `dangerSimilarity` / `normalSimilarity` 계산
- leave-one-out KNN distance로 `oodP95`, `oodP99` 추정

이 단계에서 얻은 실무상 이점은 다음과 같다.

- "왜 danger에 가깝게 봤는가"를 exemplar로 설명 가능
- 학습 분포 밖 입력을 `oodScore`로 확인 가능

### 3.3. 3단계: OOD guardrail 추가

하이브리드 엔진만으로도 여전히 단발 위험 판정이 튈 수 있어서,
정적 OOD 정책을 넣었다.

현재 정책:

- `oodCautionThreshold = 0.50`
- `oodBlockDangerThreshold = 0.75`
- `highConfidenceDanger = 0.474258`

동작 원칙:

- OOD가 높고 고신뢰 danger가 아니면 `DANGER`를 막고 `CAUTION`으로 낮춘다
- OOD가 높고 기본 판정이 `NORMAL`이면 `CAUTION`으로 보수 처리한다

이 단계로 "학습 밖 입력이 갑자기 danger로 뜨는 문제"를 줄였다.

### 3.4. 4단계: 시간축 정책 추가

실제 서비스는 window 단위 입력을 계속 받기 때문에,
단발 score만으로 경보를 확정하는 것은 적절하지 않았다.

그래서 backend 정책 레이어에 시간축 규칙을 넣었다.

현재 정책:

- `windowSize = 5`
- `cautionMinWarnHits = 2`
- `dangerMinDangerHits = 2`
- `emaAlpha = 0.4`
- `warnRelease = 0.174258`
- `dangerRelease = 0.274258`

의미:

- 경고는 반복 warn hit 또는 EMA 상승으로 진입
- 위험은 반복 danger hit 또는 고신뢰 score로 진입
- 해제는 진입보다 더 낮은 release threshold로 처리

이 단계에서 모델과 서비스 역할이 분리됐다.

- 모델: breakage-like risk score 생성
- 서비스: 시간축을 반영해 현장형 경보 단계 결정

### 3.5. 5단계: 로컬 데모 스케일 mismatch 수정

초기 `local_demo.py`는 hand-crafted synthetic feature를 썼다.
이 값들은 학습 분포와 스케일이 크게 달라서,
모델 품질보다 **입력 mismatch**가 더 큰 문제였다.

이후 다음으로 수정했다.

- artifact 안에 `demoProfiles` 저장
- local demo가 artifact 기반 demo profile을 자동 사용
- AWS env를 local demo 내부에서 비활성화

현재 local demo의 의미는 다음과 같다.

- 임의 숫자 시연이 아님
- proxy 학습 분포 안쪽에서 backend policy가 어떻게 반응하는지 보여주는 시연

현재 demo profile 결과:

- `normal`: `CAUTION`, score `0.396`
- `caution`: `CAUTION`, score `0.396`
- `danger`: `DANGER`, score `0.595192`

여기서 중요한 해석은,
**정상과 전조를 깔끔히 분리하지 못하는 문제는 여전히 dataset 한계**라는 점이다.

### 3.6. 6단계: engineered feature 비교

baseline 7개 feature만 쓰는 대신,
현장에서 계산 가능한 파생 feature를 추가한 engineered feature set도 비교했다.

baseline raw feature:

- `peakAmplitude`
- `energy`
- `durationMs`
- `riseTimeMs`
- `dominantFrequencyHz`
- `temperatureC`
- `humidityPct`

engineered feature:

- `logEnergy`
- `energyPerDuration`
- `riseRatio`
- `freqXRise`
- `tempXHumidity`

비교 결과 현재 selection 기준에서는 `engineered`가 champion으로 선택됐다.

현재 selected setting:

- feature set: `engineered`
- `weightBreakage = 0.6`
- `weightDangerSimilarity = 0.4`
- `oodPenalty = 0.3`

다만 이 selection은 단순 ROC-AUC 최대화가 아니라,
**shadow danger spill-over를 더 강하게 벌점 주는 운영형 기준**을 따른다.

### 3.7. 7단계: booster 비교 실험

이후 `CatBoost`, `LightGBM`, `XGBoost`를 실제로 설치하고
같은 core dataset 기준으로 비교 실험을 수행했다.

비교 결과:

| 모델 | Precision | Recall | F1 | ROC-AUC | PR-AUC |
|---|---:|---:|---:|---:|---:|
| Hybrid baseline | `0.3347` | `0.4050` | `0.3665` | `0.4985` | `0.3464` |
| CatBoost | `0.3483` | `0.5050` | `0.4122` | `0.5518` | `0.3479` |
| LightGBM | `0.3246` | `0.4350` | `0.3718` | `0.5274` | `0.3292` |
| XGBoost | `0.3650` | `0.4800` | `0.4147` | `0.5469` | `0.3556` |

현재 해석:

- `XGBoost`가 전체 균형이 가장 좋다
- `CatBoost`도 거의 비슷하게 낫다
- `LightGBM`은 상대적으로 덜 좋다

하지만 중요한 점은,
**booster가 hybrid baseline보다 조금 낫다고 해서 문제 자체가 해결된 것은 아니다.**

여전히:

- ROC-AUC가 높지 않다
- shadow set이 거의 전부 danger 영역으로 들어온다

즉, 모델 교체만으로 proxy dataset 한계를 해결할 수는 없다.

---

## 4. 현재 챔피언 설정

현재 artifact 기준 운영 설정은 다음과 같다.

### 4.1. feature / hybrid 설정

- selected feature set: `engineered`
- model feature count: `12`
- `weightBreakage = 0.6`
- `weightDangerSimilarity = 0.4`
- `oodPenalty = 0.3`

### 4.2. threshold / policy 설정

- `T_warn = 0.274258`
- `T_danger = 0.374258`
- `warnRelease = 0.174258`
- `dangerRelease = 0.274258`
- `highConfidenceDanger = 0.474258`
- `oodCautionThreshold = 0.50`
- `oodBlockDangerThreshold = 0.75`
- `windowSize = 5`
- `cautionMinWarnHits = 2`
- `dangerMinDangerHits = 2`
- `emaAlpha = 0.4`

### 4.3. 현재 champion hybrid 성능

core CV @ `threshold=0.5`:

- Accuracy: `0.5469`
- Precision: `0.3347`
- Recall: `0.4050`
- F1: `0.3665`
- ROC-AUC: `0.4985`
- PR-AUC: `0.3464`

danger threshold 기준:

- Precision: `0.3256`
- Recall: `0.9850`
- F1: `0.4894`
- FNR: `0.0150`

---

## 5. 현재까지의 결론

현재 연구 결과를 한 줄로 요약하면 다음과 같다.

> **하이브리드 엔진, OOD, 시간축 정책, 로컬 데모 정합성, booster 비교까지 모두 구현했지만, proxy dataset만으로는 breakage detector의 신뢰도를 충분히 끌어올리기 어렵다.**

조금 더 풀어 쓰면:

- 시스템 구현 관점: 많이 진전됐다
- 운영 정책 관점: 상당히 정교해졌다
- 데이터/모델 신뢰성 관점: 아직 연구 단계다

현재 하이브리드 구조의 가치는 다음에 있다.

- breakage score 하나보다 설명력이 좋다
- OOD 입력에 보수적으로 대응한다
- 단발 스파이크보다 시간축 패턴을 더 중시한다
- local demo가 더 이상 임의 숫자에 의존하지 않는다

반면 현재 명확한 한계도 있다.

- `0`과 `2` 분리가 약하다
- `1`, `3`이 거의 전부 danger로 spill-over 된다
- proxy dataset이 rope 실데이터를 대체하지 못한다

---

## 6. 다음 단계 제안

우선순위는 다음이 맞다.

### 6.1. 1순위

`XGBoost` 또는 `CatBoost`를 실제 서빙 후보로 올려,
현재 backend 정책 레이어와 합쳐 end-to-end 비교한다.

이유:

- 현재 비교 결과상 hybrid baseline보다 낫다
- 특히 `XGBoost`는 `F1`, `PR-AUC`, `Precision` 균형이 가장 좋다

### 6.2. 2순위

booster 모델에서도 shadow spill-over 분석을 더 명시적으로 수행한다.

즉:

- `0 vs 2` core metric
- `1/3` shadow danger ratio
- local demo policy 반응

이 세 개를 함께 봐야 한다.

### 6.3. 3순위

가장 중요하고 본질적인 작업은 **rope 실데이터 확보**다.

현재까지의 모든 실험은 결국 같은 사실을 보여준다.

> **알고리즘을 조금 바꿔도 proxy dataset의 도메인 한계는 남는다.**

따라서 다음 연구에서 가장 가치가 큰 것은
증강보다 먼저 **실제 rope AE 데이터 수집**이다.
