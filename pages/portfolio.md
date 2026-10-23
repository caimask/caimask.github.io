---
layout: page
title: Portfolio
permalink: /portfolio/
icon: "fa-folder-open"
---

데이터 파이프라인을 설계·개발하고 운영해 온 기록입니다. 직접 만든 시스템이 어디서 한계에 부딪혔는지 원인을 짚고, 구조를 바꾸거나 개선 순서를 세운 사례를 한 페이지씩 정리했습니다. 각 사례 끝에서 원인 분석과 코드 수준의 검토를 담은 상세 문서로 이어집니다.

기술: Kafka · Kafka Connect · Debezium · Apache Iceberg · Spark · Airflow · AWS(S3, Glue, Glue Catalog)
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

<span class="when">Airflow · AWS Glue</span>

### [설정 기반 Airflow 수집 DAG 생성기](/portfolio/airflow-dynamic-dag)

수집 대상 약 300개를 설정 row만으로 DAG화하는 생성기를 만들어 운영했습니다. 대상 증가에 따른 파싱 과부하를 분석해 개선 순서를 세웠습니다.

**판단**: 장애 격리 → 파싱 경로의 외부 호출 제거 → 파일 분리 순서 (미적용)

</div>

</div>

## 상세 문서 {#docs}

<div class="table-wrap" markdown="1">

| 문서 | 사례 | 내용 |
| --- | --- | --- |
| [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html) | CDC → Iceberg | 커넥터가 느렸던 원인, OCC와 lock, 삭제 방식, 테이블 운영 최적화 |
| [Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html) | CDC → Iceberg | 전환한 구조의 성과와 자기 검토로 찾은 결함, 개선안 |
| [토스 Iceberg CDC 사례 비교](/2026/10/20/toss-iceberg-cdc-compare.html) | CDC → Iceberg | 외부 사례와 비교, 커밋을 묶을 때의 정합성 문제 |
| [Airflow 동적 DAG 파싱 과부하 리뷰](/2026/10/22/airflow-dynamic-dag-parsing-review.html) | Airflow | 파싱 비용 분석, 발견한 결함, 개선안과 비용 모델 |

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
