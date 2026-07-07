# AE-Sentinel Docs

프로젝트 공통 문서와 인터페이스 계약을 관리하는 기준 저장소다.

## 역할

- 아키텍처 문서 관리
- 팀 간 인터페이스 계약 관리
- JSON Schema 와 example payload 관리
- 계약 변경 프로세스 정의

## 구조

```text
docs/
packages/contracts/
```

## 핵심 문서

- `docs/ae_sentinel_recommended_architecture.md`
- `docs/aws-system-repository-and-interface-contract.md`
- `docs/repository-split-strategy.md`
- `docs/contracts-change-process.md`

## 운영 원칙

- contracts canonical source of truth 는 이 저장소다.
- 구현 저장소는 이 저장소 기준으로 계약을 반영한다.
- breaking contract change 는 docs PR 이 먼저다.
