---
layout: post
title: Apache Kafka
tags: [Kafka, Kinesis, Stream Process]
author: caimask
---

Kafka를 처음 정리했던 2020년 글을, 이후 CDC 이벤트를 Kafka Connect로 Iceberg에 적재하면서 겪은 일을 바탕으로 다시 썼습니다. 개념 설명은 바로잡고, 실제로 문제가 됐던 지점(파티션과 키, offset 커밋 시점, 전달 보장)을 보강했습니다.

원문 2020-08-20 · 보강 2026-10-21 · 연관: [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html) · [토스 Iceberg CDC 사례 비교](/2026/10/20/toss-iceberg-cdc-compare.html)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **Kafka는 메시지 큐라기보다 분산 커밋 로그입니다.** 메시지를 읽어도 지워지지 않고, 보존 기간 동안 offset으로 다시 읽을 수 있습니다. 재처리와 여러 소비자의 독립적인 소비가 이 성질에서 나옵니다.
2. **순서는 파티션 안에서만 보장됩니다.** 같은 키는 같은 파티션으로 가므로, "무엇의 순서를 지켜야 하는가"가 곧 메시지 키 설계입니다. CDC에서는 키가 테이블의 primary key와 같아야 합니다.
3. **전달 보장은 offset을 언제 커밋하느냐로 정해집니다.** 처리 후 커밋하면 at-least-once이고, 하류 저장소가 멱등(upsert)이거나 트랜잭션을 지원해야 결과가 정확히 한 번이 됩니다.
4. **ZooKeeper는 더 이상 필요 없습니다.** Kafka 4.0(2025)부터 KRaft만 지원하며, 메타데이터와 컨트롤러 선출을 Kafka가 직접 합니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [Kafka가 해결하는 문제](#why)
2. [핵심 개념](#concepts)
3. [파티션, 키, 순서](#ordering)
4. [offset 커밋과 전달 보장](#delivery)
5. [ZooKeeper에서 KRaft로](#kraft)
6. [Kafka API](#api)
7. [Kafka와 Kinesis](#kinesis)
8. [이후 작업에서 다시 만난 Kafka](#practice)
9. [다시 읽으며 고친 점](#fixes)
10. [참고](#refs)

</div>

## 1. Kafka가 해결하는 문제 {#why}

서비스가 늘어나면 시스템끼리 데이터를 주고받는 연결이 N×M으로 늘어납니다. Kafka는 그 사이에 **모든 이벤트가 한 번 기록되는 공용 로그**를 두어, 생산자와 소비자가 서로를 모르게 만듭니다.

```
[주문 서비스]──┐                 ┌──[검색 색인]
[결제 서비스]──┼──▶ Kafka 토픽 ──┼──[데이터 레이크 적재]
[회원 서비스]──┘   (보존 7일)    └──[실시간 집계]
```

전통적인 메시지 큐와 가장 다른 점은 **소비해도 메시지가 사라지지 않는다**는 것입니다.

<div class="table-wrap" markdown="1">

|   | 전통적인 메시지 큐 (RabbitMQ 등) | Kafka |
| --- | --- | --- |
| 소비 후 메시지 | 확인(ack)하면 삭제 | 보존 기간(`retention.ms`)이나 크기 한도까지 유지 |
| 여러 소비자 | 메시지를 나눠 가짐 | consumer group마다 처음부터 독립적으로 읽음 |
| 재처리 | 어려움 | offset을 되돌려 다시 읽음 |
| 순서 | 큐 단위 | 파티션 단위 |
| 소비 위치 관리 | 브로커가 메시지별로 추적 | 소비자가 offset 하나만 기록 |

</div>

## 2. 핵심 개념 {#concepts}

![Kafka Components]({{ "/assets/img/kafka_components.png"}})

<div class="table-wrap" markdown="1">

| 개념 | 설명 |
| --- | --- |
| record (message) | 키, 값, 타임스탬프, 헤더로 이루어진 이벤트 하나 |
| topic | 같은 종류의 record를 모으는 이름 |
| partition | topic을 나눈 단위. 각 파티션은 **append-only 로그**이며 병렬 처리의 단위 |
| offset | 파티션 안에서 record의 순번. 파티션마다 0부터 증가 |
| broker | Kafka 서버 한 대. 파티션을 나눠 저장 |
| replica | 파티션의 복제본. leader 하나가 읽기·쓰기를 받고 follower가 복제 |
| ISR | leader를 따라잡고 있는 replica 집합. `acks=all`은 ISR 전체에 기록되면 성공 |
| producer | record를 쓰는 쪽. 키로 파티션을 고름 |
| consumer group | 같은 group의 consumer들이 파티션을 나눠 맡음. 한 파티션은 group 안에서 한 consumer만 읽음 |

</div>

consumer group의 병렬도는 파티션 수를 넘을 수 없습니다. 파티션이 6개인데 consumer가 8개면 2개는 놉니다. 반대로 처리량을 늘리려면 파티션을 늘려야 하는데, 파티션 수를 바꾸면 키와 파티션의 대응이 바뀌어 순서 보장이 깨집니다. 그래서 파티션 수는 처음에 여유 있게 정합니다.

## 3. 파티션, 키, 순서 {#ordering}

producer는 기본적으로 `hash(key) % 파티션 수`로 파티션을 고릅니다. 키가 없으면 파티션에 고르게 흩뿌립니다.

```
key=user-17 ──hash──▶ partition 2: [v1][v2][v3]   ← user-17의 변경은 항상 이 순서
key=user-42 ──hash──▶ partition 0: [v1][v2]
key=없음    ──────▶  partition 0, 1, 2 에 분산    ← 순서 보장 없음
```

- 순서가 필요한 단위가 "같은 회원", "같은 주문"이라면 그것을 키로 씁니다.
- 순서가 보장되는 범위는 파티션 하나와, 그 파티션을 읽는 consumer 하나입니다. consumer가 내부에서 여러 스레드로 나눠 처리하면 그 순간 순서가 깨집니다.
- 쓰기 재시도 때문에 순서가 뒤바뀌지 않으려면 idempotent producer(`enable.idempotence=true`, Kafka 3.0부터 기본값)를 씁니다.

> **CDC에서 키가 틀리면 생기는 일**
>
> CDC 이벤트의 키가 테이블의 primary key와 다르면, 같은 행의 변경이 서로 다른 파티션과 서로 다른 sink task로 갑니다. 토스증권의 Iceberg 적재 사례에서는 이 때문에 같은 커밋 안의 upsert가 이전 행을 지우지 못해 중복 행이 생겼습니다. 해결책은 Kafka 메시지 키를 테이블 id로 맞추는 것이었습니다. ([토스 Iceberg CDC 사례 비교](/2026/10/20/toss-iceberg-cdc-compare.html#issues))
{: .callout .warn}

## 4. offset 커밋과 전달 보장 {#delivery}

consumer는 "어디까지 읽었는가"를 `__consumer_offsets` 토픽에 커밋합니다. 처리와 커밋의 순서가 전달 보장을 정합니다.

<div class="table-wrap" markdown="1">

| 보장 | 순서 | 장애가 나면 | 언제 쓰나 |
| --- | --- | --- | --- |
| at-most-once | 커밋 → 처리 | 처리 못 한 record가 **유실** | 일부 유실이 괜찮은 지표 |
| at-least-once | 처리 → 커밋 | 마지막 커밋 이후 record를 **다시 처리** | 대부분의 파이프라인. 하류가 멱등이면 충분 |
| exactly-once | 처리 결과와 offset을 **한 트랜잭션으로** | 둘 다 반영되거나 둘 다 안 됨 | Kafka→Kafka 변환(Streams), 트랜잭션을 지원하는 sink |

</div>

exactly-once는 Kafka 혼자 보장하는 것이 아닙니다. Kafka 트랜잭션은 결과를 다시 Kafka에 쓸 때만 offset과 묶을 수 있습니다. 외부 저장소에 쓴다면 방법은 둘입니다.

1. **멱등 쓰기**: upsert처럼 같은 record를 두 번 써도 결과가 같게 만듭니다. at-least-once로 충분해집니다.
2. **offset을 저장소에 함께 기록**: 데이터와 "어디까지 반영했는가"를 같은 트랜잭션으로 씁니다. Iceberg Kafka Connect Sink는 커밋하는 스냅샷 요약에 offset을 함께 남기는 방식으로 exactly-once를 만듭니다.

> **offset은 저장소에 반영된 만큼만 커밋합니다**
>
> Kafka Connect sink에서는 `preCommit()`이 돌려주는 offset이 커밋됩니다. 하류 저장소의 커밋이 끝나기 전에 offset을 넘기면, 장애가 났을 때 그 구간이 영영 다시 읽히지 않습니다. 타임스탬프가 역전되면 아직 S3에 올리지 않은 record의 offset까지 커밋하던 버퍼 writer의 문제가 이 경우였습니다. ([Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html))
{: .callout .warn}

## 5. ZooKeeper에서 KRaft로 {#kraft}

2020년에 이 글을 쓸 때 Kafka는 클러스터 메타데이터(토픽, 파티션 leader, ISR, ACL)와 컨트롤러 선출을 [ZooKeeper](/2020/09/19/post-zookeeper.html)에 맡겼습니다. 그래서 Kafka를 운영하려면 ZooKeeper 클러스터를 따로 운영해야 했고, 파티션 수가 많아지면 컨트롤러 장애 복구가 느렸습니다.

<div class="table-wrap" markdown="1">

| 시점 | 변화 |
| --- | --- |
| Kafka 2.8 (2021) | KRaft 모드 early access (KIP-500) |
| Kafka 3.3 (2022) | KRaft 운영 환경 사용 가능 |
| Kafka 3.9 (2024) | ZooKeeper 모드를 지원하는 마지막 버전. ZooKeeper → KRaft 마이그레이션 경로 제공 |
| Kafka 4.0 (2025) | ZooKeeper 지원 제거. KRaft만 지원 |

</div>

KRaft에서는 controller 역할의 노드들이 Raft 합의로 메타데이터 로그(`__cluster_metadata`)를 관리합니다. 메타데이터도 결국 Kafka 로그로 다루게 된 것입니다. 지금 새로 구성한다면 ZooKeeper는 고려하지 않습니다.

## 6. Kafka API {#api}

<div class="table-wrap" markdown="1">

| API | 역할 | 대표 용도 |
| --- | --- | --- |
| Producer | topic에 record를 씀 | 애플리케이션 이벤트 발행 |
| Consumer | topic을 구독하고 읽음 | 직접 만든 소비 애플리케이션 |
| Streams | Kafka→Kafka 변환, 집계, join. exactly-once 지원 | 실시간 집계, 이벤트 보강 |
| Connect | 외부 시스템과의 입출력을 설정만으로 구성. source와 sink 커넥터 | Debezium CDC(source), S3·Iceberg 적재(sink) |
| Admin | topic, 파티션, ACL, consumer group 관리 | 운영 자동화 |

</div>

Connect는 직접 Producer/Consumer 코드를 짜지 않고 커넥터 설정으로 파이프라인을 만드는 프레임워크입니다. offset 관리, task 분산, 장애 시 재배치를 Connect가 맡습니다. 대신 커넥터가 그 프레임워크 위에서 어떻게 동작하는지(task 수, 커밋 시점)를 모르면 문제가 생깁니다. 8장에서 다룹니다.

## 7. Kafka와 Kinesis {#kinesis}

![Kafka VS Kinesis]({{ "/assets/img/kafka_vs_kinesis.png"}})

<div class="table-wrap" markdown="1">

|   | Apache Kafka (자체 운영 또는 MSK) | Amazon Kinesis Data Streams |
| --- | --- | --- |
| 병렬 단위 | partition | shard |
| 보존 기간 | 설정하기 나름 (무기한 가능) | 기본 24시간, 최대 365일 |
| 처리량 확장 | 파티션 추가, 브로커 추가 | shard 분할 또는 on-demand 모드 |
| 운영 부담 | 브로커, 디스크, 업그레이드 관리 (MSK는 일부 대행) | 서버 관리 없음 |
| 생태계 | Connect, Streams, Schema Registry 등 | AWS 서비스(Lambda, Firehose)와 통합 |
| 이식성 | 클라우드 중립 | AWS 종속 |

</div>

출처 / 참조 : [링크](https://devidea.tistory.com/68)

선택 기준은 대체로 생태계입니다. Debezium 같은 Kafka Connect 커넥터를 쓰려면 Kafka가 필요하고, AWS 서비스만으로 단순하게 구성하려면 Kinesis가 편합니다.

## 8. 이후 작업에서 다시 만난 Kafka {#practice}

MongoDB CDC 이벤트를 Kafka Connect sink로 Iceberg에 적재하는 커넥터를 검토하면서, 이 글에서 정리한 개념이 그대로 문제의 원인이 됐습니다.

<div class="table-wrap" markdown="1">

| Kafka 개념 | 실제로 생긴 일 | 정리한 문서 |
| --- | --- | --- |
| 파티션이 task에 나뉨 (`tasks.max`) | 한 토픽의 파티션이 task 3개에 나뉘어, 같은 Iceberg 테이블에 task들이 동시에 커밋하며 경합 | [커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#code) |
| 메시지 키 = 순서와 소속을 정함 | 키가 테이블 id와 다르면 같은 행이 다른 task로 가서 upsert 중복 | [토스 사례 비교](/2026/10/20/toss-iceberg-cdc-compare.html#issues) |
| offset 커밋 시점 = 전달 보장 | 타임스탬프가 역전되면 아직 S3에 올리지 않은 record의 offset까지 커밋되어, 장애 시 누락 가능 | [Buffer Writer 리뷰](/2026/10/15/buffer-writer-batch-review.html) |
| record 타임스탬프 | 타임스탬프 기준으로 파일을 나누자 지연 도착 이벤트가 이미 처리된 윈도우에 쓰임 | [Buffer Writer 리뷰](/2026/10/15/buffer-writer-batch-review.html) |

</div>

> **배운 점**
>
> Kafka의 처리량은 파티션 수로 늘리지만, 하류 저장소의 커밋은 그만큼 늘리면 안 됩니다. 병렬로 읽고 쓰되, 저장소에 반영하는 주체는 테이블당 하나로 모으는 구조가 필요합니다.
{: .callout .info}

## 9. 다시 읽으며 고친 점 {#fixes}

- 원문에서 "log: 1개의 메세지"라고 적었는데, log는 메시지 하나가 아니라 **파티션을 이루는 append-only 기록 전체**입니다. 메시지 하나는 record입니다.
- "consumer group이 offset을 공유하여 중복으로 가져가지 않는다"는 반만 맞습니다. 중복을 막는 것은 한 파티션을 group 안의 consumer 하나만 맡는다는 규칙이고, 장애나 리밸런스가 나면 마지막 커밋 이후 구간은 다시 읽힙니다.
- ZooKeeper가 필수라는 설명은 Kafka 4.0부터 맞지 않습니다.

* * *

## 10. 참고 {#refs}

- [Apache Kafka Introduction](https://kafka.apache.org/intro)
- [Apache Kafka Documentation: Design](https://kafka.apache.org/documentation/#design)
- [KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum)
- [위키백과: 아파치 카프카](https://ko.wikipedia.org/wiki/%EC%95%84%ED%8C%8C%EC%B9%98_%EC%B9%B4%ED%94%84%EC%B9%B4)
- [Amazon Kinesis Data Streams 개발자 안내서](https://docs.aws.amazon.com/ko_kr/streams/latest/dev/introduction.html)
