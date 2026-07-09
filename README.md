# AE-Sentinel Docs

이 디렉토리는 `AWS-PBL-6/docs` repo root 와 동일한 구조를 반영한다.

## 구조

```text
docs/
  docs/            # 현행 문서
  docs/archive/    # 보관(과거 접근) 문서
packages/contracts/
```

## 진행 보고서

- [진행 보고서 (2026-07-08~09)](docs/progress-report-2026-07-09.md) — 디지털 트윈·볼라드 모델·데이터셋·모델 파이프라인 전 작업의 "무엇을·왜" 의사결정 보고서

## 아키텍처 · 계약 · 프로세스 (현행)

- [AE-Sentinel 권장 AWS 전체 아키텍처](docs/ae_sentinel_recommended_architecture.md)
- [저장소 구조 및 인터페이스 계약](docs/aws-system-repository-and-interface-contract.md)
- [저장소 분리 전략](docs/repository-split-strategy.md)
- [계약 변경 프로세스](docs/contracts-change-process.md)
- [AWS 애플리케이션 계정 가이드](docs/aws-application-account-guide.md)
- [디자인 시스템](docs/design.md)

## 데이터셋 · ML (현행)

계류삭 AE 파단 실측 논문(Bashir et al. 2017)을 근거로 한 합성 데이터셋 →
항만 볼라드 도메인 변환본을 사용한다.

- [홋줄 AE 데이터셋 평가](docs/rope-ae-dataset-evaluation.md) — 논문 기반 30k 합성 데이터셋의 이용 평가 근거
- [항만 볼라드 AE 데이터셋 카드](docs/port-ae-dataset.md) — 배포 도메인 변환본(35k)의 생성·구성
- [데이터셋 신뢰성 검증 보고서](docs/dataset-reliability-report.md) — 구조·물리·모델 기반 검증

## 도메인 설계 결정 (현행)

- [센서 부착 위치 결정 — 안벽 볼라드 고정형](docs/sensor-placement-decision.md)
- [볼라드 중심 모델 리팩토링 계획·이력](docs/bollard-refactor-plan.md)

## 원본 기획서

- [부산대학교 클라우드 AI PBL 프로젝트 기획서(파싱본)](docs/pnu-cloud-ai-pbl-proposal-parsed.md)
  — 최초 제출 기획서. Kaggle proxy 데이터셋 계획 시점의 기록이며, 데이터셋
  방향은 이후 홋줄 AE 논문 기반으로 갱신되었다(위 "데이터셋 · ML" 참조).

## 보관 (과거 접근)

- [보관 문서 색인](docs/archive/README.md) — Kaggle 항공기 복합재 proxy 데이터셋 시대의 정책·결과·연구일지
