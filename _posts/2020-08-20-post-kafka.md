---
layout: post
title: Apache Kafka
tags: [Kafka, Kinesis, Data Stream]
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


#### Kafka 는 어떻게 동작하는가?

Kafka는 고성능 TCP 네트워크 프로토콜을 통해 통신하는 Server 와 Client 로 구성된 분산 시스템이다.

- Server
  - Kafka는 여러 데이터 센터 또는 클라우드 지역에 걸쳐있을 수 있는 하나 이상의 서버 클러스터로 실행된다. 이러한 서버 중 일부는 **브로커**라고 하는 스토리지 계층을 형성한다. 다른 서버는 Kafka Connect 를 실행 하여 데이터를 이벤트 스트림으로 지속적으로 가져오고 내 보내어 관계형 데이터베이스 및 기타 Kafka 클러스터와 같은 기존 시스템과 Kafka를 통합한다. 
Kafka 클러스터는 확장성이 뛰어나고 내결함성이 있다. 서버 중 하나에 장애가 발생하면 다른 서버가 작업을 대신하여 데이터 손실없이 지속적인 운영을 보장한다


- Client
  - 네트워크 문제 또는 머신 장애가 발생하는 경우에도 내결함성 방식으로 대규모로 이벤트 스트림을 읽고, 쓰고, 처리하는 분산 애플리케이션 및 마이크로 서비스를 구성 할 수 있다. Kafka 는 Kafka 커뮤니티에서 제공 하는 수십 개의 Client에 의해 보강 된 일부 Client와 함께 제공된다. Client는 Go, Python, C / C ++ 및 기타 많은 프로그래밍 언어와 REST API를위한 상위 수준 Kafka Streams 라이브러리를 포함하여 Java 및 Scala에 사용할 수 있습니다



#### Kafka & Zookeeper

Kafka를 운영하기 위해서 Zookeeper를 함께 구성하도록 하고 있다. 따라서 Zookeeper 가 무엇이고 어떤 역할을 하는지 이해해야 한다.




#### Kafka API

1. Admin API  : 여러 topic, broker 및 기타 Kafka 객체를 관리하고 검사하기 위한 API

2. Producer API  : 하나 이상의 Kafka topic에 이벤트 스트림을 Publish (Write) 하기 위한 API

3. Consumer API  : 하나 이상의 topic를 Subscribe (Read) 하고 생성된 이벤트 스트림을 처리 하기 위한 API

4. Kafka Streams API : 스트림 처리 애플리케이션 및 마이크로 서비스 구현을 위한 API 

5. Kafka Connect API : Kafka와 통합 할 수 있도록 외부 시스템 및 애플리케이션에서 이벤트 스트림을 consume (read) 또는 produce (write) 하는 재사용 가능한 데이터 가져오기 / 내보내기 커넥터를 구축하고 실행하는 API




참조 : [Kafka Introduction](https://kafka.apache.org/intro)


### Kafka VS Kinesis



참조 : [링크](https://devidea.tistory.com/68)