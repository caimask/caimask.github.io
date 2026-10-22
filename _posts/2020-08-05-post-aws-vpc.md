---
layout: post
title: AWS VPC
hide_title: false
excerpt_separator: <!--more-->
tags: [aws,vpc,network]
---

AWS에서 시스템의 네트워크를 구성하는 방법을 정리하고, 이전에 구성했던 VPC를 Multi-AZ 기준으로 다시 설계해 봅니다.

<!--more-->

AWS로 시스템을 구성하며 쓰던 네트워크 요소를 정리한 2020년 글을 보강했습니다. 구성 요소 설명에 Security Group과 Network ACL, VPC endpoint 두 종류의 차이를 더했고, 원문에 "고민 사항"으로 남겨 둔 두 질문(저장소를 AZ별로 나눠야 하는가, Hadoop NameNode 이중화는 어떻게 하는가)에 답했습니다. 마지막으로 S3 위 데이터 레이크를 운영할 때 네트워크 비용이 어디서 생기는지 정리했습니다.

원문 2020-08-05 · 보강 2026-10-21 · 연관: [Apache ZooKeeper](/2020/09/19/post-zookeeper.html) · [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **VPC는 계정 전용 가상 네트워크이고, 실제 배치 단위는 AZ에 묶인 subnet입니다.** public과 private의 차이는 subnet 자체가 아니라 연결된 route table이 Internet Gateway로 가느냐입니다.
2. **이전 구성은 private subnet이 하나뿐이라 Multi-AZ가 아니었습니다.** AZ마다 public과 private subnet을 한 쌍씩 두고, NAT Gateway도 AZ마다 둬야 AZ 하나의 장애를 견딥니다.
3. **저장소의 이중화는 "AZ마다 따로 둔다"가 아니라 "복제와 failover를 누가 맡는가"의 문제입니다.** RDS Multi-AZ, HDFS NameNode HA(ZooKeeper + JournalNode)처럼 저장소마다 정해진 방법이 있습니다.
4. **S3로 가는 트래픽은 Gateway endpoint로 보냅니다.** private subnet의 대량 적재 트래픽이 NAT Gateway를 거치면 GB당 처리 요금이 붙습니다. Gateway endpoint는 무료입니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [VPC 구성 요소](#components)
2. [Security Group과 Network ACL](#security)
3. [VPC endpoint 두 종류](#endpoint)
4. [고가용성을 위한 Multi-AZ 운영](#ha)
5. [이전에 구성했던 방식과 개선안](#before)
6. [고민 사항에 대한 답](#answers)
7. [데이터 레이크 운영에서의 네트워크 비용](#cost)
8. [새롭게 알게 된 것](#learned)
9. [성과와 기대 효과](#outcome)
10. [참고](#refs)

</div>

## 1. VPC 구성 요소 {#components}

Amazon VPC(Virtual Private Cloud)는 AWS 계정 전용으로 만드는 가상 네트워크입니다. 자체 데이터 센터의 네트워크와 비슷하게 IP 대역, subnet, 라우팅, 방화벽을 직접 정합니다.

<div class="table-wrap" markdown="1">

| 요소 | 범위 | 설명 |
| --- | --- | --- |
| Region | 지리적 지역 | AWS 서비스가 위치한 지역. VPC는 Region 하나에 만듦 |
| Availability Zone (AZ) | Region 안 | 전력·네트워크가 물리적으로 분리된 데이터 센터 묶음 |
| VPC | Region | 계정 전용 가상 네트워크. [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) 대역 하나 (예: `10.0.0.0/16`) |
| Subnet | **AZ 하나** | VPC 대역을 나눈 블록. subnet은 AZ를 넘지 못함 |
| Route Table | subnet에 연결 | 목적지별로 트래픽을 어디로 보낼지 정하는 규칙 |
| Internet Gateway | VPC | VPC와 인터넷 사이의 양방향 통로 |
| NAT Gateway | **AZ 하나** (public subnet에 둠) | private subnet에서 인터넷으로 나가는 단방향 통로. 밖에서 안으로는 연결 불가 |
| VPC Endpoint | VPC | 인터넷을 거치지 않고 AWS 서비스에 연결 |
| VPC Peering | VPC 둘 | 두 VPC를 사설 IP로 연결. 대역이 겹치면 안 됨 |

</div>

> **public subnet과 private subnet**
>
> subnet에 public이라는 설정이 따로 있는 것이 아닙니다. route table에 `0.0.0.0/0 → Internet Gateway` 규칙이 있으면 public, `0.0.0.0/0 → NAT Gateway`이거나 인터넷 경로가 없으면 private입니다.
{: .callout .info}

## 2. Security Group과 Network ACL {#security}

<div class="table-wrap" markdown="1">

|   | Security Group | Network ACL |
| --- | --- | --- |
| 적용 대상 | 인스턴스(ENI) | subnet |
| 규칙 | 허용만 | 허용과 거부 |
| 상태 | stateful. 들어온 요청의 응답은 자동 허용 | stateless. 응답 트래픽도 규칙으로 열어야 함 |
| 평가 | 모든 규칙을 함께 평가 | 번호 순서대로 평가해 처음 맞는 규칙 적용 |
| 참조 | 다른 Security Group을 출처로 지정 가능 | IP 대역만 |
| 주 용도 | 서비스 간 접근 제어 (예: "앱 SG에서만 DB 3306 허용") | subnet 단위의 넓은 차단 |

</div>

보통 Security Group으로 세밀하게 제어하고, Network ACL은 기본값(전체 허용)에 특정 대역 차단만 더합니다.

## 3. VPC endpoint 두 종류 {#endpoint}

<div class="table-wrap" markdown="1">

|   | Gateway endpoint | Interface endpoint (PrivateLink) |
| --- | --- | --- |
| 대상 서비스 | S3, DynamoDB | 대부분의 AWS 서비스 (Glue, STS, Kinesis 등), 다른 계정의 서비스 |
| 동작 | route table에 서비스 경로 추가 | subnet에 사설 IP를 가진 ENI 생성 |
| 비용 | <span class="pill ok">무료</span> | <span class="pill warn">유료</span> 시간당 + GB당 |
| VPC 밖에서 접근 | 불가 (peering, VPN 너머에서는 못 씀) | 가능 |

</div>

VPC endpoint를 쓰면 인스턴스에 public IP가 없어도 AWS 서비스와 통신하고, 트래픽이 AWS 네트워크 밖으로 나가지 않습니다.

## 4. 고가용성을 위한 Multi-AZ 운영 {#ha}

고가용성은 서버, 네트워크, 프로그램 같은 시스템이 **오랜 기간 끊김 없이 정상 운영되는 성질**입니다. 장비 장애뿐 아니라 화재, 정전, 지진처럼 데이터 센터 하나를 통째로 잃는 경우까지 대비하려면, 물리적으로 분리된 곳에 시스템을 이중화해야 합니다.

AWS에서 그 분리 단위가 AZ입니다. AWS도 한 AZ의 장애에 대비해 여러 AZ에 시스템을 나눠 배치하도록 권장하고, 단일 AZ 구성으로 생긴 손실은 사용자 책임으로 봅니다. 그래서 한 Region에서 서비스를 구성할 때는 두 개 이상의 AZ에 같은 구성을 둡니다.

```
VPC 10.0.0.0/16
├── AZ-a
│   ├── public  10.0.0.0/24   ── ALB, NAT Gateway-a
│   └── private 10.0.10.0/24  ── 앱, DB primary   ── route: 0.0.0.0/0 → NAT-a
└── AZ-c
    ├── public  10.0.1.0/24   ── ALB, NAT Gateway-c
    └── private 10.0.11.0/24  ── 앱, DB standby   ── route: 0.0.0.0/0 → NAT-c

route table (private, 공통): S3 → Gateway endpoint
```

- private subnet마다 같은 AZ의 NAT Gateway를 쓰게 route table을 AZ별로 나눕니다. NAT Gateway 하나를 공유하면 그 AZ가 죽을 때 다른 AZ의 private subnet도 인터넷이 끊깁니다.
- 합의가 필요한 클러스터(ZooKeeper, Kafka KRaft controller)는 과반수를 지키려고 AZ 3곳에 나눠 둡니다. ([Apache ZooKeeper](/2020/09/19/post-zookeeper.html#quorum))

## 5. 이전에 구성했던 방식과 개선안 {#before}

### 이전 구성

public subnet 두 개와 private subnet 하나로 구성했습니다.

<div class="table-wrap" markdown="1">

| Subnet | Route Table | 경로 |
| --- | --- | --- |
| Public Subnet 01 | Public | local, Internet Gateway, S3 endpoint, VPC Peering |
| Public Subnet 02 | Public | 위와 같음 |
| Private Subnet | Private | local, NAT Gateway, S3 endpoint, VPC Peering |

</div>

### 무엇이 문제였나

- <span class="pill bad">단일 AZ</span> private subnet이 하나라, private 영역의 서버와 DB가 모두 AZ 하나에 있었습니다. public subnet은 둘이라 인터넷 쪽 진입점은 두 AZ에 둘 수 있었지만, 그 뒤의 서버는 그렇지 않았습니다.
- <span class="pill warn">개선</span> NAT Gateway가 하나라 그 AZ가 죽으면 private subnet의 외부 통신이 끊깁니다.
- <span class="pill ok">양호</span> S3 Gateway endpoint를 private route table에도 넣어, S3 트래픽이 NAT를 거치지 않았습니다.

### 개선안

AZ마다 public과 private subnet을 한 쌍씩 두고, private route table을 AZ별로 나눠 같은 AZ의 NAT Gateway를 쓰게 합니다. 4장의 그림이 개선안입니다.

## 6. 고민 사항에 대한 답 {#answers}

원문에 남겨 둔 질문입니다.

**질문** Data 저장소(DB)를 물리적으로 나눠야 하는가? 나눈다면 각 저장소의 데이터를 어떻게 동기화하고 운영해야 하는가?
{: .q}

나누되, 동기화는 직접 하지 않고 저장소의 복제 기능에 맡깁니다.

<div class="table-wrap" markdown="1">

| 저장소 | 방식 | 장애 시 |
| --- | --- | --- |
| RDS Multi-AZ (instance) | 다른 AZ에 동기 복제 standby. 읽기는 받지 않음 | DNS가 standby로 자동 전환, 보통 1~2분 |
| RDS Multi-AZ (cluster) | standby 2대가 읽기도 받음 | 더 빠른 전환 |
| Aurora | 저장 계층이 AZ 3곳에 6벌 복제 | reader를 writer로 승격 |
| 직접 운영하는 MySQL·MongoDB | 복제(replica set)를 AZ에 나눠 배치 | 복제 구성의 선출 규칙에 따름 |

</div>

동기 복제는 쓰기마다 다른 AZ까지 다녀오므로 쓰기 지연이 조금 늘고, AZ 간 전송 요금이 붙습니다.

**질문** Hadoop HBase의 이중화는 어떻게 하는가? Data Node가 아닌 Name Node는?
{: .q}

DataNode는 블록을 여러 벌 복제하므로(기본 3벌) rack awareness 설정으로 AZ를 rack처럼 다루면 AZ에 나눠 저장됩니다. 단일 장애점은 NameNode였고, HDFS HA로 해결합니다.

```
NameNode(active) ──edit log──▶ JournalNode × 3 (AZ 3곳)  ◀──읽기── NameNode(standby)
      │                                                            │
   ZKFC ──────────────── ZooKeeper × 3 (leader 선출) ──────────── ZKFC
```

- active와 standby NameNode가 JournalNode 과반수에 edit log를 공유합니다.
- 각 NameNode 옆의 ZKFC(ZKFailoverController)가 ZooKeeper의 임시 노드로 active를 정합니다. active가 죽으면 세션이 끊겨 standby가 승격합니다.
- HBase master도 같은 방식으로 ZooKeeper를 통해 active를 정합니다.

ZooKeeper의 임시 노드와 leader 선출은 [Apache ZooKeeper](/2020/09/19/post-zookeeper.html#recipes) 글에 정리했습니다. 지금 새로 구성한다면 HDFS 대신 S3를 저장소로 쓰는 경우가 많고, 그러면 NameNode 이중화 문제 자체가 사라집니다.

## 7. 데이터 레이크 운영에서의 네트워크 비용 {#cost}

이후 Kafka에서 S3 위의 Iceberg 테이블로 CDC 이벤트를 적재하는 작업을 하면서, 네트워크 구성이 비용으로 바로 연결되는 지점을 다시 보게 됐습니다.

<div class="table-wrap" markdown="1">

| 트래픽 | 경로 | 비용 |
| --- | --- | --- |
| private subnet → S3 | Gateway endpoint | <span class="pill ok">무료</span> |
| private subnet → S3 | NAT Gateway | <span class="pill bad">GB당 처리 요금</span> 대량 적재면 큼 |
| private subnet → Glue Catalog, STS | NAT 또는 Interface endpoint | 요청 위주라 작음 |
| Kafka broker ↔ consumer, 다른 AZ | AZ 간 전송 | GB당 양방향 요금. `client.rack`으로 같은 AZ replica에서 읽게 해 줄일 수 있음 |

</div>

- Iceberg writer는 데이터 파일을 S3에 쓰고, 커밋할 때마다 Glue 카탈로그를 호출합니다. 커밋이 잦으면 Glue 호출과 작은 metadata 파일 쓰기가 늘어나므로, 커밋 빈도는 네트워크 요청 수와도 직결됩니다. ([Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html))
- Gateway endpoint는 같은 Region의 S3에만 적용됩니다. 다른 Region 버킷은 여전히 NAT를 거칩니다.

## 8. 새롭게 알게 된 것 {#learned}

- private subnet의 인스턴스는 public IP가 없어도 인터넷으로 나갈 수 있습니다. NAT Gateway 덕분입니다.
- Multi-AZ는 물리적으로 분리된 AZ에 시스템을 나눠 두는 고가용성 구성입니다. ALB만 여러 AZ에 걸쳐 있다고 Multi-AZ가 아니고, 그 뒤의 서버와 저장소, NAT Gateway까지 AZ마다 있어야 합니다.
- S3 트래픽은 Gateway endpoint로 보내야 NAT 요금이 붙지 않습니다.

## 9. 성과와 기대 효과 {#outcome}

### 기존 구성의 성과

<div class="table-wrap" markdown="1">

| 구분 | 성과 |
| --- | --- |
| 네트워크 분리 | public과 private 영역을 route table로 나눠, 서버와 DB를 인터넷에서 직접 접근할 수 없는 private subnet에 둠 |
| 외부 통신 | private subnet의 서버가 public IP 없이 NAT Gateway로만 외부에 나가도록 구성 |
| 비용 | S3 Gateway endpoint를 public과 private route table 양쪽에 넣어, S3 트래픽이 NAT를 거치지 않고 NAT 처리 요금도 붙지 않음 |
| 연결 | VPC Peering으로 다른 VPC와 사설 IP로 통신 |
| 진입점 | public subnet을 두 AZ에 둬, 인터넷 쪽 진입점은 AZ를 나눠 배치할 수 있는 구조 |

</div>

### 개선을 적용하면 얻는 것

5장의 개선안(AZ마다 public·private subnet 한 쌍, AZ별 NAT Gateway)과 6장의 저장소 이중화를 적용했을 때의 예상입니다.

<div class="table-wrap" markdown="1">

| 개선 작업 | 무엇이 바뀌나 | 지금 | 적용 후 (예상) | 근거 |
| --- | --- | --- | --- | --- |
| **private subnet을 AZ마다 하나씩 두기** | AZ 하나 장애 시 서버 영역 | 전체 중단 | 남은 AZ에서 계속 운영 | 5장 |
| **NAT Gateway를 AZ마다 두기** | NAT 단일 장애점 | 1개 (그 AZ가 죽으면 외부 통신 단절) | 없음 (AZ마다 자기 NAT 사용) | 4장 |
| **RDS를 Multi-AZ로 전환** | DB가 있는 AZ 장애 시 | DB가 단일 AZ에 있어 함께 중단 | standby로 자동 전환 (보통 1~2분) | 6장 |
| **ZooKeeper·KRaft controller를 AZ 3곳에 나눠 배치** | 합의 클러스터의 AZ 장애 내성 | 이전 구성에서 다루지 않음 | AZ 1곳이 죽어도 과반수 유지 | 4장 |
| **HDFS NameNode를 HA로 구성** | NameNode 장애 시 | 단일 NameNode라면 클러스터 중단 | ZKFC가 standby를 자동 승격 | 6장 |
| **Kafka consumer에 `client.rack` 설정** | consumer의 AZ 간 전송 | 기본 설정이면 다른 AZ의 leader에서 읽음 | 같은 AZ의 replica에서 읽음 | 7장 |

</div>

> **대가로 내주는 것**
>
> 비용: NAT 1개 → **NAT 시간당 요금 × AZ 수 + AZ 간 전송 요금**. AZ별 NAT와 동기 복제로 얻는 가용성의 대가입니다. (4장, 6장)
{: .callout .warn}

> **측정 방법**
>
> 가용성은 실제로 장애를 일으켜 확인합니다. 한 AZ의 private subnet 인스턴스를 내려 서비스가 계속되는지 보고, RDS는 "장애 조치로 재부팅"으로 전환 시간을 잽니다. AWS Fault Injection Service로 AZ 단위 장애를 연습할 수도 있습니다. 비용은 Cost Explorer에서 NAT Gateway와 데이터 전송 항목을 적용 전후로 비교합니다.
{: .callout .info}

* * *

## 10. 참고 {#refs}

### AWS 문서

1. [Amazon VPC 사용 설명서](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/what-is-amazon-vpc.html)
2. [NAT 게이트웨이](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/vpc-nat-gateway.html)
3. [Amazon S3용 게이트웨이 엔드포인트](https://docs.aws.amazon.com/ko_kr/vpc/latest/privatelink/vpc-endpoints-s3.html)
4. [보안 그룹과 네트워크 ACL 비교](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/infrastructure-security.html)
5. [Amazon RDS 다중 AZ 배포](https://docs.aws.amazon.com/ko_kr/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
6. [HDFS High Availability Using the Quorum Journal Manager](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithQJM.html)

### 정리가 잘된 블로그

1. [AWS VPC를 디자인해보자(1) - Multi AZ와 Subnet을 활용한 고가용성](https://bluese05.tistory.com/45?category=559701)
2. [AWS VPC를 디자인해보자(2) - ACL과 Security Group을 활용한 보안 강화](https://bluese05.tistory.com/47?category=559701)
3. [AWS VPC를 디자인해보자(3) - NAT Gateway 와 Bastion host](https://bluese05.tistory.com/48?category=559701)
4. [AWS VPC를 디자인해보자(4) - VPC Peering을 활용한 Multi VPC 사용하기](https://bluese05.tistory.com/49?category=559701)
