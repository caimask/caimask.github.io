---
layout: post
title: Apache ZooKeeper
tags: [Zookeeper, Distributed System]
author: caimask
---

"분산 처리 시스템은 왜 다루기 어려울까?"라는 질문만 남기고 멈췄던 2020년 ZooKeeper 글을 이어서 썼습니다. 그 질문에 답하고, ZooKeeper가 제공하는 기본 도구(znode, watch, 임시 노드)로 leader 선출과 분산 lock을 만드는 방법, 그리고 Kafka가 ZooKeeper를 떠난 이유를 정리했습니다. 마지막에는 lock 기반 조율을 Iceberg의 lock 없는 커밋과 비교했습니다.

원문 2020-09-19 · 보강 2026-10-21 · 연관: [Apache Kafka](/2020/08/20/post-kafka.html#kraft) · [AWS VPC](/2020/08/05/post-aws-vpc.html) · [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#lock)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **분산 시스템이 어려운 이유는 "다른 노드가 죽었는지, 느린지 구분할 수 없다"는 데 있습니다.** 그래서 누가 leader인지, 누가 살아 있는지, 설정이 무엇인지를 모두가 같은 답으로 알게 해 주는 별도의 합의 계층이 필요합니다. ZooKeeper가 그 역할을 합니다.
2. **ZooKeeper는 작은 파일 시스템처럼 생긴 일관된 저장소입니다.** znode, 세션이 끊기면 사라지는 임시(ephemeral) 노드, 순번(sequential) 노드, 변경 알림(watch) 네 가지로 leader 선출, 멤버십, 분산 lock을 만듭니다.
3. **클러스터는 과반수로 동작합니다.** 노드 2f+1대로 f대 장애를 견디며, 그래서 홀수로 구성합니다. 4대는 3대보다 견딜 수 있는 장애 수가 같고 쓰기만 느립니다.
4. **지금은 시스템들이 ZooKeeper를 내장 합의로 대체하는 추세입니다.** Kafka는 4.0에서 KRaft로 완전히 옮겼습니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [분산 시스템은 왜 어려운가](#why)
2. [코디네이션 서비스](#coordination)
3. [데이터 모델: znode와 watch](#model)
4. [앙상블과 과반수](#quorum)
5. [ZooKeeper로 만드는 것들](#recipes)
6. [ZooKeeper를 쓰던 시스템과 지금](#users)
7. [lock 기반 조율과 Iceberg의 OCC](#occ)
8. [참고](#refs)

</div>

## 1. 분산 시스템은 왜 어려운가 {#why}

원문에서 남긴 질문 세 개에 대한 답입니다.

**분산 처리 시스템은 무엇인가.** 여러 대의 서버가 네트워크로 메시지를 주고받으며 하나의 시스템처럼 동작하는 것입니다. 한 대로 감당할 수 없는 처리량, 저장 용량, 가용성을 얻기 위해 씁니다.

**왜 다루기 어려운가.** 한 대일 때는 없던 문제가 생깁니다.

<div class="table-wrap" markdown="1">

| 문제 | 설명 |
| --- | --- |
| 부분 장애 | 일부 노드만 죽거나, 일부 네트워크만 끊김. 나머지는 계속 동작 |
| 죽음과 느림을 구분할 수 없음 | 응답이 없는 노드가 죽은 것인지, 느린 것인지, 네트워크만 끊긴 것인지 알 수 없음 |
| 공유된 시계가 없음 | 노드마다 시계가 달라 "어느 쪽이 먼저인가"를 시각으로 정할 수 없음 |
| split brain | 네트워크가 둘로 갈라지면 양쪽이 각자 자신이 leader라고 판단할 수 있음 |

</div>

**어떻게 구성하는가.** 모든 노드가 이런 판단을 각자 하면 서로 다른 결론을 냅니다. 그래서 "누가 leader인가", "누가 살아 있는가" 같은 판단을 **합의 알고리즘으로 보호되는 작은 저장소 하나에 맡기고**, 나머지 노드는 그 답을 따르게 구성합니다. 그 저장소가 코디네이션 서비스입니다.

## 2. 코디네이션 서비스 {#coordination}

coordination은 "여러 요소를 조화롭게 맞추는 일"이라는 뜻입니다. ZooKeeper는 데이터를 처리하지 않고, 분산된 애플리케이션들이 같은 사실을 보게 만드는 일만 합니다.

- 저장하는 데이터는 작습니다(znode 하나에 기본 최대 1MB). 설정, 상태, 주소 같은 메타데이터만 둡니다.
- 모든 쓰기는 leader를 거쳐 전체에 같은 순서로 적용됩니다(ZAB 프로토콜). 그래서 모든 클라이언트가 같은 순서의 변경을 봅니다.
- 읽기는 클라이언트가 붙은 서버에서 바로 응답하므로 빠르지만, 아주 최신 값이 아닐 수 있습니다. 최신 값이 꼭 필요하면 `sync` 후에 읽습니다.

## 3. 데이터 모델: znode와 watch {#model}

```
/
├── kafka
│   ├── brokers/ids/1     (ephemeral)  ← broker 1이 살아 있는 동안만 존재
│   ├── brokers/ids/2     (ephemeral)
│   └── controller        (ephemeral)  ← 먼저 만든 broker가 controller
└── locks
    └── table-a
        ├── lock-0000000041 (ephemeral, sequential)
        └── lock-0000000042 (ephemeral, sequential)
```

<div class="table-wrap" markdown="1">

| 기능 | 설명 | 쓰임 |
| --- | --- | --- |
| persistent znode | 지울 때까지 남음 | 설정, 정적인 메타데이터 |
| ephemeral znode | 만든 클라이언트의 세션이 끊기면 자동 삭제 | 살아 있음 표시, leader 표시 |
| sequential znode | 이름 뒤에 단조 증가하는 순번이 붙음 | 대기열, 공정한 lock |
| watch | znode가 바뀌거나 지워지면 한 번 알림 | 설정 변경 감지, leader 교체 감지 |
| version | znode마다 버전이 있어 조건부 쓰기 가능 | "내가 읽은 버전일 때만 수정" |

</div>

임시 노드가 ZooKeeper의 핵심입니다. "죽었는지 느린지 모른다"는 문제를 **세션 타임아웃 안에 heartbeat가 없으면 죽은 것으로 본다**는 하나의 규칙으로 바꾸고, 그 판단을 모든 클라이언트가 똑같이 보게 해 줍니다.

## 4. 앙상블과 과반수 {#quorum}

ZooKeeper 서버 여러 대를 묶은 것을 앙상블(ensemble)이라고 합니다. 쓰기는 과반수 서버에 기록되어야 성공합니다.

<div class="table-wrap" markdown="1">

| 서버 수 | 과반수 | 견딜 수 있는 장애 |
| --- | --- | --- |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 6 | 4 | 2 |

</div>

- 4대는 3대와 견디는 장애 수가 같으면서 쓰기마다 확인해야 할 서버만 늘어납니다. 그래서 홀수로 구성합니다.
- 과반수 규칙 덕분에 split brain이 생기지 않습니다. 네트워크가 갈라져도 과반수를 가진 쪽은 하나뿐입니다.
- 서버를 여러 AZ에 나눠 두면 AZ 하나의 장애를 견딥니다. 3대라면 AZ 3곳에 한 대씩 둡니다. ([AWS VPC](/2020/08/05/post-aws-vpc.html#ha))

## 5. ZooKeeper로 만드는 것들 {#recipes}

### leader 선출

1. 후보들이 모두 `/election/leader`를 ephemeral로 만들려고 시도합니다.
2. 하나만 성공합니다. 성공한 쪽이 leader입니다.
3. 나머지는 그 znode에 watch를 겁니다. leader가 죽어 세션이 끊기면 znode가 지워지고, 알림을 받은 후보들이 다시 1번을 합니다.

### 분산 lock

1. `/locks/table-a/lock-`를 ephemeral + sequential로 만듭니다.
2. 자식 목록에서 내 순번이 가장 작으면 lock을 얻은 것입니다.
3. 아니면 **바로 앞 순번** znode에만 watch를 겁니다. 모두가 같은 znode를 지켜보다 한꺼번에 깨어나는 문제(herd effect)를 피하기 위해서입니다.
4. 작업이 끝나면 내 znode를 지웁니다. 도중에 죽어도 세션이 끊기면 자동으로 지워져 lock이 남지 않습니다.

직접 구현하기보다 [Apache Curator](https://curator.apache.org/)의 레시피(`LeaderLatch`, `InterProcessMutex`)를 씁니다.

> **lock이 있어도 안전하지 않은 경우**
>
> lock을 쥔 프로세스가 GC로 오래 멈추면 세션이 만료되어 다른 프로세스가 lock을 가져갑니다. 멈췄던 프로세스는 깨어난 뒤에도 자신이 lock을 쥐고 있다고 믿고 씁니다. 저장소 쪽에서 lock 순번(fencing token)을 확인해 오래된 쓰기를 거부해야 완전해집니다.
{: .callout .warn}

## 6. ZooKeeper를 쓰던 시스템과 지금 {#users}

<div class="table-wrap" markdown="1">

| 시스템 | ZooKeeper로 하던 일 | 지금 |
| --- | --- | --- |
| Kafka | broker 목록, controller 선출, 토픽 메타데이터 | 4.0(2025)부터 KRaft만 지원. ZooKeeper 제거 |
| HDFS NameNode HA | ZKFC가 active NameNode 선출과 자동 failover | 여전히 ZooKeeper 사용 |
| HBase | RegionServer 감시, master 선출, meta 위치 | 여전히 ZooKeeper 사용 |
| Flink HA | JobManager leader 선출, 체크포인트 메타데이터 위치 | ZooKeeper 또는 Kubernetes ConfigMap 기반 HA |
| ClickHouse 복제 | 복제 로그 조율 | ClickHouse Keeper(내장 Raft)로 대체 가능 |

</div>

추세는 분명합니다. 별도 ZooKeeper 클러스터를 운영하는 부담을 없애기 위해 각 시스템이 Raft 기반 합의를 내장하고 있습니다. 그래도 개념은 그대로입니다. 임시 노드 대신 lease, watch 대신 메타데이터 로그 구독으로 같은 일을 합니다.

## 7. lock 기반 조율과 Iceberg의 OCC {#occ}

CDC 이벤트를 Iceberg에 적재하는 커넥터를 검토할 때 "table lock 때문에 느렸다"는 얘기가 있었습니다. 하지만 Iceberg는 ZooKeeper 같은 외부 lock을 잡지 않습니다. 커밋할 때 카탈로그의 metadata 포인터를 **"내가 읽은 버전일 때만"** 바꾸는 조건부 교체를 하고, 실패하면 다시 시도합니다. znode의 version을 조건으로 쓰는 것과 같은 원리입니다.

<div class="table-wrap" markdown="1">

|   | ZooKeeper 분산 lock (비관적) | Iceberg 커밋 (낙관적, OCC) |
| --- | --- | --- |
| 막는 시점 | 작업 전에 lock을 얻음 | 작업은 자유롭게, 마지막 포인터 교체만 조건부 |
| 경쟁에서 진 쪽 | 순서를 기다림 | 작업 일부를 다시 하고 재시도 |
| 프로세스가 죽으면 | 세션 만료까지 lock이 남음 | 남는 것 없음 (고아 파일만) |
| 경합이 잦을 때 | 순서대로 진행 | 재시도가 폭주해 lock보다 느릴 수 있음 |

</div>

writer 여러 개가 같은 테이블에 몇 초마다 커밋하던 커넥터는 OCC가 가장 불리한 상황이었습니다. 해법은 lock을 도입하는 것이 아니라 **테이블당 커밋 주체를 하나로 줄여 경쟁 자체를 없애는 것**이었습니다. ([Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#lock))

> **배운 점**
>
> 조율 방식은 "경쟁이 얼마나 잦은가"로 고릅니다. 경쟁이 잦으면 lock으로 줄을 세우고, 드물면 낙관적으로 시도합니다. 그보다 나은 것은 구조를 바꿔 경쟁을 없애는 것입니다. ZooKeeper의 leader 선출도 결국 "쓰는 주체를 하나로 정하는 것"입니다.
{: .callout .info}

* * *

## 8. 참고 {#refs}

- [ZooKeeper Programmer's Guide](https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html)
- [ZooKeeper Recipes and Solutions](https://zookeeper.apache.org/doc/current/recipes.html)
- [Apache Curator](https://curator.apache.org/)
- [KIP-500: Replace ZooKeeper with a Self-Managed Metadata Quorum](https://cwiki.apache.org/confluence/display/KAFKA/KIP-500%3A+Replace+ZooKeeper+with+a+Self-Managed+Metadata+Quorum)
- Martin Kleppmann, [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) (fencing token)
