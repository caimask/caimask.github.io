---
layout: post
title: Apache Flink
tags: [Flink, Stream Process, Data Stream]
author: caimask
---

공식 소개 문서를 옮겨 적는 데서 멈췄던 2020년 Flink 글을 다시 썼습니다. Flink가 스트림 처리에서 어려운 문제(상태, 장애 복구, 늦게 오는 이벤트)를 어떻게 푸는지 정리하고, 이후 Iceberg 적재를 검토하며 Flink의 체크포인트가 왜 "커밋 주기"의 기준이 되는지 연결했습니다.

원문 2020-09-02 · 보강 2026-10-21 · 기준 Flink 1.x ~ 2.x · 연관: [Apache Kafka](/2020/08/20/post-kafka.html) · [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#scale)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **Flink는 끝없는 이벤트 스트림에 상태를 두고 계산하는 엔진입니다.** 배치는 끝이 있는 스트림의 특수한 경우로 다룹니다.
2. **상태와 체크포인트가 핵심입니다.** 연산자의 상태를 주기적으로 스냅샷으로 저장하고, 장애가 나면 마지막 체크포인트와 그 시점의 소스 offset으로 되돌아가 다시 처리합니다. 상태 기준 exactly-once는 이렇게 만들어집니다.
3. **event time과 watermark로 늦게 오는 이벤트를 다룹니다.** 이벤트가 발생한 시각 기준으로 윈도우를 닫되, "이 시각 이전 이벤트는 다 왔다"는 판단을 watermark로 명시합니다.
4. **외부 저장소의 exactly-once는 체크포인트에 맞춘 2단계 커밋으로 만듭니다.** Iceberg sink는 체크포인트마다 한 번, 하나의 committer가 커밋합니다. 그래서 체크포인트 주기가 곧 Iceberg 커밋 주기입니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [스트림 처리에서 어려운 것](#hard)
2. [제한 없는 스트림과 제한된 스트림](#bounded)
3. [상태와 체크포인트](#state)
4. [event time과 watermark](#time)
5. [외부 저장소와 exactly-once](#sink)
6. [배포와 실행](#deploy)
7. [Iceberg 적재에서 다시 만난 Flink](#practice)
8. [참고](#refs)

</div>

## 1. 스트림 처리에서 어려운 것 {#hard}

이벤트를 하나씩 받아 변환만 하는 일은 어렵지 않습니다. 어려운 것은 다음 세 가지입니다.

<div class="table-wrap" markdown="1">

| 문제 | 예 | Flink의 답 |
| --- | --- | --- |
| 상태 | "회원별 최근 10분 결제 합계"처럼 이전 이벤트를 기억해야 함 | keyed state (로컬 메모리 또는 RocksDB) |
| 장애 복구 | 처리 중에 죽으면 어디부터 다시 할지, 상태는 어떻게 되돌릴지 | 체크포인트 (상태 + 소스 위치를 함께 저장) |
| 시간 | 모바일 이벤트가 네트워크 사정으로 몇 분 늦게 도착 | event time + watermark |

</div>

## 2. 제한 없는 스트림과 제한된 스트림 {#bounded}

<div class="table-wrap" markdown="1">

|   | 제한 없는 스트림 (unbounded) | 제한된 스트림 (bounded) |
| --- | --- | --- |
| 끝 | 없음. 계속 들어옴 | 있음 |
| 처리 방식 | 도착하는 대로 처리. 전체를 기다릴 수 없음 | 다 모은 뒤 처리 가능 |
| 정렬 | 순서가 중요하면 시간 기준을 정해야 함 | 언제든 정렬 가능 |
| 예 | Kafka 토픽, CDC 이벤트 | 하루치 파일, 테이블 스냅샷 |

</div>

Flink는 제한된 스트림을 "끝이 있는 스트림"으로 보고 같은 API로 처리합니다. DataStream API에서 실행 모드를 `BATCH`로 바꾸면 같은 코드가 배치 최적화(정렬 기반 처리, 체크포인트 없음)로 돕니다. Flink 2.0에서는 배치 전용이던 DataSet API가 제거되고 이 방식으로 통합됐습니다.

## 3. 상태와 체크포인트 {#state}

Flink는 키별 상태를 연산자 가까이(TaskManager의 메모리나 로컬 RocksDB)에 두어 외부 저장소를 거치지 않고 빠르게 읽고 씁니다. 대신 그 상태를 잃지 않도록 주기적으로 체크포인트를 남깁니다.

```
Kafka ─▶ [source] ─▶ [keyBy(user) → 합계 상태] ─▶ [sink]
           │ barrier n이 흐름을 따라 내려감
           ├ source:   "partition 0 offset 1520" 저장
           ├ 합계 연산: 상태 스냅샷 저장 (S3 등 내구성 저장소)
           └ sink:     barrier 도착 = 체크포인트 n 완료 준비

장애 발생 → 체크포인트 n의 상태 복원 + Kafka offset 1520부터 다시 읽기
```

- 체크포인트 barrier가 데이터와 함께 흘러가며, barrier를 받은 연산자가 그 순간의 상태를 저장합니다. 전체 흐름을 멈추지 않고 일관된 스냅샷을 얻는 방법입니다.
- 복구하면 소스 offset과 상태가 같은 시점으로 돌아가므로, **상태 기준으로는** 각 이벤트가 정확히 한 번 반영됩니다.
- 상태가 크면 RocksDB와 증분 체크포인트를 씁니다. Flink 2.0은 상태를 원격 저장소에 두는 분리형 상태 저장(ForSt)을 도입해, 상태 크기와 로컬 디스크의 결합을 풀었습니다.

> **체크포인트와 savepoint**
>
> 체크포인트는 장애 복구용으로 Flink가 자동으로 만들고 지웁니다. savepoint는 사람이 버전 업그레이드나 job 변경을 위해 직접 만드는 것으로, job을 멈추고 코드를 바꾼 뒤 그 지점부터 이어서 실행할 수 있습니다.
{: .callout .info}

## 4. event time과 watermark {#time}

<div class="table-wrap" markdown="1">

| 시간 기준 | 의미 | 결과 |
| --- | --- | --- |
| processing time | Flink가 이벤트를 처리한 시각 | 빠르고 단순하지만, 지연·재처리 때 결과가 달라짐 |
| event time | 이벤트가 실제로 발생한 시각 (이벤트 안의 타임스탬프) | 재처리해도 같은 결과. 늦게 오는 이벤트를 어떻게 기다릴지 정해야 함 |

</div>

watermark는 "시각 T 이전의 이벤트는 더 오지 않는다"는 선언입니다. 보통 지금까지 본 가장 큰 event time에서 허용 지연을 뺀 값으로 만듭니다.

```
허용 지연 = 2분
이벤트 도착: 10:03, 10:01, 10:05, 09:59(늦음), 10:06
watermark:   10:01, 10:01, 10:03, 10:03,       10:04
                                               └ 10:04 이전에 끝나는 윈도우는 닫아도 됨
```

- watermark가 윈도우 끝을 지나면 윈도우를 계산하고 내보냅니다.
- watermark보다 늦게 온 이벤트는 버리거나, `allowedLateness`로 결과를 갱신하거나, side output으로 따로 모읍니다.

> **벽시계 시간으로 윈도우를 닫으면 생기는 문제**
>
> 이벤트 시각 기준 10분 파일을 만들면서 "벽시계로 10분이 지났으니 그 윈도우는 끝났다"고 판단하면, 늦게 도착한 이벤트가 이미 처리된 윈도우 경로에 나중에 쓰여 조용히 누락됩니다. watermark가 하는 일을 직접 구현할 때 빠지기 쉬운 함정입니다. ([Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html#handoff))
{: .callout .warn}

## 5. 외부 저장소와 exactly-once {#sink}

체크포인트는 Flink 내부 상태만 되돌립니다. 이미 외부 DB에 쓴 결과는 되돌릴 수 없습니다. 그래서 sink는 체크포인트에 맞춘 **2단계 커밋**을 씁니다.

1. **pre-commit**: 체크포인트 사이에는 결과를 임시로 써 둡니다(트랜잭션을 열어 두거나, 아직 등록하지 않은 파일로).
2. **commit**: 체크포인트가 모든 연산자에서 완료되면, 그때 임시 결과를 확정합니다.
3. 장애가 나면 확정되지 않은 임시 결과는 버리고, 체크포인트부터 다시 처리합니다.

Kafka sink(트랜잭션), 파일 sink, Iceberg sink가 모두 이 방식입니다. 대가로 **결과가 체크포인트 주기만큼 늦게 보입니다.**

## 6. 배포와 실행 {#deploy}

<div class="table-wrap" markdown="1">

| 구성 요소 | 역할 |
| --- | --- |
| JobManager | job 계획, task 배치, 체크포인트 조율. HA 구성 시 ZooKeeper 또는 Kubernetes로 leader 선출 |
| TaskManager | 실제 연산을 실행하는 worker. slot 단위로 task를 받음 |
| 리소스 관리자 | Kubernetes(Flink Kubernetes Operator), YARN, standalone |

</div>

2020년 글에 적었던 Apache Mesos 지원은 Flink 1.14에서 제거됐습니다. 지금은 Kubernetes Operator로 배포하거나, 관리형 서비스(Amazon Managed Service for Apache Flink 등)를 쓰는 경우가 많습니다.

## 7. Iceberg 적재에서 다시 만난 Flink {#practice}

CDC 이벤트를 Iceberg에 적재하는 커넥터의 커밋 경합을 검토할 때, 비교 대상으로 가장 먼저 꼽은 것이 Flink Iceberg sink의 구조였습니다.

```
[Kafka 파티션 1..N] → [IcebergStreamWriter × N: Parquet 병렬 쓰기]
                                  ↓ 체크포인트마다 파일 목록만 전달
                      [IcebergFilesCommitter × 1: 체크포인트 완료 시 테이블당 1회 커밋]
```

<div class="table-wrap" markdown="1">

| Flink 개념 | Iceberg 적재에서의 의미 |
| --- | --- |
| 병렬 writer | 처리량은 writer 수로 늘림. 커밋 횟수는 그대로 |
| committer parallelism 1 | 테이블당 커밋 주체가 하나라 커밋 경합이 없음 |
| 체크포인트 주기 | Iceberg 커밋 주기 = 데이터가 보이기까지의 지연. 보통 1~5분 |
| 2단계 커밋 | 체크포인트에 포함된 파일만 커밋되므로 장애 시 중복·누락 없음 |
| upsert 모드 | 같은 체크포인트 안에서 같은 키가 다시 오면 position delete, 이전 커밋의 행은 equality delete. 컴팩션 운영이 필요 |

</div>

직접 만든 커넥터는 writer마다 몇 초 간격으로 각자 커밋해서, Flink sink가 처음부터 피하는 구조를 거꾸로 갖고 있었습니다. ([Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#scale))

> **배운 점**
>
> 스트림 처리의 정확성은 "상태와 입력 위치를 같은 시점으로 묶는 것", 외부 저장소의 정확성은 "그 시점에 맞춰 한 번에 확정하는 것"에서 나옵니다. Flink가 프레임워크로 해 주는 이 두 가지를 직접 구현하면 대부분 여기서 틀립니다.
{: .callout .info}

* * *

## 8. 참고 {#refs}

- [What is Apache Flink? Architecture](https://flink.apache.org/what-is-flink/flink-architecture/)
- [Stateful Stream Processing](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/stateful-stream-processing/)
- [Timely Stream Processing (event time, watermark)](https://nightlies.apache.org/flink/flink-docs-stable/docs/concepts/time/)
- [Apache Iceberg: Flink Writes](https://iceberg.apache.org/docs/latest/flink-writes/)
- [Apache Flink 2.0.0 release announcement](https://flink.apache.org/2025/03/24/apache-flink-2.0.0-a-new-era-of-real-time-data-processing/)
