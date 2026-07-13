# 항만 계류삭 연결 고리 AE 데이터셋 (`port_mooring_eye_ae_dataset.csv`)

> 분류: 현행 ML·데이터

> 재생성일: 2026-07-13 · 생성기: `ml/training-pipeline/port_domain_synthesis.py`  
> 원본: `synthetic_rope_damage_classification_30k.csv` (Bashir et al. 2017 실측 기반 합성)

## 무엇인가

수중 실험실 데이터(하이드로폰, dB re 1 μPa)를 **계류삭의 선박 측 연결 고리
(아이·스플라이스 인근)에 부착한 접촉식 피에조 센서 + 실항만 배경 소음** 도메인으로
변환한 학습용 합성 데이터셋이다. 원본의 하중-손상 물리와 4개 신호군은 보존하고,
계류삭 내 거리 감쇠·센서 클램프 전달 손실·검출 임계·환경 교란을 주입한다.

## 변환 체인

1. 진폭 사상: dB re 1 μPa → dB_AE(re 1 μV)
2. 계류삭 전파 손실: 거리 × 주파수 의존 감쇠(1.0~2.8 dB/m)
3. 센서 결합: 계류삭→클램프형 피에조 센서 전달 손실 1.5±0.5 dB(가정)
4. 부동 검출 임계: 노이즈 플로어 + 6 dB
5. 강우·크레인·바람 교란 이벤트 주입
6. 실제 항만용 로프 규격과 기상 공변량 부여

## 생성 결과

| 항목 | 값 |
|---|---:|
| 전체 행 | 36,115 |
| 세션 | 55 |
| 로프 기원 이벤트 | 20,386 |
| 환경 교란 이벤트 | 15,729 |
| 원본 로프 이벤트 미탐 | 32.0% |
| 안전 / 주의 / 위험 | 13,092 / 12,009 / 11,014 |

신호군 생존율:

| 신호군 | 생존율 |
|---|---:|
| High amplitude | 99.94% |
| Medium amplitude | 89.58% |
| Low amplitude | 56.92% |
| Low-to-high frequency | 24.36% |

## 컬럼 규약

- 학습 피처 후보: 진폭·대역·지속시간·히트·SNR + 환경 공변량 + 로프 규격
- `Truth_*`: 생성 진실값으로 학습 입력 금지
- `Truth_Distance_m`: 계류삭 연결 고리 센서와 이벤트 발생 지점 사이 거리
- 라벨: `DamageType` 안전/주의/위험 → `NORMAL/CAUTION/DANGER`

## 재학습 검증

`artifacts/line-eye-latest`를 GroupKFold 5(세션 단위)로 재학습했다.

| 정확도 | macro-F1 | 위험 재현율 | 위험 정밀도 |
|---:|---:|---:|---:|
| 0.8700 | 0.8717 | 0.9701 | 0.9998 |

이 수치는 합성 규칙 아래 파이프라인 학습 가능성을 보여줄 뿐, 실환경 성능을 보장하지 않는다.
실제 연결 고리 피에조 데이터가 수집되면 `sensor_coupling_loss`, 감쇠, 주파수 기준선을
계류삭별로 재보정해야 한다.

## 재현

```bash
PYTHONPATH=<repo>/.vendor_ml python3 ml/training-pipeline/port_domain_synthesis.py \
  --src ~/Downloads/synthetic_rope_damage_classification_30k.csv \
  --out ~/Downloads/port_mooring_eye_ae_dataset.csv

PYTHONPATH=<repo>/.vendor_ml python3 ml/training-pipeline/port_breakage_pipeline.py \
  --src ~/Downloads/port_mooring_eye_ae_dataset.csv \
  --out ml/training-pipeline/artifacts/line-eye-latest
```
