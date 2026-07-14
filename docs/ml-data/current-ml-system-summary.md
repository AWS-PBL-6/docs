# 현재 ML 시스템 한눈에 보기

> 기준일: 2026-07-14 · 팀 공유용 빠른 요약

## 10초 요약

- 목표: 계류삭이 끊어지기 전 신호를 `NORMAL / CAUTION / DANGER`로 판정한다.
- 운영 모델: 최근 센서 이벤트를 묶어 보는 **XGBoost 3분류 모델**이 SageMaker에 배포돼 있다.
- 실제 센서: 계류삭 연결 고리의 피에조 센서가 **진동수 `freqHz` 하나**를 보낸다.
- 현재 최종 판정: 피에조 입력은 ML 결과를 참고하되, **진동수 기반 물리 정책이 최종 위험도를 확정**한다.
- 생성형 AI: Bedrock 대응 안내 계층은 별도 배포됐지만, 운영 오케스트레이터 연결은 남아 있다.

## 지금 실제로 쓰는 것

| 구성 | 현재 상태 | 역할 |
|---|---|---|
| SageMaker XGBoost | 운영 배포 완료 | 최근 이벤트 패턴의 위험 확률 계산 |
| 피에조 주파수 정책 | 운영 적용 | 실제 센서의 최종 `riskLevel`, `riskScore` 확정 |
| 시간 흐름 정책 | 운영 적용 | 순간 튐보다 최근 판정 흐름을 우선 |
| Bedrock RAG 대응 안내 | AWS 배포 완료, 앱 연결 대기 | 위험 이후 관리자 행동 지침 생성 |
| Kaggle proxy 모델 | 레거시 | 현재 운영 학습·판정에 사용하지 않음 |
| 로컬 모델·mock | 개발용 fallback | SageMaker가 없는 로컬 환경에서만 사용 |

## 전체 흐름

```text
피에조 센서(freqHz)
  → IoT Core: aesentinel/{sensorId}/piezo
  → Backend: 센서↔계류삭 확인
  → freqHz → loadRatio(T ∝ f²) → AE proxy 특징
  → 최근 최대 12개 이벤트 → 17개 윈도 피처
  → SageMaker XGBoost → NORMAL/CAUTION/DANGER 확률
  → 시간 흐름 정책
  → 피에조 주파수 정책이 최종 위험도 확정
  → DynamoDB 저장 + WebSocket 화면 갱신
  → CAUTION/DANGER이면 Agent 안내 + SNS 이메일/SMS
  → DANGER이면 추가로 스냅백 구역 계산
```

모델 입력은 단일 측정값이 아니라 최근 이벤트의 평균·최댓값·발생 횟수·환경값 등
**17개 요약 피처**다. 이벤트가 3개 미만이면 ML 호출은 준비 단계지만, 피에조 물리 판정은
첫 측정부터 적용된다.

## 피에조 최종 판정 기준

기본 보정값은 slack `8 Hz`, critical `32 Hz`이며 배포 환경변수로 바꿀 수 있다.

| 최종 상태 | loadRatio | 기본값 기준 진동수 |
|---|---:|---:|
| NORMAL | `< 0.33` | 약 `< 19.5 Hz` |
| CAUTION | `0.33 ~ 0.66` | 약 `19.5 ~ 26.4 Hz` |
| DANGER | `>= 0.66` | 약 `>= 26.4 Hz` |

> 현재 하드웨어가 직접 측정하는 값은 진동수뿐이다. 모델용 AE 진폭·히트·SNR은 진동수에서
> 만든 proxy이므로, 모델 오류로 낮은 진동수가 위험으로 바뀌지 않도록 물리 기준을 최종값으로 쓴다.

## 모델 카드

| 항목 | 현재 값 |
|---|---|
| 모델 버전 | `ae-port-window-xgb-v1-20260713` |
| 알고리즘 | XGBoost 3-class + temperature scaling |
| 학습 데이터 | 항만 환경으로 변환한 합성 AE 데이터 36,115행 / 55세션 |
| 입력 윈도 | 최근 12개 이벤트, 최소 3개 |
| 입력 피처 | 진폭·SNR·히트·신호군·환경값 등 17개 |
| 검증 방식 | 세션 단위 GroupKFold 5 |
| 클라우드 검증 | 정확도 86.95%, macro-F1 87.07% |
| 안전 핵심 지표 | 위험 재현율 96.99%, 위험 정밀도 99.93% |

성능 수치는 **합성 데이터에서의 파이프라인 검증 결과**다. 실제 항만·실계류삭 성능을
보장하는 수치로 사용하면 안 된다.

## 데이터 출처 핵심

- 원 물리 근거는 Exeter DMaC 설비에서 폴리에스터 계류삭 3개를 파단까지 시험한 실측 논문이다.
- 현재 36,115행 학습표는 논문 결과에 항만 소음·기상·센서 감쇠를 주입한 **2차 합성 데이터**다.
- 비·바람·크레인 값은 실제 항만에서 수집한 값이 아니라 생성 코드의 확률 시나리오다.
- 운영 센서의 실제 `freqHz`는 시스템 검증에만 사용했고 모델 학습에는 아직 쓰지 않았다.

수집 방법, 생성 단계, 근거와 가정은 [데이터 출처·수집·작성 근거](dataset-provenance-and-collection.md)에 정리했다.

## AWS 배포 정보

| 항목 | 값 |
|---|---|
| 계정 / 리전 | `727847798739` / `ap-northeast-2` |
| Endpoint | `ae-sentinel-port-window` (`InService`) |
| Model Registry | `ae-sentinel-line-eye-models/1` (`Approved`) |
| 서빙 방식 | SageMaker Serverless, 3 GB, 최대 동시성 5 |
| Training Job | `ae-sentinel-line-eye-training-20260713-02` |
| 모델 이미지 | SageMaker scikit-learn 1.2-1 + XGBoost 2.1.4 서빙 |

실제 endpoint smoke test와 IoT → Lambda → SageMaker → WebSocket 종단 흐름까지 확인했다.

## Bedrock 대응 안내 계층

- ML 위험도를 다시 판정하지 않는다.
- 안전 매뉴얼·항만사 규정을 RAG로 검색해 관리자 행동 지침을 만든다.
- `CAUTION/DANGER`에서 SNS·대시보드용 구조화 JSON을 생성한다.
- Bedrock 장애 시 고정 템플릿으로 응답한다.
- AWS 배포와 단독 테스트는 끝났고, Backend 비동기 호출·화면 표시는 통합 대기 상태다.

## 팀이 꼭 기억할 한계와 다음 작업

1. **현장 데이터 수집이 최우선**이다. 지금 데이터는 논문 기반 합성 데이터다.
2. 계류삭별 재질·길이·장력에 맞게 slack/critical Hz를 개별 보정해야 한다.
3. 실측 데이터가 쌓이면 AE proxy가 아닌 실제 센서 특징으로 재학습·재검증한다.
4. 모델 변경 시 학습과 Backend의 17개 피처 이름·순서·윈도 버전을 함께 맞춘다.
5. Bedrock 대응 안내는 위험 판정과 분리해 연결하고, 위험도 자체는 변경하지 못하게 유지한다.

## 상세 자료 위치

- 데이터: [출처·수집·작성 근거](dataset-provenance-and-collection.md) · [데이터셋 카드](port-ae-dataset.md) · [신뢰성 검증](dataset-reliability-report.md)
- 배포 증적: [프로덕션 배포 보고서](../reports/deployment-migration-2026-07-13.md)
- 학습 코드: [ML training-pipeline](https://github.com/AWS-PBL-6/ML/tree/main/training-pipeline)
- SageMaker 자산: [ML infra/ml-account](https://github.com/AWS-PBL-6/ML/tree/main/infra/ml-account)
- 추론 연동: [Backend inference_service.py](https://github.com/AWS-PBL-6/Backend/blob/main/application-api/src/services/inference_service.py)
- 생성형 AI: [Agent-Based Response Layer](https://github.com/AWS-PBL-6/ML/tree/main/agent-response-layer)
