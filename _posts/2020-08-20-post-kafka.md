---
layout: post
title: Apache Kafka
tags: [Kafka, Kinesis]
author: caimask
---

### Apache Kafka 란?
아파치 카프카(Apache Kafka)는 아파치 소프트웨어 재단이 스칼라로 개발한 오픈 소스 메시지 브로커 프로젝트이다. 

이 프로젝트는 실시간 데이터 피드를 관리하기 위해 통일된, 높은 처리량, 낮은 지연시간을 지닌 플랫폼을 제공하는 것이 목표이다. 

요컨대 분산 트랜잭션 로그로 구성된, 상당히 확장 가능한 pub/sub 메시지 큐로 정의할 수 있으며, 스트리밍 데이터를 처리하기 위한 기업 인프라를 위한 고부가 가치 기능이다.

참조 : [위키 아파치 카프카](https://ko.wikipedia.org/wiki/%EC%95%84%ED%8C%8C%EC%B9%98_%EC%B9%B4%ED%94%84%EC%B9%B4)



### To Use Kafka
Kafka 의 사용에 앞서서 Kafka 가 어떤 것이며, 어떤 기능을 하는지 이해는 것이 중요하다. 

앞에서는 단순한 정의를 기술한 것이라면 여기에서는 Kafka 의 기능과 동작에 대해서 이해한 것을 정리해 보자.


#### Kafka는 아래 세 가지 주요 기능의 결합으로 이벤트 스트리밍 종단 간 Use-Case를 구현 할 수 있다.

- 다른 시스템에서 데이터의 지속적인 가져 오기 / 내보내기를 포함하여 이벤트 스트림을 **publish (write)** 및 **subscribe (read)** 합니다.
  - 이벤트 메세지 큐로써 이벤트를 **큐에 추가하고 (생성), 가져 올 수 있는 (소모) 기능**을 제공 한다.


- 원하는 기간 동안 지속적이고 안정적으로 이벤트 스트림 을 **저장** 합니다.
  - 이벤트 메세지 큐에 추가된 이벤트는 **지정된 기간** 동안 큐 내부에 유지된다.


- 이벤트가 발생하는 즉시 또는 소급하여 **처리**
  - 이벤트 메세지 큐에 특정 point 를 지정하여 이벤트를 소모 할 수 있다.


#### Kafka 주요 개념

- producer : 메세지 생산자
- consumer : 메세지 소비자
- consumer group : consumer 들끼리 메세지를 나눠서 가져간다. offset 을 공유하여 중복으로 가져가지 않는다.
- broker : 카프카 서버를 가리킴
- zookeeper : 카프카 서버 (+클러스터) 상태를 관리하고
- cluster : 브로커들의 묶음
- topic : 메세지 종류
- partitions : topic 이 나눠지는 단위
- log : 1개의 메세지
- offset : 파티션 내에서 각 메시지가 가지는 unique id


#### Kafka & Zookeeper

Kafka를 운영하기 위해서 Zookeeper를 함께 구성하도록 하고 있다. 따라서 Zookeeper 가 무엇이고 어떤 역할을 하는지 이해해야 한다.

![Kafka Components]({{ "/assets/img/kafka_components.png"}}) 

- zookeeper 가 kafka 의 상태와 클러스터 관리를 해준다.

: kafka는 클러스터에서 전체 컨트롤러 상태를 관리하는 방법이 필요합니다. 

어떤 이유로 든 컨트롤러가 중단되면 나머지 브로커 세트에서 다른 컨트롤러를 선택하기위한 프로토콜이 있습니다. 

컨트롤러 선택, 심장 박동 등의 실제 메커니즘은 크게 ZooKeeper에서 구현됩니다. 

ZooKeeper는 클러스터 메타 데이터, 리더 팔로워 상태, 할당량, 사용자 정보, ACL 및 기타 하우스 키핑 항목을 유지 관리하는 구성 저장소 역할도합니다. 

근본적인 험담 및 합의 프로토콜로 인해 ZooKeeper 노드의 수는 홀수여야합니다.


#### Kafka API

1. Admin API  

: 여러 topic, broker 및 기타 Kafka 객체를 관리하고 검사하기 위한 API

2. Producer API  

: 하나 이상의 Kafka topic에 이벤트 스트림을 Publish (Write) 하기 위한 API

3. Consumer API  

: 하나 이상의 topic를 Subscribe (Read) 하고 생성된 이벤트 스트림을 처리 하기 위한 API

4. Kafka Streams API 

: 스트림 처리 애플리케이션 및 마이크로 서비스 구현을 위한 API 

5. Kafka Connect API 

: Kafka와 통합 할 수 있도록 외부 시스템 및 애플리케이션에서 이벤트 스트림을 consume (read) 또는 produce (write) 하는 재사용 가능한 데이터 가져오기 / 내보내기 커넥터를 구축하고 실행하는 API




참조 : [Kafka Introduction](https://kafka.apache.org/intro)


### Kafka VS Kinesis

![Kafka VS Kinesis]({{ "/assets/img/kafka_vs_kinesis.png"}})


출처 / 참조 : [링크](https://devidea.tistory.com/68)
