---
layout: post
title: MWAA 도입을 통한 데이터 입수 파이프라인 관리
date: 2026-10-23
tags: [Airflow, MWAA, AWS Glue, Terraform, Data Pipeline]
author: caimask
---

여러 원천(MySQL, Elasticsearch, S3 파일, DynamoDB)의 데이터와 서비스 로그를 S3 데이터 레이크에 입수하는 파이프라인을 직접 만들어 운영했습니다. Glue job 하나가 스케줄러로 동작하던 구조를 AWS MWAA로 옮겨, 설정 row 하나가 Airflow DAG 하나가 되게 했습니다. 이후 대상이 늘면서 생긴 DAG 파싱 과부하도 분석해 개선 순서를 정했습니다.

역할: 입수 라이브러리와 Glue 스케줄러 설계·개발, MWAA 환경 구성(Terraform)과 DAG 전환(2022-10), Glue 4.0 전환(2024-08 ~ 09), DAG 파싱 자기 검토(2026-10)
{: .meta}

기술: AWS MWAA(Airflow 2.2.2) · AWS Glue · Terraform · Spark · MySQL · Trino · OpenSearch
{: .meta}

<div class="summary" markdown="1">

## 한눈에 보기 {#tldr}

- **문제**: 스케줄러가 Glue job 하나 안에서 스레드와 큐로 실행 판단, 동시성 제어, 선행 작업 대기를 모두 직접 처리했습니다. 대상별 실행 이력이 없었고, 실패한 대상만 다시 돌릴 방법도 없었습니다.
- **판단**: 입수 로직은 그대로 두고 실행 계층만 Airflow로 바꿨습니다. 스케줄러가 하던 일은 cron, pool, 선행 작업 DAG, 수동 실행 설정에 하나씩 대응시켰습니다.
- **결과**: 대상마다 DAG(로그 수집만 약 300개)이 생겼고, 태스크 단위 재시도와 감사 로그, 대상 단위 재처리가 가능해졌습니다. 스케줄러 Glue job 2개는 삭제했습니다.

</div>

## 문제 {#problem}

Glue Trigger가 매시 스케줄러 job을 깨우면, 스케줄러가 수집 그룹마다 스레드를 띄워 실행할 대상을 고르고, 메모리 안에서 가용 리소스를 계산하며 Spark job을 실행했습니다. 워크플로 엔진이 기본으로 주는 기능을 직접 만들고 있었던 셈입니다. 실패해도 재시도나 알림으로 이어지지 않았고, 대상 하나를 다시 처리하려면 스케줄러 job 전체를 인자를 바꿔 다시 돌려야 했습니다.

## 핵심 판단 {#decisions}

<div class="table-wrap" markdown="1">

| 선택지 | 판단 | 이유 |
| --- | --- | --- |
| 파이프라인 전체를 다시 만들기 vs 실행 계층만 교체 | 실행 계층만 교체 | 설정 테이블, 입수 라이브러리, Spark job을 그대로 둬서 대상 등록 절차가 바뀌지 않고 전환 범위가 작아짐 |
| 대상을 묶은 DAG 몇 개 vs 대상마다 DAG | 대상마다 DAG | 대상마다 스케줄, 선행 작업, 재처리 단위가 달라 실패와 재처리를 대상 하나 단위로 다룸 |
| 동시성·선행 작업을 코드로 유지 vs Airflow 기능 | pool과 선행 작업 DAG | 그룹이 쓸 수 있는 리소스와 작업이 쓰는 리소스를 선언만 하면 됨. 선행 작업은 실행 기록을 확인해 그룹 안에서 한 번만 실행 |

</div>

Airflow를 직접 운영하지 않고 MWAA를 택해 스케줄러·워커·메타 DB 운영을 AWS에 넘겼습니다. 환경은 Terraform으로 정의하고, 웹서버는 비공개로 두고 내부 ALB를 거쳐서만 접근하게 했습니다.

## 결과 {#result}

<div class="table-wrap" markdown="1">

| 항목 | 전환 전 (Glue 스케줄러) | 전환 후 (MWAA) | 평가 |
| --- | --- | --- | --- |
| 실행 단위 | 스케줄러 job 1회 안에서 전체 처리 | 대상마다 DAG | <span class="pill ok">개선</span> |
| 동시성 제어 | 스레드·큐·메모리 안의 리소스 계산 | 그룹·원천 유형별 pool | <span class="pill ok">개선</span> |
| 실패 처리 | 재시도·알림 없음 | 태스크 재시도 + 알림 | <span class="pill ok">개선</span> |
| 재처리 | 스케줄러 job 전체 재실행 | 대상 하나만 수동 실행 | <span class="pill ok">개선</span> |
| 실행 이력 | 스케줄러 로그 하나에 섞임 | 태스크 단위 감사 로그 | <span class="pill ok">개선</span> |
| 플러그인·의존성 변경 | S3에 올리면 다음 실행부터 | MWAA 환경 갱신에 약 20분 | <span class="pill warn">감수</span> Airflow 운영을 AWS에 맡긴 대가 |

</div>

## 하위 이슈: DAG 파싱 과부하 {#parsing}

대상마다 DAG을 만들면서 "DAG을 만드는 비용"이 새로 생겼습니다. DAG 파일 하나가 설정 row 약 300개로 DAG 300개를 만들고, 워커는 태스크를 시작할 때마다 이 파일을 다시 실행합니다. 그래서 파싱 경로에 있는 외부 호출(설정 조회, pool 조회·생성)이 "설정 수 × 파싱 횟수"로 곱해졌습니다. 설정 조회가 실패하면 DAG 300개가 함께 UI에서 빠지는 문제도 찾았습니다.

개선 순서는 **장애 격리 → 파싱 경로의 외부 호출 제거 → 워커에서 DAG 하나만 생성 → 파일 분리**로 정했습니다. 파일을 먼저 나누면 파일마다 드는 고정 비용이 300배가 되므로 순서를 바꾸지 않습니다. 아직 적용하지 않았습니다.

## 다음 과제 {#next}

- **파싱 과부하 개선 적용**: 위 순서대로 적용하고, 파일별 파싱 시간과 태스크 대기 시간을 적용 전후로 비교합니다.
- **Airflow 버전 업그레이드**: 워커에서 DAG 하나만 만드는 데 쓰는 `get_parsing_context()`는 Airflow 2.4부터 있어, 2.2.2 환경을 올리는 것이 선행 조건입니다.

> **배운 점**
>
> 스케줄러를 걷어 낸 효과는 코드를 줄인 것보다 **실행 단위를 대상 하나로 바꾼 것**에서 왔습니다. 반대로 대상마다 DAG을 만들자 DAG 생성 비용이 대상 수만큼 곱해졌습니다. Iceberg 커넥터의 커밋 경합과 같은 형태라, 코드를 볼 때는 **언제, 몇 번 실행되느냐**를 먼저 봅니다. ([CDC 이벤트를 Iceberg에 적재하는 파이프라인](/portfolio/cdc-iceberg-ingestion))
{: .callout .info}

## 상세 문서 {#docs}

<div class="table-wrap" markdown="1">

| 문서 | 내용 |
| --- | --- |
| [Glue 스케줄러를 MWAA로 옮긴 데이터 입수 파이프라인 전환](/2026/10/23/glue-scheduler-to-mwaa.html) | 전환 전 구조, 옮긴 이유, MWAA 환경 구성, 스케줄러 기능을 Airflow로 대응시킨 방식, Glue 오퍼레이터, Glue 4.0 전환 |
| [Airflow 동적 DAG 파싱 과부하 리뷰](/2026/10/22/airflow-dynamic-dag-parsing-review.html) | 파싱 시점과 비용 계산, 발견한 결함, 개선안 코드, 파일 하나와 파일 300개의 비용 모델, 적용 순서 |

</div>
