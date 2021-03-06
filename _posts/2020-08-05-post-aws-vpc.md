---
layout: post
title: AWS VPC
hide_title: false
excerpt_separator: <!--more-->
tags: [aws,vpc,network]
---

AWS에서 사용자의 시스템의 네트워크를 구성하기 위한 방법을 알아보자.

<!--more-->

AWS 를 통해 시스템을 구성함에 있어서 사용하고 있던 여러 구성 요소들이 있는데, 각각의 요소들을 정리하면서 좀 더 잘 알고 사용하자.
 
<hr/>


## VPC ( Virtual Private Cloud )

 Amazon VPC는 Amazon EC2의 네트워킹 계층입니다. 사용자의 AWS 계정 전용 가상 네트워크이다.

 Amazon Virtual Private Cloud(Amazon VPC)에서는 사용자가 정의한 가상 네트워크로 AWS 리소스를 시작할 수 있다. 

 이 가상 네트워크는 AWS의 확장 가능한 인프라를 사용한다는 이점과 함께 고객의 자체 데이터 센터에서 운영하는 기존 네트워크와 매우 유사하다.
 
 네트워크 구성을 위한 아래의 단위 개념을 잘 이해해야 한다.

#### Region

 AWS의 서비스가 위치한 지리적인 장소이며, 글로벌 기준으로 지역적 위치를 묶어서 관리하는 단위.
 

#### Availability Zone
 
 Region 내에 실제 컴퓨팅 리소스들이 물리적으로 분리되어 있는 단위이다


#### Subnet

 VPC 내 특정 IP 주소 범위 블록. [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing) 형식으로 지정한다.


#### Route Table

 네트워크 트래픽을 전달할 위치를 결정하는 데 사용되는 라우팅이라는 규칙 집합.


#### Internet Gateway

 VPC의 리소스와 인터넷 간의 통신을 활성화하기 위해 VPC에 연결하는 게이트웨이.


#### NAT Gateway

 [NAT](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/vpc-nat.html) (Network Address Translation) Gateway를 사용하여 Private Subnet의 인스턴스를 외부 인터넷 또는 기타 AWS 서비스에 연결하는 한편, 외부 인터넷에서 해당 Private Subnet의 인스턴스와의 연결 할 수 없도록 할 수 있다.


#### VPC Endpoint

 인터넷 게이트웨이, NAT 디바이스, VPN 연결 또는 AWS Direct Connect 연결을 필요로 하지 않고 PrivateLink 구동 지원 AWS 서비스 및 VPC 엔드포인트 서비스에 VPC를 비공개로 연결할 수 있다. 


<hr/>
 ✓ 트래픽이 외부로 나간다는 것을 기반으로 비용이 추가 부과될 수 있는 서비스들이 있으므로 시스템 구성에 참조할 필요가 있다

 ✓ VPC의 인스턴스는 서비스의 리소스와 통신하는데 퍼블릭 IP 주소를 필요로 하지 않습니다. VPC와 기타 서비스 간의 트래픽은 Amazon 네트워크를 벗어나지 않는다.
<hr/>

## 고가용성을 위한 Multi AZ 운영

 고가용성이란 사용 가능성이 높다라고 직설적으로 이해 할 수 있지만, 시스템 운영 개념에서 서버와 네트워크, 프로그램 등의 정보 시스템이 **상당히 오랜 기간 동안 지속적으로 정상 운영이 가능한 성질** 을 말한다. 이를 위해서 시스템의 장애를 적절히 대응 할 수 있도록 시스템을 설계 할 수 있어야 한다. 
 
 지진 등과 같은 천재 지변에도 마찬가지이다. 이런 경우에 대비하여 네트워크와 서버 장비등의 안정적인 운영을 위해 물리적으로 분리된 데이터 센터로 이중화하여 운영해야 한다. 
 
 AWS 에서는 각각의 Region별로 물리적으로 분리된 리소스를 관리하는 Availability Zone 이 있다. 시스템 설계에 있어서 이를 활용한 네트워크 설계와 인스턴스 배치를 하여 서비스 운영에 이중화를 하도록 권장하고 있다. 특히, AWS 자체적으로도 이런 구조를 권장하고 있고, 하나의 AZ의 물리 장비 장애로 발생하는 문제에 대해서 시스템을 이중화하지 않아서 손실된 부분에 대해서 책임지지 않는다고 한다.
 
 따라서 하나의 Region 에서 서비스 시스템을 구축하는 경우에는 해당 Region 에서 지원하는 AZ 중에서 두개 이상의 AZ에 각각의 시스템을 동일하게 구축하도록 한다.
 
 - 고민 사항
   - Data 저장소(DB)를 물리적으로 나눠야하는가? 나눈다면 각각의 Data 저장소의 데이터 어떻게 Sink / 운영 해야 하는가?
   - Hadoop hbase 의 이중화는 어떻게? Data Node 가 아닌 Name Node의 경우, 처리 방안은?
 
<hr/>

## 이전에 구성했던 방식은...

### VPC Subnet

두개의 Public Subnet 과 하나의 Private Subnet 으로 구성 했었으나, 진정한 Multi AZ 의 개념으로 각각의 AZ 에 Public Subnet 와 Private Subnet 이 Pair 가 되도록 구성하는 것이 맞을 것이다.

 - Public Subnet 01
   + Public Route Table
   
 - Public Subnet 02
   + Public Route Table
   
 - Private Subnet
   + Private Route Table

### Route Table

Public 영역과 Private 영역에 대한 각각의 Route Table 을 구성하여 각 Subnet에 연결한다.

 - Public Route Table
   + local : 해당 VPC IP 대역 
   + Internet Gateway
   + Service Endpoint for S3
   + VPC Peering Connections

 - Private Route Table
   + local : 해당 VPC IP 대역 
   + NAT Gateway
   + Service Endpoint for S3
   + VPC Peering Connections


<hr/>


### 내용을 정리하고 테스트하면서 새롭게 알게된 사실.

- Private Subnet 에는 public ip를 설정하지 않아도 인터넷이 됨. 당연히 NAT Gateway 덕분이다.
- Multi AZ 라는 용어에 대해서 이해는 못했는데, 시스템의 고가용성 구성을 위해서 물리적으로 구분된 리소스 운영을 위한 개념으로 AZ 는 Availability Zone 의 약자 이다.

<p/>
<hr/>
<p/>

### AWS 참조 문서
1. [VPC](https://docs.aws.amazon.com/ko_kr/vpc/index.html)
2. [Amazon VPC란 무엇입니까?](https://docs.aws.amazon.com/ko_kr/vpc/latest/userguide/what-is-amazon-vpc.html)

<p/>

### 내용이 너무 정리가 잘된 블로그 링크
1. [AWS VPC를 디자인해보자(1) - Multi AZ와 Subnet을 활용한 고가용성](https://bluese05.tistory.com/45?category=559701)
2. [AWS VPC를 디자인해보자(2) - ACL과 Security Group을 활용한 보안 강화](https://bluese05.tistory.com/47?category=559701)
3. [AWS VPC를 디자인해보자(3) - NAT Gateway 와 Bastion host](https://bluese05.tistory.com/48?category=559701)
4. [AWS VPC를 디자인해보자(4) - VPC Peering을 활용한 Multi VPC 사용하기](https://bluese05.tistory.com/49?category=559701)

