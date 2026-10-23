---
layout: page
title: Portfolio
permalink: /portfolio/
icon: "fa-folder-open"
---

데이터 파이프라인을 설계·개발하고 운영해 온 기록입니다. 직접 만든 시스템이 어디서 한계에 부딪혔는지 원인을 짚고, 구조를 바꾸거나 개선 순서를 세운 사례를 한 페이지씩 정리했습니다. 각 사례 끝에서 원인 분석과 코드 수준의 검토를 담은 상세 문서로 이어집니다.

기술: Kafka · Kafka Connect · Debezium · Apache Iceberg · Spark · Airflow · Kubernetes · Helm · Terraform · AWS(S3, Glue, Glue Catalog, EKS, MWAA)
{: .meta}

## 사례 {#cases}

<div class="steps" markdown="1">

<div class="step" markdown="1">

<span class="when">Kafka Connect · Iceberg · Spark</span>

### [CDC 이벤트를 Iceberg에 적재하는 파이프라인](/portfolio/cdc-iceberg-ingestion)

MongoDB CDC 이벤트를 Iceberg에 적재하는 Kafka Connect sink를 만들어 운영했습니다. 커밋 경쟁으로 느려지자 원인을 짚어 buffer + Spark 배치 구조로 바꿨습니다.

**결과**: 테이블당 커밋 주체 8 → 1, 커밋 몇 초마다 → 10분에 2~3회, equality delete 0

</div>

<div class="step" markdown="1">

<span class="when">Kafka Connect · Kubernetes · Helm</span>

### [쿠버네티스에서 운영하는 Kafka Connect 클러스터와 재빌드 이미지](/portfolio/kafka-connect-k8s-image)

EKS에 Kafka Connect 클러스터를 Helm chart로 구성하고 공식 이미지를 재빌드해 운영했습니다. 설정으로 풀리지 않는 문제는 오픈소스(Avro Converter, Debezium)를 직접 고쳐 이미지에 넣었습니다.

**결과**: 커넥터 배포와 이미지 빌드 분리, Schema Registry 일시 실패 시 task 중단 → 재시도, `@` 필드 필터링으로 CDC 메시지 축소

</div>

<div class="step" markdown="1">

<span class="when">MWAA · AWS Glue · Terraform</span>

### [MWAA 도입을 통한 데이터 입수 파이프라인 관리](/portfolio/mwaa-ingestion-pipeline)

Glue job 하나가 스케줄러로 동작하던 입수 파이프라인을 MWAA로 옮겼습니다. 환경은 Terraform으로 구성하고, MySQL 설정 row 하나가 DAG 하나가 되게 했습니다. 이후 대상 증가로 생긴 DAG 파싱 과부하를 분석해 개선 순서를 세웠습니다.

**결과**: 대상마다 DAG(로그 수집만 약 300개), 동시성 큐 → pool, 태스크 단위 재시도·감사 로그·대상 단위 재처리, 스케줄러 Glue job 2개 삭제

</div>

</div>

## 상세 문서 {#docs}

<div class="table-wrap" markdown="1">

| 문서 | 사례 | 내용 |
| --- | --- | --- |
| [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html) | CDC → Iceberg | 커넥터가 느렸던 원인, OCC와 lock, 삭제 방식, 테이블 운영 최적화 |
| [Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html) | CDC → Iceberg | 전환한 구조의 성과와 자기 검토로 찾은 결함, 개선안 |
| [토스 Iceberg CDC 사례 비교](/2026/10/20/toss-iceberg-cdc-compare.html) | CDC → Iceberg | 외부 사례와 비교, 커밋을 묶을 때의 정합성 문제 |
| [Airflow 동적 DAG 파싱 과부하 리뷰](/2026/10/22/airflow-dynamic-dag-parsing-review.html) | MWAA 입수 파이프라인 | 파싱 비용 분석, 발견한 결함, 개선안과 비용 모델 |

</div>

## 기초 기술 노트 {#notes}

2020년에 정리한 기술 노트를 2026년에 보강했습니다. 각 글 끝에 이후 작업에서 다시 만난 부분을 더해 위 사례와 연결했습니다.

<div class="table-wrap" markdown="1">

| 글 | 다룬 내용 | 사례와의 연결 |
| --- | --- | --- |
| [Logstash로 MySQL 데이터를 Kafka로 보내기](/2020/08/06/post-logstash.html) | 쿼리 기반 수집의 한계, 로그 기반 CDC와 비교 | 수집을 CDC로 바꾸면 어려움이 하류의 upsert 반영으로 옮겨 감 |
| [Apache Kafka](/2020/08/20/post-kafka.html) | 파티션·키·순서, offset 커밋과 전달 보장, KRaft | 키 설계와 offset 커밋 시점이 적재 정합성을 정함 |
| [Apache Flink](/2020/09/02/post-flink.html) | 상태와 체크포인트, event time과 watermark, 2단계 커밋 | 단일 committer 구조의 기준 |
| [Apache ZooKeeper](/2020/09/19/post-zookeeper.html) | 과반수, leader 선출과 분산 lock | lock 기반 조율과 Iceberg OCC 커밋의 차이 |
| [Column-oriented Database](/2020/08/27/post-column-db.html) | 컬럼 저장의 장점과 대가, Parquet 구조 | 작은 파일, 삭제 파일, 컴팩션 문제의 뿌리 |
| [AWS VPC](/2020/08/05/post-aws-vpc.html) | Multi-AZ 재설계, endpoint, 저장소 이중화 | S3 데이터 레이크의 네트워크 비용 |
| [GitHub.io 구축하기](/2020/07/30/post-github-io.html) | 이 블로그의 구성과 문서 형식 | — |

</div>
