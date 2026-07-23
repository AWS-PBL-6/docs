# AWS 비용 최적화 마이그레이션 요약 (2026-07-22~23)

> 상태: 전환·검증·기존 리소스 정리 완료
>
> 계정/리전: `727847798739` / `ap-northeast-2`
>
> 한 줄 성과: **운영 기능을 유지하면서 월 약 `$262.51`의 상시 실행비를 제거했다.**

## 1. 결과 요약

| 비용 원인 | 변경 전 | 변경 후 | 제거한 월 실행비 |
|---|---|---|---:|
| SageMaker Serverless | 3 GB, Provisioned Concurrency `1` | 3 GB 온디맨드, PC 없음 | `$48.62` |
| RAG 벡터 저장소 | OpenSearch Classic, 상시 `1.0 OCU` | Amazon S3 Vectors | `$213.89` |
| **합계** | 유휴 상태에도 과금 | 사용량 중심 과금 | **`$262.51/월`** |

- 연간 환산: **약 `$3,150.10`**
- 원화 환산: **약 36.8만 원/월, 441만 원/년** (`1 USD = 1,400 KRW` 가정)
- 절감 범위: 위 두 항목의 **고정 실행비 100% 제거**
- 남는 비용: 실제 추론, 벡터 저장·질의, Bedrock 임베딩·생성, 데이터 전송 등

따라서 “AWS 비용이 완전히 0원”이라는 뜻은 아니다. **사용하지 않아도 발생하던 비용을
없애고, 실제 사용할 때만 발생하는 비용 구조로 바꾼 것**이다.

## 2. 비용 계산 근거

### OpenSearch Classic

Classic 개발·테스트 모드는 최소 `0.5 Indexing OCU + 0.5 Search OCU`, 즉 `1.0 OCU`를
유지한다. 당시 CloudWatch에서도 두 항목이 각각 `0.5 OCU`로 관찰됐다.

```text
월 비용 = 1.0 OCU × $0.293/OCU-hour × 730시간
        = $213.89
연 비용 = $213.89 × 12 = $2,566.68
```

- 서울 단가: AWS Price List API의 `APN2-IndexingOCU`, `APN2-SearchOCU` 모두
  `$0.293/OCU-hour`
- 기존 Classic 컬렉션: `bedrock-knowledge-base-x6cck6`
  (`q7utlryg5sm3ibh6pinj`)

### SageMaker Provisioned Concurrency

기존 엔드포인트는 3 GB 메모리와 Provisioned Concurrency `1`로 항상 한 개의 실행 환경을
예열했다. 서울 리전 3 GB PC 단가는 초당 `$0.0000185`다.

```text
월 비용 = PC 1 × $0.0000185/초 × 730시간 × 3,600초
        = $48.618 ≈ $48.62
연 비용 = $48.618 × 12 = $583.42
```

- AWS Price List 사용 유형: `APN2-ProvisionedConcurrency:Usage-3GB`
- 변경 후에는 `ProvisionedConcurrency=null`이므로 위 예열 비용은 발생하지 않는다.
- 요청이 들어오면 온디맨드 추론 시간과 처리 데이터 비용은 계속 발생한다.

> 이 금액은 세금·환율·사용량 비용을 제외한 **월간 실행비 추정치**다. Cost Explorer는
> Organizations SCP로 조회가 차단되어, 실제 설정·CloudWatch 측정·AWS 공식 단가를 기준으로
> 계산했다. 삭제 전 누적 사용량과 청구 반영 지연은 이후 청구서에 남을 수 있다.

## 3. 판단과 실행 흐름

```text
비용 원인 확인
→ 기존 서비스는 유지한 채 대체 구성 생성
→ 데이터 동기화
→ Retrieve·RAG·Lambda 종단 검증
→ Lambda와 CloudFormation 참조 전환
→ 기존 Classic·실패한 시험 리소스 삭제
→ OpenSearch 과금 리소스 0개 재확인
```

| 단계 | 판단과 처리 | 안전장치 |
|---|---|---|
| 1. 비용 분리 | SageMaker PC와 Classic 최소 OCU를 상시 과금 원인으로 특정 | 삭제 전에 설정·사용량 확인 |
| 2. SageMaker 전환 | 모델과 엔드포인트는 유지하고 PC만 제거한 새 설정으로 교체 | `InService` 확인 후 종료 |
| 3. NextGen 시험 | 유휴 10분 후 0 OCU가 가능한 NextGen을 우선 검토 | Classic을 남긴 채 별도 KB로 시험 |
| 4. 방향 변경 | Bedrock Retrieve 오류 때문에 NextGen을 운영에 사용하지 않음 | 실패 상태에서 운영 참조·Classic 미삭제 |
| 5. S3 Vectors 이관 | 저빈도 RAG에 맞는 저장·요청 기반 벡터 저장소 구성 | 새 KB에서 8개 문서 동기화 |
| 6. 운영 전환 | Retrieve, RAG 생성, Lambda 호출을 차례로 검증 | Lambda와 CloudFormation을 함께 변경 |
| 7. 정리 | Classic과 NextGen 시험 리소스·관련 정책 삭제 | 삭제 후에도 Lambda 종단 재검증 |

## 4. 시행착오와 해결

| 문제 | 확인된 현상 | 대응과 교훈 |
|---|---|---|
| NextGen 연동 실패 | 해당 시점의 별도 KB 시험에서 문서 8개 동기화 후 Bedrock Retrieve가 `Request Content Checksum Verification Failed` 반환 | 기능 검증 전에는 기존 저장소를 지우지 않았다. NextGen 대신 S3 Vectors로 전환했다. |
| 메타데이터 제한 | 첫 동기화에서 8개 중 3개만 색인, 5개 무시 | S3 Vectors의 사용자 메타데이터 `1 KB` 제한에 맞춰 5개 파일을 `513~653 bytes`로 축소했다. 원본은 S3 백업 경로에 보존했다. |
| 배포 설정 불일치 위험 | Lambda 환경변수만 바꾸면 다음 SAM 배포 때 이전 KB로 돌아갈 수 있음 | Lambda와 CloudFormation 파라미터를 모두 `LWEXZE8ZFE`로 변경했다. |

이 과정에서 적용한 원칙은 **“복제 → 검증 → 참조 전환 → 기존 삭제”**다. 비용 절감을
서두르더라도 검색 실패나 배포 롤백 가능성을 먼저 제거했다.

## 5. 최종 정량 결과

| 검증 항목 | 최종 결과 |
|---|---|
| SageMaker 엔드포인트 | `ae-sentinel-port-window` / `InService` |
| SageMaker 설정 | 3,072 MB, 최대 동시성 5, Provisioned Concurrency 없음 |
| OpenSearch 컬렉션 / 그룹 | `0 / 0` |
| 운영 Knowledge Base | `LWEXZE8ZFE` / `ACTIVE` |
| 운영 Data Source | `2SPLOJLAET` |
| S3 Vectors 인덱스 | 1,024차원 `float32`, cosine, AES256 |
| 이관 문서 | `8개` |
| 색인 결과 | 1차 `3개` + 재처리 `5개`, 실패 `0개` |
| 저장 벡터 | `2,577개` |
| 순수 벡터 데이터 | `10,555,392 bytes` ≈ `10.07 MiB` |
| 승인 문서 Retrieve 재검증 | 성공, 상위 결과 유사도 `0.7686` |
| 운영 Lambda 종단 검증 | HTTP `200`, KB 출처, 인용 `4개`, 신뢰도 `HIGH` |
| CloudFormation | `ae-sentinel-app` / `UPDATE_COMPLETE` |

S3 Vectors의 실제 저장 크기에는 위 10.07 MiB 외에 키와 메타데이터가 더해진다.
S3 Vectors는 고정 OCU가 없고 저장량·PUT·질의량에 따라 과금되므로, **사용량 비용은
0이라고 단정하지 않는다.** 다만 현재 2,577개 규모에서는 기존 월 `$213.89`의 고정
OpenSearch 컴퓨트보다 비용 구조가 현저히 작다.

## 6. 최종 운영 구성

| 구분 | 현재 값 |
|---|---|
| Vector bucket | `ae-sentinel-kb-vectors-727847798739` |
| Vector index | `ae-sentinel-kb-index` |
| Knowledge Base | `ae-sentinel-mooring-kb-s3vectors` (`LWEXZE8ZFE`) |
| Lambda | `ae-sentinel-app-response-guidance` |
| Lambda의 KB 참조 | `LWEXZE8ZFE` |
| 문서 원본 | `s3://ae-sentinel-kb-762616536500/approved/korean/` |
| 메타데이터 원본 백업 | `s3://ae-sentinel-kb-762616536500/migration-backup/2026-07-23/approved/korean/` |

관련 변경 이력:

- Backend: `7ab109c`
- ML: `b6749db`
- 통합 문서 저장소: `44e6b94`

## 7. 포트폴리오용 요약

> 저사용 RAG 워크로드의 상시 과금 구조를 분석해 OpenSearch Serverless Classic을
> Amazon S3 Vectors로 마이그레이션하고, SageMaker Serverless의 불필요한 Provisioned
> Concurrency를 제거했다. 병행 구축과 종단 검증 후에만 기존 리소스를 삭제해 서비스
> 연속성을 유지했으며, 월 약 `$262.51`, 연 약 `$3,150.10`의 고정 실행비를 제거했다.

## 8. 근거 자료

- [Amazon OpenSearch Service 요금](https://aws.amazon.com/opensearch-service/pricing/):
  Classic 최소 OCU, NextGen의 유휴 시 0 OCU 조건
- [Amazon S3 Vectors 요금](https://aws.amazon.com/s3/pricing/#Vectors):
  저장·PUT·질의 기반 과금 구조와 벡터 크기 계산법
- [S3 Vectors와 Bedrock Knowledge Bases 연동](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-vectors-bedrock-kb.html):
  Bedrock 연동 및 사용자 메타데이터 1 KB 제한
- [Amazon S3 Vectors 소개](https://aws.amazon.com/s3/features/vectors/):
  인프라 사전 프로비저닝 없는 벡터 저장소
- [SageMaker AI 요금](https://aws.amazon.com/sagemaker/ai/pricing/):
  온디맨드 추론과 Provisioned Concurrency 과금 방식

AWS 상태와 단가는 2026-07-23에 AWS CLI와 AWS Price List API로 재확인했다.
