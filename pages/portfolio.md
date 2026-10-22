---
layout: page
title: Portfolio
permalink: /portfolio/
icon: "fa-folder-open"
---

이 블로그의 글을 하나의 흐름으로 묶었습니다. 2020년에 데이터 파이프라인의 기초 기술을 정리한 글에서 시작해, 2026년에 MongoDB CDC 이벤트를 Kafka에서 Iceberg로 적재하는 구조를 검토하고 개선 방향을 찾은 문서로 이어집니다. 2020년 글은 이후 작업에서 실제로 문제가 됐던 지점을 중심으로 2026-10-21에 보강했습니다.

<div class="summary" markdown="1">

## 한눈에 보기 {#tldr}

- **다룬 문제**: CDC 이벤트를 데이터 레이크(Iceberg)에 빠르고 정확하게 적재하기. 커밋 경합, upsert 정합성, 작은 파일과 삭제 파일, 지연 도착 이벤트의 누락.
- **접근 방식**: 기존 구현을 코드 단위로 읽고 원인을 짚은 뒤, 공식 스펙과 외부 사례(토스증권)와 비교해 개선안과 우선순위를 냅니다.
- **기술**: Kafka, Kafka Connect, Debezium, Apache Iceberg, Spark, Flink, Parquet, AWS(S3, Glue, VPC).

</div>

## CDC → Iceberg 적재 검토 (2026)

<div class="table-wrap" markdown="1">

| 문서 | 다룬 문제 | 결론 |
| --- | --- | --- |
| [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html) | 자체 개발 Kafka Connect sink가 느렸던 원인, "Iceberg도 row lock이 된다"는 정보의 진위 | 원인은 lock이 아니라 OCC 커밋 재시도 폭주. 병렬 writer + 테이블당 단일 committer + 1~5분 커밋으로 전환 |
| [Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html) | 커밋 경합을 피하려고 바꾼 "S3 버퍼 + Spark 배치" 구조 | 방향은 맞음. 윈도우 완료 판단, `MERGE` 원본 선택, offset 커밋에서 조용한 누락 위험 3건과 시간이 갈수록 느려지는 구조를 짚음 |
| [토스 Iceberg CDC 사례 비교](/2026/10/20/toss-iceberg-cdc-compare.html) | 공개된 외부 사례와 우리 구조의 비교, 그 방식으로 옮길 때 고칠 코드 | 같은 문제의 다른 갈래. 커밋을 묶는 순간 생기는 equality delete 정합성 문제와 그 해결 경로 |

</div>

## 파이프라인 오케스트레이션 검토 (2026)

<div class="table-wrap" markdown="1">

| 문서 | 다룬 문제 | 결론 |
| --- | --- | --- |
| [Airflow 동적 DAG 파싱 과부하 리뷰](/2026/10/22/airflow-dynamic-dag-parsing-review.html) | MySQL 설정 row마다 DAG을 만드는 수집 DAG이 설정 증가에 따라 무거워지는 문제 | 원인은 파싱 경로의 외부 호출이 "설정 수 × 파싱 주기 × 태스크 수"로 곱해지는 것. 외부 호출 제거 → 워커 파싱 축소 → 파일 분리 순서로 개선 |

</div>

## 기초 기술 (2020, 2026 보강)

각 글 끝에 "이후 작업에서 다시 만난 부분"을 더해, 위 검토 문서와 연결했습니다.

<div class="table-wrap" markdown="1">

| 글 | 보강한 내용 | 위 검토와의 연결 |
| --- | --- | --- |
| [Logstash로 MySQL 데이터를 Kafka로 보내기](/2020/08/06/post-logstash.html) | 쿼리 기반 수집의 한계(update·delete 미감지, 늦게 커밋된 행 누락), 로그 기반 CDC와 비교 | 수집을 CDC로 바꾸면 어려움이 하류의 upsert 반영으로 옮겨 감 |
| [Apache Kafka](/2020/08/20/post-kafka.html) | 파티션·키·순서, offset 커밋과 전달 보장, KRaft 전환 | 키 설계와 offset 커밋 시점이 적재 정합성을 정함 |
| [Apache Flink](/2020/09/02/post-flink.html) | 상태와 체크포인트, event time과 watermark, 2단계 커밋 | Flink Iceberg sink의 단일 committer 구조가 개선안의 기준 |
| [Apache ZooKeeper](/2020/09/19/post-zookeeper.html) | 분산 시스템이 어려운 이유, 과반수, leader 선출과 분산 lock | lock 기반 조율과 Iceberg OCC 커밋의 차이 |
| [Column-oriented Database](/2020/08/27/post-column-db.html) | 컬럼 저장이 빠른 이유와 대가, Parquet 구조 | 작은 파일, 삭제 파일, 컴팩션 문제의 뿌리 |
| [AWS VPC](/2020/08/05/post-aws-vpc.html) | Multi-AZ 재설계, Security Group과 NACL, endpoint 종류, 저장소 이중화 | S3 데이터 레이크의 네트워크 비용 |
| [GitHub.io 구축하기](/2020/07/30/post-github-io.html) | 저장소 이름 규칙, 검토 문서용 스타일 컴포넌트 | 이 블로그의 문서 형식 |

</div>

## 읽는 순서

처음 읽는다면 [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html)의 요약부터 보고, 낯선 개념이 나올 때 위 기초 기술 글로 내려가는 순서를 권합니다.
