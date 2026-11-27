---
layout: post
title: 이벤트 기반 데이터 파이프라인 구축
date: 2026-10-23
tags: [Kafka, Kafka Connect, Debezium, CDC, Kubernetes]
author: caimask
---

원본 DB(처음 AWS DocumentDB, 이후 MongoDB Atlas)의 변경을 Debezium으로 Kafka에 이벤트로 발행하고, 그 이벤트 하나를 여러 하류가 각자의 목적으로 소비하는 파이프라인을 구축해 운영했습니다. 하류 연동은 모두 Kafka Connect 플러그인으로 만들어 같은 클러스터에서 돌렸습니다. 데이터 레이크 적재, 내부 API 호출, 외부 공개용 토픽 발행, 수집 지연 측정이 모두 같은 CDC 토픽에서 출발합니다. 이벤트를 만들어 토픽으로 전달하는 부분과 지연 측정 커넥터는 지금도 운영 중이고, 제가 만든 분배·트리거 커넥터는 2025-09에 운영을 종료했습니다.

역할: Kafka Connect 클러스터(EKS·Helm)와 이미지 구성, CDC source 커넥터 구성과 오픈소스 패치, 커넥터 배포 CI, 이벤트 분배·트리거·적재·지연 측정 커넥터 설계·개발(2023-08 ~ 2025-09). 외부 공개용 발행 커넥터는 팀원이 개발했습니다.
{: .meta}

기술: Kafka · Kafka Connect · Debezium(MongoDB) · Schema Registry(Avro) · gRPC · Kubernetes(EKS) · Helm · Jenkins · CloudWatch
{: .meta}

<div class="summary" markdown="1">

## 한눈에 보기 {#tldr}

- **만든 것**: 원본 DB 변경 → CDC 토픽 하나 → 용도별 토픽과 커넥터로 갈라지는 구조입니다. 새 하류를 붙일 때 원본 쪽은 손대지 않습니다.
- **판단**: 하류 연동을 별도 서비스가 아니라 Kafka Connect 플러그인으로 만들었습니다. offset 관리, task 분산, 재시도, DLQ를 런타임에 맡기고 플러그인에는 변환과 호출 로직만 남겼습니다.
- **운영**: 설정으로 풀리지 않는 문제는 Avro Converter와 Debezium 코드를 직접 고쳐 이미지에 넣었습니다. 외부 API 호출 실패는 DLQ로 격리하고(2025-01), 원본 변경에서 Kafka 기록까지의 지연을 10분 구간으로 측정했습니다.
- **회고**: 코드를 다시 읽어 보니 분배 커넥터는 비동기 발행의 실패가 offset 커밋을 막지 못하는 구조였고, 트리거는 API 실패만 DLQ로 격리했습니다. 적용하지 않았고, 개선 방향을 적었습니다.

</div>

## 구조 {#architecture}

```
MongoDB ──(Debezium source)──▶ CDC 토픽
                                 │
          ┌──────────────────────┼──────────────────────┬──────────────────────┐
          ▼                      ▼                      ▼                      ▼
   이벤트 분배 커넥터       데이터 레이크 적재        외부 공개 발행          수집 지연 측정
   (테이블·작업·키 필터)    (Iceberg sink →          (팀원 개발)            (10분 구간 집계
          │                 buffer + Spark 배치)                           → CloudWatch Logs)
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
| 이벤트 분배 | CDC 토픽에서 실제로 바뀐 생성·수정만 골라, 대상 테이블별로 하류가 쓸 키로 다시 키잉해 용도별 토픽에 발행 | 2024-01 ~ 2025-09 (운영 종료) |
| 트리거 2종 | 분배된 토픽을 소비해 내부 gRPC API(정합성 보정, 콘텐츠 보강)를 레코드마다 호출. 실패는 DLQ로(2025-01 ~) | 2024-01 ~ 2025-09 (운영 종료) |
| 수집 지연 측정 | 원본 변경 시각·Debezium 처리 시각과 Kafka 기록 시각의 차이를 파티션·DB별 10분 구간으로 집계 | 2025-06 ~ |

</div>

## 핵심 판단 {#decisions}

<div class="table-wrap" markdown="1">

| 판단 | 한 일 |
| --- | --- |
| 하류 연동은 Kafka Connect 플러그인으로 | offset 커밋, task 분산, 재시도, DLQ는 런타임 기능을 쓰고 플러그인에는 변환과 호출만 두었습니다. 처리량은 `tasks.max`로(분배 10, 보정 트리거 6, 보강 트리거 72. 보강 트리거는 2025-01에 파티션 증설과 함께 12에서 늘림), 배포는 설정 JSON 하나를 REST API로 반영합니다 |
| 하류에는 "실제로 바뀐 것"만 | update 이벤트 중 수정 시각이 바뀐 것만 자체 작업 코드 `m`으로 표시하고, 생성(`c`)과 `m`만 분배했습니다. 메시지에는 문서 내용 없이 엔티티 종류, 키, 작업 코드, 작업 시각만 싣습니다 |
| 분배는 설정으로, 설정 오류는 그 분배 안에서 | 분배 대상마다 토픽·테이블 정규식·작업 코드·키 필드를 설정 묶음으로 두고(2024-04), 키 필드가 없는 문서는 그 분배만 건너뛰게 했습니다(2024-09) |
| 외부 호출 실패는 레코드 단위로 격리 | 트리거의 gRPC 호출 실패를 처음에는 로그만 남기고 넘겼고, 2025-01부터 `ErrantRecordReporter`로 DLQ 토픽에 남겨 다시 처리할 수 있게 했습니다 |
| 메시지 크기는 원본에서 | 하류에 필요한 것은 큰 본문 필드가 **바뀌었다는 사실**뿐이라, Debezium 코드를 고쳐 원본에서 값을 빼고 변경 사실만 남겼습니다 |
| 지연은 숫자로 | 원본 DB 변경 시각과 Debezium 처리 시각에서 Kafka 기록까지의 지연을 따로 계산해, 지연이 어느 구간에서 생기는지 가를 수 있게 했습니다(2025-06) |

</div>

커넥터별 동작과 설정은 [CDC 이벤트 분배·트리거·지연 측정 커넥터 설계와 회고](/2026/11/26/cdc-event-distribution-connectors.html)에, Debezium 패치는 [쿠버네티스 위 Kafka Connect 클러스터 구성과 오픈소스 패치](/2026/10/23/kafka-connect-platform.html#debezium-filter)에 정리했습니다.

## 운영 기반 {#platform}

Kafka Connect 클러스터와 그 위의 수정은 상세 문서로 따로 정리했습니다.

- 이미지에는 런타임 수정만 넣고, 커넥터 플러그인은 init container가 pod 시작 때 설치합니다. 커넥터 배포와 이미지 빌드가 분리됩니다.
- Schema Registry 일시 실패로 task가 내려가던 문제는, Avro Converter가 `RetriableException`을 던지도록 고쳐 런타임 재시도 경로로 넘겼습니다.
- 패치는 같은 버전 tag에서 최소로 고치고, 수정 위치와 빌드 절차를 이미지 저장소에 남겼습니다.

## 회고 {#retro}

이 글들을 쓰며 코드를 다시 읽고 찾은 결함입니다. 세 항목 모두 적용하지 않았습니다. E1·E2의 커넥터는 2025-09에 운영을 종료했습니다.

- <span class="pill bad">높음</span> **E1 분배 커넥터**: `send()`를 비동기로 호출하고 `flush()`를 비워 두어, 브로커가 확인하지 않은 메시지의 offset도 커밋될 수 있습니다. 실패 콜백의 예외는 producer I/O 스레드에서 끝나 task에 전달되지 않습니다. → `flush()`에서 발행 완료를 기다리고 실패를 예외로 올립니다.
- <span class="pill warn">중간</span> **E2 트리거**: DLQ로 보내는 경로가 gRPC 실패에만 있어, 키 변환에 실패하면 task가 멈춥니다. → 레코드 처리 전체를 DLQ 경로로 감쌉니다.
- <span class="pill info">낮음</span> **E3 지연 측정**: 다음 이벤트가 와야 10분 구간을 닫아, 이벤트가 뜸하면 지표가 비고 경보를 걸기 어렵습니다. → 구간을 벽시계로 닫고 지표로 기록합니다.

> **배운 점**
>
> Kafka Connect 플러그인을 만들면 offset 관리를 런타임에 맡길 수 있다고 생각하기 쉽습니다. 하지만 런타임은 `flush()`가 반환된 것을 "여기까지 처리 끝"으로 믿을 뿐이고, `errors.tolerance`도 플러그인 바깥 단계에만 적용됩니다. 런타임에 맡긴 것과 플러그인이 지켜야 하는 것의 경계를 먼저 그었어야 했습니다.
{: .callout .info}

## 상세 문서 {#docs}

<div class="table-wrap" markdown="1">

| 문서 | 내용 |
| --- | --- |
| [CDC 이벤트 분배·트리거·지연 측정 커넥터 설계와 회고](/2026/11/26/cdc-event-distribution-connectors.html) | 분배 커넥터의 이벤트 모델·필터·설정 구조, 트리거의 호출과 DLQ, 지연 측정 방식, 회고 E1~E3 |
| [쿠버네티스 위 Kafka Connect 클러스터 구성과 오픈소스 패치](/2026/10/23/kafka-connect-platform.html) | EKS·Helm 클러스터 구성, 커넥터 배포 CI, Avro Converter·Debezium 패치의 원인과 대가 |
| [CDC 이벤트를 Iceberg에 적재하는 파이프라인](/portfolio/cdc-iceberg-ingestion) | 같은 CDC 토픽을 데이터 레이크에 적재한 사례. 커밋 경합과 구조 전환 |

</div>
