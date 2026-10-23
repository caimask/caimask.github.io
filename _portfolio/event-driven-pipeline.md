---
layout: post
title: 이벤트 기반 데이터 파이프라인 구축
date: 2026-10-23
tags: [Kafka, Kafka Connect, Debezium, CDC, Kubernetes]
author: caimask
---

원본 MongoDB의 변경을 Debezium으로 Kafka에 이벤트로 발행하고, 그 이벤트 하나를 여러 하류가 각자의 목적으로 소비하는 파이프라인을 구축해 운영했습니다. 하류 연동은 모두 Kafka Connect 플러그인으로 만들어 같은 클러스터에서 돌렸습니다. 데이터 레이크 적재, 내부 API 호출, 외부 공개용 토픽 발행, 수집 지연 측정이 모두 같은 CDC 토픽에서 출발합니다.

역할: Kafka Connect 클러스터(EKS·Helm)와 이미지 구성, CDC source 커넥터 구성과 오픈소스 패치, 커넥터 배포 CI, 이벤트 분배·트리거·적재·지연 측정 커넥터 설계·개발(2023-08 ~ 2025-10). 외부 공개용 발행 커넥터는 팀원이 개발했습니다.
{: .meta}

기술: Kafka · Kafka Connect · Debezium(MongoDB) · Schema Registry(Avro) · gRPC · Kubernetes(EKS) · Helm · Jenkins · CloudWatch
{: .meta}

<div class="summary" markdown="1">

## 한눈에 보기 {#tldr}

- **만든 것**: 원본 DB 변경 → CDC 토픽 하나 → 용도별 토픽과 커넥터로 갈라지는 구조입니다. 새 하류를 붙일 때 원본 쪽은 손대지 않습니다.
- **판단**: 하류 연동을 별도 서비스가 아니라 Kafka Connect 플러그인으로 만들었습니다. offset 관리, task 분산, 재시도, DLQ를 런타임에 맡기고 플러그인에는 변환과 호출 로직만 남겼습니다.
- **운영**: 설정으로 풀리지 않는 문제는 Avro Converter와 Debezium 코드를 직접 고쳐 이미지에 넣었습니다. 외부 API 호출 실패는 DLQ로 격리하고, 원본 변경에서 Kafka 기록까지의 지연을 10분 단위로 측정했습니다.
- **회고**: 이 글을 쓰며 분배 커넥터를 다시 보니, 비동기 발행의 실패가 offset 커밋을 막지 못하는 구조였습니다. 운영 종료로 고치지 못했고, 개선 방향을 적었습니다.

</div>

## 구조 {#architecture}

```
MongoDB ──(Debezium source)──▶ CDC 토픽
                                 │
          ┌──────────────────────┼──────────────────────┬──────────────────────┐
          ▼                      ▼                      ▼                      ▼
   이벤트 분배 커넥터       데이터 레이크 적재        외부 공개 발행          수집 지연 측정
   (테이블·작업·키 필터)    (Iceberg sink →          (팀원 개발)            (10분 단위 지표
          │                 buffer + Spark 배치)                           → CloudWatch)
    ┌─────┴──────┐
    ▼            ▼
 콘텐츠 토픽   보강 토픽
    │            │
    ▼            ▼
 정합성 보정    콘텐츠 보강
 트리거         트리거
 (gRPC 호출,   (gRPC 호출,
  실패 → DLQ)   실패 → DLQ)
```

<div class="table-wrap" markdown="1">

| 구성 요소 | 하는 일 | 시기 |
| --- | --- | --- |
| Kafka Connect 클러스터 | EKS에 Helm chart로 구성. 런타임 수정만 담은 이미지 + pod 시작 시 플러그인 설치 | 2023-08 ~ |
| CDC source | Debezium MongoDB 커넥터로 변경을 CDC 토픽에 발행. 큰 필드를 제외하려고 커넥터 코드를 고침 | 2023-08 ~ |
| 커넥터 배포 CI | 커넥터별 환경 설정 JSON을 REST API로 배포·일시정지·재시작·삭제. 배포 때 스키마 선등록 | 2023-08 ~ |
| 데이터 레이크 적재 | Iceberg sink 커넥터 → buffer sink + Spark 배치로 전환 ([별도 사례](/portfolio/cdc-iceberg-ingestion)) | 2023-08 ~ 2024-12 |
| 이벤트 분배 | CDC 토픽에서 대상 테이블·작업 종류를 골라 용도별 토픽으로, 하류가 쓸 키로 다시 키잉해 발행 | 2024-01 ~ 2025-10 |
| 트리거 2종 | 분배된 토픽을 소비해 내부 gRPC API(정합성 보정, 콘텐츠 보강) 호출 | 2024-01 ~ 2025-10 |
| 수집 지연 측정 | 원본 변경 시각과 Kafka 기록 시각의 차이를 DB별로 10분마다 집계 | 2025-06 ~ |

</div>

## 핵심 판단 {#decisions}

### 하류 연동을 Kafka Connect 플러그인으로 {#why-connect}

하류마다 consumer 서비스를 따로 만들면 offset 커밋, 재배치, 재시도, 배포를 서비스마다 다시 구현해야 합니다. Kafka Connect 플러그인으로 만들면 이 부분은 런타임이 맡고, 플러그인은 `SinkTask.put()` 안의 변환과 호출만 가집니다. 처리량은 설정의 `tasks.max`로 조절했습니다(분배 10, 보정 트리거 6, 보강 트리거 72). 배포도 설정 JSON 하나를 REST API로 반영하는 것으로 통일됐습니다.

대가는 모든 하류가 한 클러스터의 자원과 장애를 공유한다는 점입니다. 그래서 클러스터를 둘로 나눠, 자주 바뀌는 자체 커넥터와 안정화된 공개 커넥터가 서로의 재시작에 영향을 받지 않게 했습니다.

### 원본 토픽은 하나, 용도별 분배는 설정으로 {#distribute}

하류마다 필요한 것이 달랐습니다. 어떤 하류는 일부 테이블의 생성·수정만, 어떤 하류는 다른 키 기준의 이벤트를 원했습니다. CDC 커넥터를 하류마다 늘리면 원본 DB 부하와 change stream 수가 늘어납니다. 그래서 CDC는 한 번만 받고, 분배 커넥터가 설정에 따라 갈라 보내게 했습니다.

```
distribute.list = content, enhance
distribute.config.<이름>.topic          = 보낼 토픽
distribute.config.<이름>.table.regex    = 대상 테이블 정규식
distribute.config.<이름>.operation.type = c,m   (생성·수정만)
distribute.config.<이름>.key.field      = 하류가 쓸 키 필드
```

분배 대상마다 producer 하나를 두고, 메시지 키를 하류의 키 필드로 다시 만들어 발행합니다. 같은 엔티티의 이벤트는 같은 파티션으로 가므로 하류에서 순서가 유지됩니다. 새 분배 대상은 코드 변경 없이 설정 몇 줄로 추가했습니다.

### 외부 호출 실패는 격리 {#dlq}

트리거는 이벤트마다 내부 gRPC API를 동기로 호출합니다. 호출 하나가 실패할 때 task 전체를 멈추면, 한 엔티티의 문제 때문에 모든 엔티티의 처리가 멈춥니다. 그래서 트리거는 다음처럼 만들었습니다.

- 호출에 deadline(기본 1초)을 두고, 실패하면 Connect의 `ErrantRecordReporter`로 해당 레코드를 DLQ 토픽에 보냅니다. 오류 맥락은 헤더에 남깁니다.
- 커넥터의 `errors.tolerance`는 `all`로 두어 변환 단계의 실패도 DLQ로 갑니다.
- gRPC keep-alive와 그 주기를 설정으로 켜고 끌 수 있게 했습니다.

반대로 분배·적재 커넥터는 레코드를 건너뛰면 하류 데이터가 틀어지므로, 실패하면 task가 멈추도록 기본 tolerance를 유지했습니다. 커넥터마다 "멈출지, 격리하고 지나갈지"를 하류의 성격에 맞춰 정했습니다.

### 메시지 크기는 원본에서 {#message-size}

큰 본문 필드 때문에 CDC 메시지가 커졌는데, 하류에 필요한 것은 그 필드가 **바뀌었다는 사실**뿐이었습니다. Debezium의 필드 제외 기능은 `@`로 시작하는 필드를 설정할 수 없었고, 제외한 필드는 변경 목록에서도 사라졌습니다. 하류마다 필드를 버리게 하는 대신 Debezium 코드를 고쳐 원본에서 값을 빼고 변경 사실만 남겼습니다. 상세는 [Kafka Connect 클러스터 구성과 오픈소스 패치](/2026/10/23/kafka-connect-platform.html#debezium-filter)에 있습니다.

### 지연을 숫자로 {#latency}

"이벤트가 늦다"는 말이 나올 때 원본에서 늦는지 하류에서 늦는지 가를 수단이 필요했습니다. 그래서 CDC 토픽을 따로 소비하는 측정 커넥터를 만들었습니다.

- 지표 두 가지: Kafka 기록 시각 − 원본 DB 변경 시각, Kafka 기록 시각 − Debezium 처리 시각
- 원본 DB별로 10분 구간의 평균·최대·최소·건수를 계산해 CloudWatch에 기록
- 처음 배포할 때 과거 이벤트를 다시 읽지 않도록 최신 offset부터 소비

두 지표를 나눠서, 지연이 원본 DB의 change stream 쪽인지 Debezium 처리 쪽인지 구분할 수 있게 했습니다.

## 운영 기반 {#platform}

Kafka Connect 클러스터와 그 위의 수정은 상세 문서로 따로 정리했습니다.

- 이미지에는 런타임 수정만 넣고, 커넥터 플러그인은 init container가 pod 시작 때 설치합니다. 커넥터 배포와 이미지 빌드가 분리됩니다.
- Schema Registry 일시 실패로 task가 내려가던 문제는, Avro Converter가 `RetriableException`을 던지도록 고쳐 런타임 재시도 경로로 넘겼습니다.
- 패치는 같은 버전 tag에서 최소로 고치고, 수정 위치와 빌드 절차를 이미지 저장소에 남겼습니다.

## 회고 {#retro}

<div class="findings" markdown="1">

<div class="finding" markdown="1">

### <span class="pill bad">높음</span> <span class="id">E1</span> 분배 커넥터의 비동기 발행 실패가 offset 커밋을 막지 못합니다

이벤트 분배 커넥터 (발행·flush 경로)
{: .where}

분배 커넥터는 `put()`에서 producer의 `send()`를 비동기로 호출하고, `flush()`는 비워 두었습니다. Connect 런타임은 `flush()`가 정상 반환하면 그 시점까지의 offset을 커밋하므로, 아직 브로커가 확인하지 않은 메시지의 offset도 커밋될 수 있습니다. 발행 실패는 콜백에서 예외를 던지게 했지만, 콜백은 producer의 I/O 스레드에서 실행돼 task를 실패시키지 못하고 로그로만 남습니다. `acks=all`로 설정해 두었지만, 이 경로로 실패한 메시지는 재처리되지 않습니다.

**개선 방향** `flush()`에서 각 producer의 `flush()`를 호출해 미확인 메시지를 모두 기다리고, 콜백에서 실패를 기록해 두었다가 `flush()`나 다음 `put()`에서 예외로 올려 offset 커밋을 막습니다. **상태** 이 글을 쓰며 찾았고, 분배 커넥터 운영을 종료(2025-10)해 적용하지 않았습니다.

</div>

</div>

> **배운 점**
>
> Kafka Connect 플러그인을 만들면 offset 관리를 런타임에 맡길 수 있다고 생각하기 쉽습니다. 하지만 런타임은 `flush()`가 반환된 것을 "여기까지 처리 끝"으로 믿을 뿐입니다. 플러그인 안에서 비동기로 내보내는 것이 있다면, 그것이 끝났는지는 플러그인이 `flush()`에서 책임져야 합니다. 런타임에 맡긴 것과 플러그인이 지켜야 하는 것의 경계를 먼저 그었어야 했습니다.
{: .callout .info}

## 상세 문서 {#docs}

<div class="table-wrap" markdown="1">

| 문서 | 내용 |
| --- | --- |
| [Kafka Connect 클러스터 구성과 오픈소스 패치](/2026/10/23/kafka-connect-platform.html) | EKS·Helm 클러스터 구성, 커넥터 배포 CI, Avro Converter·Debezium 패치의 원인과 대가 |
| [CDC 이벤트를 Iceberg에 적재하는 파이프라인](/portfolio/cdc-iceberg-ingestion) | 같은 CDC 토픽을 데이터 레이크에 적재한 사례. 커밋 경합과 구조 전환 |

</div>
