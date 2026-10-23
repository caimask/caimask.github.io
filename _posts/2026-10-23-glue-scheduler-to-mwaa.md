---
layout: post
title: Glue 스케줄러를 MWAA로 옮긴 데이터 입수 파이프라인 전환
tags: [Airflow, MWAA, AWS Glue, Terraform, Data Pipeline]
author: caimask
---

여러 원천(MySQL, Elasticsearch, S3 파일, DynamoDB)의 데이터와 서비스 로그를 S3 데이터 레이크에 입수하는 파이프라인을 직접 만들어 운영했습니다. 처음에는 Glue job 하나가 스케줄러로 동작했고, 이를 AWS MWAA로 옮겼습니다. 이 글은 전환 전 구조, 옮긴 이유, MWAA 환경 구성, 스케줄러 로직을 DAG으로 옮긴 방식, 이후 운영하며 더한 것을 정리한 기록입니다.

작성 2026-10-23 · 역할 입수 라이브러리와 Glue 스케줄러 설계·개발, MWAA 환경 구성과 DAG 전환(2022-10), Glue 4.0 전환(2024-08 ~ 09) · Airflow 2.2.2(MWAA) · AWS Glue 2.0 → 4.0 · Terraform
{: .meta}

이 글을 한 페이지로 요약한 사례: [MWAA 도입을 통한 데이터 입수 파이프라인 관리](/portfolio/mwaa-ingestion-pipeline)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **전환 전**: Glue Trigger가 매시 Glue job(스케줄러)을 실행했습니다. 스케줄러는 스레드와 큐로 실행 시각 판단, 동시성 제어, 선행 작업 대기, Spark job 실행을 모두 처리했습니다.
2. **옮긴 이유**: 워크플로 엔진이 기본으로 주는 기능을 직접 만들고 있었고, 대상별 실행 이력과 실패한 대상만 다시 돌리는 방법이 없었습니다.
3. **옮긴 방식**: 설정 테이블과 입수 라이브러리, Spark job은 그대로 두고 실행 계층만 바꿨습니다. 실행 시각 판단은 cron으로, 동시성 큐는 pool로, 선행 작업은 별도 DAG으로, 재처리는 수동 실행 설정으로 옮겼습니다.
4. **이후**: 수집 대상이 늘면서 DAG을 만드는 비용이 커졌고, 이것은 [별도 글](/2026/10/22/airflow-dynamic-dag-parsing-review.html)에서 다뤘습니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [전환 전 구조](#before)
2. [Airflow로 옮긴 이유](#why)
3. [MWAA 환경 구성](#mwaa)
4. [스케줄러 로직을 DAG으로 옮기기](#migration)
5. [운영하며 더한 것](#ops)
6. [판단과 대가](#decisions)
7. [결과](#result)
8. [이후: DAG 파싱 과부하](#next)
9. [참고](#refs)

</div>

## 1. 전환 전 구조 {#before}

### 입수 라이브러리 {#library}

원천마다 수집 코드를 따로 쓰지 않도록, 원천 유형별 수집기를 같은 인터페이스로 묶은 라이브러리를 만들었습니다. 수집기는 설정(JSON) 하나로 만들어지고, 원천을 읽어 DataFrame으로 만든 뒤(`load`) S3에 Parquet로 적재합니다(`write`).

<div class="table-wrap" markdown="1">

| 기능 | 내용 |
| --- | --- |
| 원천 유형 | MySQL, Elasticsearch, S3 파일, DynamoDB. 유형마다 `load`만 다르게 구현 |
| 등록 절차 | 실행 계획 확인(시간별 쿼리, 적재 스키마, 적재 경로) → 샘플 조회 → 등록. 등록하면 Glue 테이블을 만들고 설정을 MySQL `crawler_config`에 저장 |
| 스키마 변경 감지 | 저장된 스키마와 비교해, 컬럼이 늘면 알림만 보내고 타입이 바뀌면 적재를 중단하고 알림 |
| 원본 보존 | 적재 결과를 원본 경로에도 복사. 타입 변환이 필요하면 원본을 다시 읽어 결과를 갱신 |
| 컬럼 정리 | camelCase → snake_case, 예약어 컬럼 이스케이프, 지정 컬럼 SHA-512 해시, 타입 지정 변환 |
| 적재 후 처리 | 파티션 추가, 조회용 view 갱신, 적재 건수를 통계 테이블에 기록 |
| 수집 그룹 | 같은 원천을 그룹으로 묶고, 그룹마다 최대 리소스(예: MySQL 동시 연결 수)와 선행 작업을 정의 |

</div>

### Glue job으로 만든 스케줄러 {#scheduler}

실행은 Glue만으로 처리했습니다. Glue Trigger가 매시 Python Shell job(스케줄러)을 실행하고, 스케줄러가 이번 시각에 돌아야 할 대상을 골라 Spark job을 실행했습니다.

```
Glue Trigger (매시)
  └▶ 스케줄러 job (Glue Python Shell)
       ├─ 수집 그룹마다 스레드 1개
       │    ├─ 선행 작업이 있으면 Glue job 실행 → 5초마다 상태 조회하며 대기
       │    ├─ 대상마다 실행 시각 판단 (단위·offset·시간대)
       │    └─ 큐(크기 20)에 넣기 전 가용 리소스 계산 → 모자라면 대기
       └─ 작업 스레드 20개: 큐에서 꺼내 Spark job 실행 (Glue)
```

로그 수집도 같은 방식이었습니다. 로그 수집 스케줄러가 전날 로그를 대상으로, 로그마다 실시간 파티션을 만들고 입수할 파일 크기로 Glue worker 수를 계산해 Spark job을 실행했습니다.

## 2. Airflow로 옮긴 이유 {#why}

스케줄러는 동작했지만, 워크플로 엔진이 기본으로 주는 기능을 하나씩 직접 만들고 있었습니다.

<div class="table-wrap" markdown="1">

| 스케줄러가 직접 하던 일 | 한계 |
| --- | --- |
| 매시 깨어나 대상마다 실행 시각인지 판단 | 다음 실행이 언제인지 어디에도 보이지 않음 |
| 그룹별 큐와 메모리 안의 가용 리소스 계산 | 스케줄러 job이 끝나면 상태가 사라짐. 동시성 현황을 볼 수 없음 |
| 선행 작업을 실행하고 상태를 조회하며 대기 | 그룹 스레드 하나가 대기하는 동안 그 그룹 전체가 멈춤 |
| 대상 Spark job 실행 | DB 수집은 결과를 기다렸지만 실패해도 재시도나 알림으로 이어지지 않음. 로그 수집은 실행만 하고 결과를 기다리지 않음. 대상별 이력이 스케줄러 로그 하나에 섞임 |
| 기간·그룹·대상을 인자로 받아 재처리 | 재처리할 때마다 스케줄러 job 전체를 인자를 바꿔 다시 실행 |

</div>

대상이 늘수록 이 코드를 계속 키워야 했습니다. 실행 판단과 재시도, 이력, 동시성 제어는 Airflow에 맡기고, 입수 로직(라이브러리와 Spark job)만 남기기로 했습니다. 직접 Airflow를 운영하는 대신 MWAA를 택한 것은 스케줄러·워커·메타 DB의 운영을 AWS에 넘기기 위해서였습니다.

## 3. MWAA 환경 구성 {#mwaa}

Terraform으로 MWAA 환경과 접근 경로를 정의하고, 개발·운영 환경은 Terraform workspace와 변수 맵으로 나눴습니다.

<div class="table-wrap" markdown="1">

| 항목 | 구성 |
| --- | --- |
| 환경 | Airflow 2.2.2, `mw1.medium`, worker 1 ~ 10(개발) / 1 ~ 20(운영). 스케줄러·DAG 처리·태스크·워커·웹서버 로그를 CloudWatch로 |
| 동시성 | `core.parallelism` 100, `core.dag_concurrency` 20 |
| 파싱 | `min_file_process_interval` 300초, `dag_dir_list_interval` 300초, `dag_file_processor_timeout` 240초 |
| 웹서버 접근 | 웹서버는 `PRIVATE_ONLY`로 두고, 내부 ALB(HTTP → HTTPS 리다이렉트) + Route 53 레코드로 사내망에서만 접근 |
| 설정 전달 | Airflow 설정의 `custom.*` 값(환경, Glue 스크립트 버킷, 실행 role)을 플러그인이 환경 변수로 풀어 줌. DAG 코드와 Glue job이 같은 방식으로 설정을 읽음 |
| 배포 | 플러그인(zip)과 requirements를 S3에 올리고, 원하면 바로 환경에 반영(`update-environment`, 약 20분). 배포 기록은 Slack으로 |
| 사내 패키지 | 입수 라이브러리를 사내 패키지 저장소에 올리고 requirements와 Glue job에서 같은 버전을 받음 |

</div>

> **Terraform으로 풀리지 않은 것: 웹서버 주소**
>
> 내부 ALB의 target으로 MWAA 웹서버의 VPC endpoint IP가 필요했는데, 당시 Terraform AWS provider는 이 주소를 돌려주지 않았습니다. 그래서 `external` data source로 스크립트를 실행해, 웹서버 URL의 CNAME을 따라가 IP 목록을 얻고 target group에 붙였습니다. 웹서버 로그인 토큰으로 접속 URL을 만드는 스크립트도 함께 두어, 다른 서버에서도 MWAA UI에 들어갈 수 있게 했습니다.
{: .callout .info}

## 4. 스케줄러 로직을 DAG으로 옮기기 {#migration}

설정 테이블(`crawler_config`, `crawler_group`, 로그 수집용 `source_registry`)은 그대로 두고, DAG 파일이 이 테이블을 읽어 row마다 DAG을 만들게 했습니다. 대상을 등록하는 절차와 입수 라이브러리, Spark job은 바꾸지 않았습니다.

<div class="table-wrap" markdown="1">

| 스케줄러(Glue job) | Airflow로 옮긴 형태 |
| --- | --- |
| 매시 대상마다 단위·offset·시간대로 실행 여부 판단 | 같은 설정으로 UTC 기준 cron을 계산해 DAG 스케줄로 지정. 시간대 때문에 날짜가 하루 당겨지는 경우(1일 → 전월 말일 `L`)도 처리 |
| 그룹별 큐 + 가용 리소스 계산 | 그룹·원천 유형별 pool. pool 크기 = 그룹의 최대 리소스, 태스크가 차지하는 slot = 그 작업의 executor 수 |
| 그룹 스레드가 선행 작업을 실행하고 대기 | 선행 작업을 별도 DAG으로 분리. 선행 hook 태스크가 그날 실행 기록을 보고 성공이면 넘어가고, 실행 중이면 기다리고, 없으면 실행 |
| 인자로 기간·대상을 넘겨 스케줄러 재실행 | DAG을 수동 실행할 때 설정(`script_args`, `run_job_kwargs`)으로 날짜와 Glue 리소스를 덮어써 대상 하나만 재처리. 여러 DAG·파티션을 한 번에 다시 돌리는 관리용 DAG 추가 |
| 실패해도 다음 대상으로 넘어감 | 태스크 재시도, 성공·실패·건너뜀 분기 태스크, Slack 알림 |

</div>

pool로 옮기면서 동시성 제어가 단순해졌습니다. 예전에는 스케줄러가 메모리 안에서 가용 리소스를 더하고 빼며 기다렸지만, 지금은 "그룹이 쓸 수 있는 리소스"를 pool 크기로, "작업이 쓰는 리소스"를 `pool_slots`로 선언만 하면 Airflow가 대기와 해제를 맡습니다.

선행 작업을 DAG으로 뺀 것은 한 그룹의 여러 DAG이 같은 선행 작업을 기다리기 때문입니다. 대상 DAG마다 선행 작업을 실행하면 같은 작업이 그룹 크기만큼 중복 실행됩니다. 선행 hook 태스크는 실행 기록을 먼저 확인하므로, 그룹 안에서 처음 시작한 DAG만 선행 작업을 실행하고 나머지는 그 결과를 기다립니다. API로 받는 원천도 이 구조로 등록해, 선행 작업 DAG이 API 결과를 받아 S3에 내려 두게 했습니다.

### DAG 하나의 구성 {#dag}

DB 수집과 로그 수집 DAG은 같은 골격을 씁니다.

```
선행 hook (선행 작업 DAG 실행·대기)
  └▶ 준비 (Branch: 실행 인자 결정. 로그 수집은 입수할 파일 크기로 Glue 리소스 계산)
       ├▶ Glue job 실행 (pool로 동시성 제한)
       │     ├▶ 성공 처리 (파일·데이터 크기, Trino ANALYZE, 통계 저장, 감사 로그)
       │     └▶ 실패 처리 (감사 로그)
       └▶ 건너뜀 (입수할 데이터가 없으면 Glue 생략)
                ▼
           후행 hook ▶ 적재 건수 이상 탐지 ▶ 종료 (감사 로그)
```

### Glue 오퍼레이터 {#glue-operator}

Glue job 실행은 플러그인에 둔 Glue 오퍼레이터가 맡습니다. Airflow 2.2.2에 들어 있는 Amazon provider의 Glue 오퍼레이터를 바탕으로 세 가지를 더했습니다.

- **job이 없으면 만들기**: Glue 버전, 네트워크 연결, 기본 인자를 DAG 코드에 두고, job이 없으면 그 설정으로 만듭니다. Glue job 정의를 Terraform이 아니라 DAG 코드에서 관리하게 됐습니다.
- **실행 시점의 값으로 덮어쓰기**: 앞 태스크가 XCom으로 넘긴 인자(날짜, worker 수)와 수동 실행 설정을 실행 직전에 합칩니다. 리소스를 DAG을 만들 때가 아니라 실행할 때 정합니다.
- **AWS 한도에 걸리면 기다리기**: 동시 실행 한도 초과나 API throttling이면 잠시 기다렸다가 다시 시작합니다. 이것은 실패로 세지 않으므로 태스크 재시도 횟수를 쓰지 않습니다.

## 5. 운영하며 더한 것 {#ops}

<div class="table-wrap" markdown="1">

| 시기 | 한 일 |
| --- | --- |
| 2022-11 | 태스크 단위 감사 로그: 성공·실패·건너뜀·종료와 적재 크기·건수, DAG·태스크 실행 정보를 OpenSearch에 기록. 태스크 실행 시각 기준의 월별 인덱스로 나눔 |
| 2022-11 | API 수집을 선행 작업 DAG으로 등록(4장) |
| 2024-08 ~ 09 | Glue 2.0 지원 종료에 따른 4.0 전환. Spark job과 입수 라이브러리, MWAA 플러그인을 Glue 4.0 런타임과 충돌하지 않게 고침(의존성 버전 고정, MWAA의 Python 3.7 호환). Glue job 이름을 새로 정해, 오퍼레이터가 4.0 job을 새로 만들게 하고 기존 2.0 job은 건드리지 않음 |
| 2024-09 | AWS 지원 케이스의 권고를 반영해 운영 환경의 최대 worker 수를 10 → 20으로 늘리고, worker당 동시 태스크 수를 10으로 고정 |

</div>

## 6. 판단과 대가 {#decisions}

<div class="table-wrap" markdown="1">

| 선택지 | 판단 | 이유 |
| --- | --- | --- |
| 스케줄러 job을 고쳐 쓰기 vs 워크플로 엔진 도입 | Airflow 도입 | 실행 판단, 재시도, 이력, 동시성 제어를 계속 직접 만들고 있었음. 입수 로직만 남기고 나머지는 엔진에 맡김 |
| Airflow 직접 운영 vs MWAA | MWAA | 스케줄러·워커·메타 DB 운영을 AWS에 넘김. 대가로 Airflow 버전과 설정 변경 속도(반영에 약 20분)는 MWAA를 따름 |
| 등록 절차를 Airflow 방식으로 바꾸기 vs 설정 테이블 유지 | 설정 테이블 유지 | 대상 등록과 입수 라이브러리, Spark job을 그대로 두고 실행 계층만 바꿔 전환 범위를 줄임 |
| 대상을 태스크로 묶은 DAG 몇 개 vs 대상마다 DAG | 대상마다 DAG | 대상마다 스케줄·선행 작업·재처리 단위가 달라, 실패와 재처리를 대상 하나 단위로 다룸 |
| 동시성 제어를 코드로 유지 vs pool | pool | 그룹이 쓸 수 있는 양과 작업이 쓰는 양을 선언만 하고, 대기와 해제는 Airflow에 맡김 |

</div>

## 7. 결과 {#result}

<div class="table-wrap" markdown="1">

| 항목 | 전환 전 (Glue 스케줄러) | 전환 후 (MWAA) |
| --- | --- | --- |
| 실행 단위 | 매시 스케줄러 job 1회 안에서 전체 처리 | 대상마다 DAG (로그 수집만 약 300개) |
| 실행 시각 | 매시 코드로 판단 | DAG마다 cron, UI에서 다음 실행 확인 |
| 동시성 제어 | 스레드·큐·메모리 안의 리소스 계산 | 그룹·원천 유형별 pool |
| 실패 처리 | 재시도·알림 없음 | 태스크 재시도 + 실패 분기 + 알림 |
| 재처리 | 스케줄러 job 전체를 인자를 바꿔 재실행 | 대상 하나를 수동 실행 |
| 실행 이력 | 스케줄러 로그 하나에 섞임 | 태스크 단위 감사 로그 (OpenSearch) |
| 스케줄러 코드 | Glue job 2개 | 삭제 |

</div>

## 8. 이후: DAG 파싱 과부하 {#next}

대상마다 DAG을 만들면서 "DAG을 만드는 비용"이 새로 생겼습니다. DAG 파일 하나가 설정 row 수만큼 DAG을 만들고, 그 파일을 스케줄러와 워커가 반복해서 실행하기 때문입니다. MWAA 환경에서 스케줄러 파싱 주기는 300초로 늘려 두었지만, 워커는 이 설정과 상관없이 태스크를 시작할 때마다 파일을 다시 실행합니다.

수집 대상이 늘자 이 비용이 커졌고, 설정 조회가 실패하면 DAG 전체가 UI에서 빠지는 장애 격리 문제도 있었습니다. 원인과 개선 순서는 [Airflow 동적 DAG 파싱 과부하 리뷰](/2026/10/22/airflow-dynamic-dag-parsing-review.html)에 정리했습니다. DB 수집 DAG 파일도 같은 구조라 같은 개선을 적용할 수 있습니다. 개선에 쓰는 `get_parsing_context()`는 Airflow 2.4부터 있으므로, 환경의 Airflow 버전 업그레이드가 선행 조건입니다.

* * *

## 9. 참고 {#refs}

- [Amazon MWAA User Guide](https://docs.aws.amazon.com/mwaa/latest/userguide/what-is-mwaa.html)
- [Amazon MWAA: Managing Python dependencies](https://docs.aws.amazon.com/mwaa/latest/userguide/best-practices-dependencies.html)
- [Airflow: Pools](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/pools.html)
- [AWS Glue version support policy](https://docs.aws.amazon.com/glue/latest/dg/glue-version-support-policy.html)
- 연관 문서: [Airflow 동적 DAG 파싱 과부하 리뷰](/2026/10/22/airflow-dynamic-dag-parsing-review.html)
