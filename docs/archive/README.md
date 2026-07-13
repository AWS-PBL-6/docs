# 보관 문서 (Archive)

더 이상 현행 방법이 아닌, 과거 접근을 기록한 문서 모음. 히스토리 보존 및
"무엇을 왜 바꿨는가"의 근거로 남긴다.

## Kaggle 항공기 복합재 proxy 데이터셋 시대 (2026-07-09 이전)

초기에는 공개 rope AE 라벨 데이터가 없어 Kaggle 항공기 복합재 AE 데이터셋
(AE-Damage-Detection, 1,000행, 손상 4종)을 대리(proxy)로 사용해 `0(정상)` vs
`2(섬유파단)` 이진 검출기를 만들었다. 파이프라인은 완성됐으나 분류 성능이
사실상 무작위 수준(ROC-AUC ≈ 0.50)이었고, 항공 복합재 → 항만 계류삭의 도메인
갭이 커 폐기했다.

- [proxy 데이터셋 학습·임계값·발표 대응 정책](ae_proxy_dataset_training_policy.md)
- [proxy breakage 모델 구현 결과 보고서](ae_proxy_training_result_report.md)
- [proxy 하이브리드 엔진 연구 일지](ae_proxy_hybrid_research_journal.md)

## 현행 방법으로의 전환

계류삭 AE 파단 실측 논문(Bashir et al. 2017, *Applied Acoustics* 121)을
근거로 한 합성 데이터셋(안전/주의/위험 3단계) → 계류삭 연결 고리 센서 도메인 변환본으로
교체했다. 근거·평가·검증은 현행 문서 참조:

- [홋줄 AE 데이터셋 평가](../ml-data/rope-ae-dataset-evaluation.md)
- [계류삭 연결 고리 AE 데이터셋 카드](../ml-data/port-ae-dataset.md)
- [데이터셋 신뢰성 검증 보고서](../ml-data/dataset-reliability-report.md)

## 대체된 도메인 설계

- [볼라드 중심 센서 모델 리팩토링 계획](bollard-refactor-plan.md) — 2026-07-13 계류삭별 센서 모델로 대체

> 참고: 아직 유지되는 코드 자산(`ml/training-pipeline/proxy_breakage_pipeline.py`,
> `ml/endpoint-runtime/local_model_runtime.py`, `artifacts/latest/*`)은 로컬 추론
> 경로(`LOCAL_ML_MODEL_PATH`)에 물려 있어 즉시 삭제하지 않고, 새 rope-AE 학습
> 파이프라인이 동일 아티팩트 계약으로 대체할 때까지 남겨둔다.
