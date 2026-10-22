---
layout: post
title: 토스 Iceberg CDC 사례 비교
tags: [Iceberg, Kafka Connect, CDC]
author: caimask
---

토스증권이 공개한 「Iceberg 적용기 #1: CDC 환경은 왜 제대로 동작하지 않을까?」의 문제와 해결책을 우리 작업(자체 Iceberg 커넥터, buffer + Spark 배치)과 비교했습니다. 그리고 토스 방식을 구현하려면 기존 코드의 어디를 바꿔야 하는지 정리했습니다.

작성 2026-10-20 · 원문 2025-09-10 · 토스증권 Realtime Data Team · 연관: 커밋 경합 리뷰 · Buffer Writer 배치 적재 리뷰
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **토스와 우리는 같은 문제의 다른 갈래를 겪었습니다.** 토스는 처음부터 coordinator 하나가 커밋하는 커넥터(Databricks/Tabular Iceberg Sink)를 써서 커밋 경합은 없었습니다. 대신 **같은 커밋 안에서 equality delete가 제대로 동작하지 않아 중복 행이 생기는 정합성 문제**를 세 가지 경로로 겪었습니다. 우리 커넥터는 task마다 `put()`마다 커밋해서 이 중복 문제는 피했지만, 그 대가로 커밋 경합을 겪었습니다.
2. **토스의 세 가지 문제는 지금 우리 코드에서는 발생하지 않습니다.** 하지만 앞 문서에서 제안한 "커밋 묶기"를 적용하는 순간, 그중 스키마 변경 문제(이슈 3)가 그대로 생깁니다. coordinator까지 도입하면 이슈 1, 2도 해당됩니다.
3. **토스 방식을 구현하는 가장 짧은 길**은 자체 커넥터에 ① 커밋을 일정 간격으로 묶고 ② `insertedRowMap`을 writer 밖의 레지스트리로 옮기는 것입니다. 우리 커넥터의 writer 핵심부(`BaseEqualityDeltaWriter`)가 토스가 쓴 커넥터와 같은 구조라서, 고칠 지점도 거의 같습니다.
4. **토스 글은 Apache 공식 커넥터가 upsert를 지원하지 않는다고 밝혔습니다(2025년 9월 기준).** 앞 문서에서 "확인하지 못했다"고 남긴 부분입니다. 그래서 토스도 Databricks(Tabular) 버전을 썼습니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [토스 글 요약](#toss)
2. [세 가지 방식 비교](#compare)
3. [토스의 세 가지 이슈와 우리 코드](#issues)
4. [토스 방식 구현 경로](#path)
5. [코드 변경 사항 (1단계: 커밋 묶기 + 레지스트리)](#changes)
6. [코드 변경 사항 (2단계: coordinator)](#coord)
7. [배치 방식과의 선택 기준](#choose)
8. [성과와 기대 효과](#outcome)
9. [참고](#refs)

</div>

## 1. 토스 글 요약 {#toss}

### 구성

```
Kafka (Debezium CDC)
  └─ Kafka Connect: Databricks(Tabular) Iceberg Sink Connector (upsert 모드)
       Worker task 여러 개: data 파일 + position/equality delete 파일 쓰기
       Coordinator 1개:     Control Topic으로 CommitStart → 각 Worker가 DataWritten 응답
                            → 테이블당 스냅샷 1개를 원자적으로 커밋
```

### 핵심 규칙

Iceberg 스펙의 sequence number 규칙 때문에 생기는 제약입니다.

<div class="table-wrap" markdown="1">

| delete 종류 | 적용 대상 | 같은 커밋에서 쓴 행을 지울 수 있는가 |
| --- | --- | --- |
| position delete | sequence number가 **같거나 작은** data 파일 | <span class="pill ok">가능</span> |
| equality delete | sequence number가 **작은** data 파일만 | <span class="pill bad">불가</span> 같은 커밋의 이전 행이 남아 중복이 생김 |

</div>

그래서 한 커밋 안에서 같은 id가 다시 오면 반드시 position delete로 이전 행을 지워야 합니다. `BaseEqualityDeltaWriter`는 이번 커밋에서 쓴 행의 위치를 `insertedRowMap`(키 → 파일 경로, 행 위치)에 기록해 두고, 키가 맵에 있으면 position delete를, 없으면 equality delete를 씁니다. 이 방식은 **같은 키가 항상 같은 writer로 가고, 맵이 커밋 때까지 유지된다**는 전제가 있어야 동작합니다.

### 전제가 깨진 세 가지 경우와 토스의 해결

<div class="table-wrap" markdown="1">

| 이슈 | 원인 | 해결 |
| --- | --- | --- |
| 1. Kafka 키 ≠ 테이블 id | 같은 id가 다른 파티션, 다른 Worker로 가서 각자의 맵에 없음 → 같은 커밋에서 equality delete끼리 서로를 지우지 못함 | Kafka 메시지 키를 Iceberg 테이블의 id로 설정 |
| 2. 커밋 타임아웃 | 늦게 도착한 파일이 다음 커밋에 섞임. Worker는 `CommitStart`에서 writer를 새로 만들며 맵을 비움 → 섞인 커밋 안에서 equality delete가 이전 행을 못 지움 | Coordinator에 **preCommit 단계**를 추가. 새 커밋을 시작하기 전에 남은 파일을 먼저 따로 커밋 |
| 3. 스키마 변경 | 스키마가 바뀌면 파일을 닫고 writer를 새로 만듦 → 새 writer의 맵이 비어 있음 | `insertedRowMap`을 writer 밖 **Registry**로 옮기고, 커밋할 때만 비움 |

</div>

### 글에 없는 내용

- 설정값, 커밋 주기, 파티셔닝, 컴팩션 전략, 처리량 수치는 나오지 않습니다. 이런 내용은 2편에서 다룬다고 했습니다. 이 문서를 쓰는 시점(2026-10-20)에 `iceberg-cdc-2` 주소는 404였습니다.
- preCommit과 Registry의 구현 코드는 공개되지 않았습니다. 댓글로 구현 방법을 묻는 질문이 있었지만 답이 달리지 않았습니다.
- 토스는 이력(History) 테이블, 최신값(Upsert) 테이블, 삭제만 반영하는(Delete-only) 테이블을 함께 운영한다고 밝혔지만, 각각의 설정 방법은 쓰지 않았습니다.

## 2. 세 가지 방식 비교 {#compare}

<div class="table-wrap" markdown="1">

| 항목 | 토스 | 우리: 자체 Iceberg 커넥터 | 우리: buffer + Spark 배치 |
| --- | --- | --- | --- |
| 커밋 주체 | <span class="pill ok">테이블당 1</span> Coordinator | <span class="pill bad">task마다</span> 최대 8개 | <span class="pill ok">1</span> 배치 job |
| 커밋 주기 | 설정한 간격 (분 단위, 글에 수치 없음) | <span class="pill bad">몇 초</span> `put()`마다 | 10분 |
| writer 핵심부 | `BaseEqualityDeltaWriter` 계열 | `BaseEqualityDeltaWriter` (`DeltaTaskWriter`) | Spark `MERGE` (CoW) |
| 삭제 방식 | equality + position delete (MOR) | equality + position delete (MOR) | 파일 재작성 (CoW) |
| 같은 커밋 내 중복 위험 | <span class="pill warn">있었음</span> 세 가지 경우, 수정 완료 | <span class="pill ok">현재 없음</span> 커밋이 `put()` 단위라 writer와 맵의 수명이 같음 | <span class="pill ok">없음</span> |
| 커밋 경합 | <span class="pill ok">없음</span> | <span class="pill bad">심함</span> | <span class="pill ok">없음</span> (중복 실행 시 예외) |
| 최신성 | near-real-time | 수 초 (경합이 없을 때) | 15~25분 |
| 스키마 변경 | writer 교체 + Registry | 레코드마다 확인 후 별도 커밋 | `ALTER TABLE` + mergeSchema |
| Exactly-once | Coordinator + control topic | at-least-once + upsert 멱등성 | at-least-once + 파티션 덮어쓰기 |

</div>

> **정리**
>
> 토스 방식은 "우리 커넥터의 writer 핵심부 + 단일 committer + 정합성 보완 3가지"로 볼 수 있습니다. writer 핵심부는 이미 같으므로, 우리에게 빠진 것은 커밋을 묶고 한 곳에서 하는 구조와, 그 구조로 바꿀 때 생기는 정합성 구멍을 막는 장치입니다.
{: .callout .info}

## 3. 토스의 세 가지 이슈와 우리 코드 {#issues}

<div class="findings" markdown="1">

<div class="finding" markdown="1">

### <span class="pill ok">충족</span> <span class="id">이슈 1</span> Kafka 키 = 테이블 id

connectors/mdp-repo-log-collector · MongoEventToRecordConverter.java:164-182
{: .where}

- Debezium MongoDB 커넥터는 문서의 `_id`를 Kafka 메시지 키로 씁니다. `RegexRouter`는 토픽 이름만 바꾸고 키는 그대로 둡니다. 그래서 같은 문서는 항상 같은 파티션으로 갑니다.
- 커넥터는 테이블 id 컬럼을 Kafka 키에서 꺼내므로(`id.prefix` + 키 필드), 토스가 요구한 조건을 이미 만족합니다.

> **주의: 같은 mirror 테이블을 두 토픽에서 씀**
>
> `repo_cdc_mirror_{product,…}`는 cdc 토픽(upsert-major 커넥터)과 snapshot 토픽(snapshot 커넥터) 양쪽에서 씁니다. 지금은 커넥터마다 따로 커밋해서 순서 문제만 있습니다. 하지만 coordinator 방식에서 두 토픽을 한 커넥터로 합치면, 같은 id가 다른 Worker를 거쳐 같은 커밋에 들어가 **이슈 1과 똑같은 중복**이 생깁니다. 토픽이 달라도 같은 테이블에 쓰면 이슈 1에 해당합니다.
{: .callout .warn}

</div>

<div class="finding" markdown="1">

### <span class="pill info">현재 무관</span> <span class="id">이슈 2</span> 커밋 타임아웃 시 파일 섞임

우리 커넥터에는 coordinator가 없고, 각 task가 `put()` 안에서 동기적으로 커밋합니다. 커밋에 실패하면 예외로 task가 멈추고, 파일은 버려지고, offset부터 다시 처리합니다. 다른 커밋에 파일이 섞일 경로가 없습니다. coordinator를 도입하는 2단계(6장)에서 해당됩니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">1단계부터 해당</span> <span class="id">이슈 3</span> 스키마 변경 시 writer 교체

IcebergUtil.java:52-62, 92-93 · IcebergDataWriter.java:28-95 · DeltaTaskWriter.java:83-97
{: .where}

- 지금은 `addRowsToTable()`이 먼저 스키마 변경을 모두 커밋한 다음, 새 writer를 하나 만들어 배치 전체를 쓰고 바로 커밋합니다. writer와 맵이 커밋과 같은 수명이라 문제가 없습니다.
- 커밋 묶기를 적용하면 writer가 여러 `put()`에 걸쳐 살아 있게 됩니다. 그러면 중간에 새 필드가 들어올 때 writer를 바꿔야 하고, 새 `RowDataDeltaWriter`의 `insertedRowMap`은 비어 있습니다. 같은 커밋 안에서 이전 writer가 쓴 id가 다시 오면 equality delete가 되어, **토스와 똑같은 중복**이 생깁니다.
- `insertedRowMap`은 Iceberg의 `BaseTaskWriter.BaseEqualityDeltaWriter` 안에 private으로 있습니다. 그래서 상속만으로는 밖으로 꺼낼 수 없고, 이 클래스를 우리 코드로 옮겨 와 고쳐야 합니다(5장).

</div>

</div>

## 4. 토스 방식 구현 경로 {#path}

<div class="table-wrap" markdown="1">

| 경로 | 내용 | 장점 | 부담 |
| --- | --- | --- | --- |
| **A. 자체 커넥터 1단계** (권장 시작점) | task별로 커밋을 일정 간격으로 묶음 + `insertedRowMap` 레지스트리 | 코드 변경 범위가 작음. Mongo 변환 로직 그대로 사용. 커밋이 몇 초마다에서 5분마다로 줄어듦 | 커밋 주체는 여전히 task 수만큼 (3개면 5분에 3회, 경합은 미미) |
| B. 자체 커넥터 2단계 | A + control topic 기반 coordinator | 토스와 같은 구조. 테이블당 커밋 1회, exactly-once 가능 | coordinator 선출, 이벤트 프로토콜, 타임아웃 처리(이슈 2)를 직접 구현 |
| C. Databricks(Tabular) 커넥터 포크 | 토스와 같은 출발점. coordinator와 upsert가 이미 있음. 이슈 2, 3 수정 + Mongo 변환을 SMT로 이식 | coordinator가 이미 검증됨 | 저장소가 유지보수 중단 상태라 포크를 직접 관리해야 함. Mongo 변환 로직을 SMT로 다시 작성해야 함 |
| D. Apache 공식 커넥터 | coordinator는 있음 | 활발히 관리됨 | upsert(delta writer)가 없어 mirror 테이블에는 바로 쓸 수 없음. log 테이블(append)에는 적합 |

</div>

> **권장**
>
> **A로 시작하세요.** 우리 구성은 모든 컬렉션이 한 토픽(`mdp.private.cdc.accommodation`)에 있고 task가 3개라, task별로 5분마다 커밋해도 테이블당 5분에 최대 3회입니다. 앞 문서에서 본 경합(몇 초마다 최대 8개)과 비교하면 사실상 사라지는 수준입니다. 테이블당 커밋을 하나로 줄이거나 exactly-once가 필요해지면 그때 B나 C로 갑니다. 또 log 테이블은 D(공식 커넥터, append)로, mirror는 Spark `MERGE`로 나누는 조합도 유효합니다(7장).
{: .callout .ok}

## 5. 코드 변경 사항 (1단계: 커밋 묶기 + 레지스트리) {#changes}

### 변경 파일 목록

<div class="table-wrap" markdown="1">

| 파일 | 변경 내용 |
| --- | --- |
| `IcebergSinkTask.java` | `put()`은 쓰기만 하고, 커밋은 `preCommit()`에서 간격이나 건수 조건이 차면 수행. 커밋에 성공한 경우에만 offset 반환. `close()`/`stop()`에서 열린 writer abort |
| `IcebergUtil.java` | `addRecord()`를 쓰기(`write`)와 커밋(`commit`)으로 분리. 스키마 비교는 배치 단위로 한 번. 스키마가 바뀌면 writer 교체(roll) |
| **신규** `TableWriterState.java` | 테이블별 상태: 현재 writer, 교체된 writer들의 `WriteResult` 목록, 레지스트리, 파티션별 마지막 offset |
| **신규** `InsertedRowRegistry.java` | 키 → (파일 경로, 행 위치) 맵. 테이블별로 하나, **커밋할 때만** 비움 |
| **신규** `RegistryEqualityDeltaWriter.java` | Iceberg `BaseEqualityDeltaWriter`를 옮겨 와서 내부 맵 대신 레지스트리를 쓰도록 수정 |
| `DeltaTaskWriter.java` | `RowDataDeltaWriter`가 새 클래스를 상속하고 생성자로 레지스트리를 받음 |
| `IcebergDataWriter.java` | `getWriter()`가 레지스트리를 받아 delta writer에 전달 |
| `IcebergSinkConfig*.java` | `iceberg.commit.interval-ms`(기본 300000), `iceberg.commit.max-records` 추가 |

</div>

### IcebergSinkTask: 쓰기와 커밋 분리

```java
private final Map<String, TableWriterState> states = new HashMap<>();
private long lastCommitMs = System.currentTimeMillis();

@Override
public void put(Collection<SinkRecord> records) {
  // 변환과 테이블별 그룹핑은 기존과 같음
  for (var e : grouped.entrySet()) {
    TableWriterState st = states.computeIfAbsent(e.getKey(), this::openState);
    icebergUtil.write(st, e.getValue());        // 커밋하지 않음
    st.trackOffsets(e.getValue());              // TopicPartition별 최대 offset
  }
}

@Override
public Map<TopicPartition, OffsetAndMetadata> preCommit(Map<TopicPartition, OffsetAndMetadata> current) {
  if (!commitDue()) return Map.of();            // offset 커밋 보류
  Map<TopicPartition, OffsetAndMetadata> safe = new HashMap<>();
  for (TableWriterState st : states.values()) {
    icebergUtil.commit(st);                     // RowDelta 1회 + registry.clear()
    st.offsets().forEach((tp, off) -> safe.merge(tp, new OffsetAndMetadata(off + 1),
        (a, b) -> a.offset() >= b.offset() ? a : b));
  }
  states.clear();
  lastCommitMs = System.currentTimeMillis();
  return safe;
}
```

- Kafka Connect는 `offset.flush.interval.ms`(기본 60초)마다 `preCommit()`을 부릅니다. 커밋 간격은 이 값의 배수로 잡으세요.
- 한 TopicPartition의 레코드가 여러 테이블로 나뉘므로, 모든 테이블 커밋이 끝난 뒤에 offset을 반환해야 합니다. 중간에 실패하면 예외를 던져 재처리합니다. upsert라 재처리해도 결과는 같습니다.
- 파티션에 들어온 레코드가 전부 필터링되거나 변환에서 빠지면 그 파티션은 어느 테이블 상태에도 기록되지 않습니다. 이런 파티션의 offset은 Connect가 넘겨준 `current` 값으로 함께 전진시켜야 lag이 쌓이지 않습니다.
- 건수 조건(`max-records`)을 함께 두세요. 레지스트리와 Parquet 버퍼의 메모리 사용량이 커밋 간격 동안 들어온 키 수에 비례합니다.

### IcebergUtil: 스키마 변경 시 writer 교체

```java
public void write(TableWriterState st, List<IcebergChangeRecord> records) {
  if (config.isAllowFieldAddition()) {
    Schema union = unionSchemas(records);                // 배치에서 한 번만 계산
    if (applyFieldAddition(st.table(), union)) {         // 스키마 커밋이 일어난 경우
      st.table().refresh();
      st.rollWriter(IcebergDataWriter.getWriter(st.table(), config, st.registry()));
      // rollWriter: 이전 writer.complete() 결과를 pending 목록에 보관하고 새 writer로 교체
    }
  }
  for (IcebergChangeRecord r : records) {
    st.writer().write(r.asIcebergRecord(st.table().schema(), ...), r.operationCode());
  }
}

public void commit(TableWriterState st) {
  WriteResult last = st.closeWriter();
  RowDelta delta = st.table().newRowDelta();
  for (WriteResult r : st.allResults(last)) {            // 교체된 writer들 + 마지막 writer
    Arrays.stream(r.dataFiles()).forEach(delta::addRows);
    Arrays.stream(r.deleteFiles()).forEach(delta::addDeletes);
  }
  delta.commit();
  st.registry().clear();                                 // 커밋 후에만 비움
}
```

### RegistryEqualityDeltaWriter: 맵을 밖으로

Iceberg의 `BaseTaskWriter.BaseEqualityDeltaWriter`를 복사해 와서 다음 부분만 고칩니다(Iceberg 1.4.x 기준 구조).

```java
// 변경 전 (Iceberg 내부)
private Map<StructLike, PathOffset> insertedRowMap = StructLikeMap.create(deleteSchema.asStruct());

// 변경 후
private final InsertedRowRegistry registry;              // 생성자로 주입, writer 간 공유

public void write(T row) {
  PathOffset pathOffset = PathOffset.of(dataWriter.currentPath(), dataWriter.currentRows());
  StructLike copiedKey = StructCopy.copy(structProjection.wrap(asStructLike(row)));
  PathOffset previous = registry.put(copiedKey, pathOffset);
  if (previous != null) {
    posDeleteWriter.delete(previous.path, previous.rowOffset, null);   // 같은 커밋 → position delete
  }
  dataWriter.write(row);
}

private boolean internalPosDelete(StructLike key) {
  PathOffset previous = registry.remove(key);
  if (previous != null) {
    posDeleteWriter.delete(previous.path, previous.rowOffset, null);
    return true;
  }
  return false;                                          // 없으면 호출자가 equality delete
}

// close(): 기존에는 여기서 insertedRowMap을 비웠지만, 이제 registry는 비우지 않음
```

- **레지스트리는 테이블별로 하나**이고, 키 타입은 테이블의 identifier 필드(`deleteSchema`)로 만듭니다. 스키마가 바뀌어도 identifier 필드가 그대로면 같은 맵을 계속 쓸 수 있습니다. identifier 필드가 바뀌는 변경이면 먼저 커밋한 뒤 교체하세요.
- **파티션 테이블**(`PartitionedDeltaWriter`)은 파티션마다 writer가 따로 있습니다. 하지만 레지스트리 키가 id라서 테이블 단위로 하나만 두면 됩니다. 같은 id가 커밋 안에서 다른 파티션으로 옮겨 가도(예: `__p_time` 변경) 이전 위치를 position delete로 지울 수 있습니다. 지금 코드는 파티션별로 맵이 따로라 이 경우를 놓치므로, 오히려 더 정확해집니다.
- 이전 writer가 닫은 파일을 가리키는 position delete도 유효합니다. 같은 커밋에 들어가면 sequence number가 같아서 "같거나 작은" 규칙에 맞습니다.

### 함께 정리할 것

- **major mirror 테이블의 이중 writer**: snapshot 커넥터와 upsert-major 커넥터를 시간으로 분리하거나, snapshot은 별도 테이블로 적재한 뒤 Spark `MERGE`로 반영합니다.
- **`__source_ts_ms_ord` 인코딩**(Buffer Writer 리뷰 S3): 같은 계산이 이 커넥터에도 있습니다(`MongoEventToRecordConverter.java:249`).
- **실패 시 writer abort**: 커밋 전에 예외가 나면 열린 writer와 교체된 writer의 파일을 모두 `abort()`해서 고아 파일을 남기지 않습니다.
- **컴팩션**: 커밋을 묶어도 equality delete는 계속 생깁니다. 「커밋 경합 리뷰」 7장의 증분 컴팩션을 함께 운영해야 합니다. 토스 글 댓글에서도 컴팩션 비용이 미해결 질문으로 남아 있습니다.

## 6. 코드 변경 사항 (2단계: coordinator) {#coord}

테이블당 커밋을 하나로 만들고 exactly-once가 필요할 때 진행합니다. Tabular 커넥터의 구조를 따르면 다음 요소가 필요합니다.

<div class="table-wrap" markdown="1">

| 요소 | 내용 |
| --- | --- |
| control topic | coordinator와 worker가 주고받는 이벤트용 토픽: `StartCommit`, `DataWritten`(파일 목록), `DataComplete`(offset), `CommitComplete` |
| coordinator 선출 | 특정 파티션(예: 첫 토픽의 0번 파티션)을 할당받은 task가 coordinator 역할을 겸함. 리밸런싱 때 넘겨받음 |
| worker 변경 | 1단계의 `preCommit()` 커밋 대신, `StartCommit`을 받으면 writer를 닫고 `WriteResult`를 control topic으로 보냄. 레지스트리는 이 시점에 비움 |
| coordinator 커밋 | 모든 worker의 응답을 모아 테이블당 `RowDelta` 1회. 스냅샷 summary에 커밋한 offset을 기록해서, 재시작할 때 중복 커밋을 막음 |
| **토스 이슈 2 대응** | 타임아웃으로 다음 라운드에 늦게 온 파일은, 새 라운드 커밋 **전에 별도 `RowDelta`로 먼저 커밋**(토스의 preCommit). 두 라운드의 파일이 한 스냅샷에 섞이지 않게 해서 sequence number 순서를 지킴 |
| offset 커밋 | worker는 coordinator가 커밋을 완료한 offset만 Kafka에 커밋(트랜잭션 producer 또는 `preCommit` 반환값) |

</div>

이 부분은 직접 만들기보다 Databricks(Tabular) 커넥터 코드의 coordinator, worker, 커밋 상태 관리 클래스를 참고하거나 포크하는 편이 빠릅니다. 토스도 이 커넥터를 기반으로 수정했습니다. 저장소가 유지보수 중단 상태라는 점은 감안해야 합니다.

## 7. 배치 방식과의 선택 기준 {#choose}

<div class="table-wrap" markdown="1">

| 요구사항 | 적합한 방식 |
| --- | --- |
| mirror 최신성 5~10분 이내 | 커넥터 1단계 (토스 방식). equality delete가 생기므로 컴팩션 운영 필수 |
| 최신성 15~30분 허용, 읽기 성능과 단순한 테이블 상태가 중요 | buffer + Spark 배치. 「Buffer Writer 배치 적재 리뷰」의 넘겨받기와 정합성 문제를 먼저 수정 |
| 이력 테이블(log)만 실시간, mirror는 주기적 | log는 Apache 공식 커넥터(append, exactly-once), mirror는 log에서 Spark `MERGE`. 버퍼 S3 계층과 윈도우 넘겨받기 문제가 사라짐 |
| 테이블당 커밋 1회, exactly-once, 다수 토픽 통합 | 커넥터 2단계 또는 Tabular 포크 |

</div>

> **확인이 필요한 전제**
>
> source가 AWS DocumentDB(`mdp-docdb5-prod…docdb.amazonaws.com`)입니다. 토스 글과 이 문서의 분석은 Debezium MongoDB 커넥터의 키와 순서 의미를 전제로 합니다. DocumentDB change stream에서 `source.ord`와 키 형식이 MongoDB와 같은지 실제 데이터로 한 번 확인하세요.
{: .callout .warn}

## 8. 성과와 기대 효과 {#outcome}

### 기존 구현의 성과

외부 사례와 나란히 놓고 보니, 기존 구현이 이미 갖추고 있던 것이 분명해졌습니다.

<div class="table-wrap" markdown="1">

| 구분 | 성과 |
| --- | --- |
| 정합성 | 토스가 운영 중에 겪은 세 가지 중복 문제가 지금 구조에서는 생기지 않음. Kafka 키가 테이블 id와 같고(이슈 1), 커밋이 `put()` 단위라 writer와 맵의 수명이 커밋과 같음(이슈 2, 3) |
| writer 설계 | writer 핵심부(`BaseEqualityDeltaWriter`)가 토스가 쓴 커넥터와 같은 구조. 같은 커밋 안의 중복은 position delete, 이전 커밋의 행은 equality delete로 지우는 규칙을 이미 따름 |
| 구조 전환 | buffer + Spark 배치로 바꾸면서 커밋 경합과 equality delete를 함께 없앰. 토스와는 다른 갈래로 같은 문제를 푼 셈 |
| 분석 | 다음 단계로 제안한 "커밋 묶기"가 이슈 3을 새로 만든다는 점을 **적용 전에** 찾아, 회귀를 막을 수정 지점(레지스트리)을 미리 정함 |

</div>

### 개선을 적용하면 얻는 것

자체 커넥터에 토스 방식을 적용할 때의 예상입니다. 수치는 운영 설정(task 3개, 토픽 1개, 커밋 간격 5분)으로 계산했습니다.

<div class="table-wrap" markdown="1">

| 개선 | 지표 | 지금 | 적용 후 (예상) | 근거 |
| --- | --- | --- | --- | --- |
| 1단계: 커밋 묶기 | 테이블당 커밋 | 몇 초마다, 최대 8개 writer | <span class="pill ok">5분에 최대 3회</span> 경합은 사실상 사라짐 | 4장 |
| 1단계: 레지스트리 | 같은 커밋 안의 중복 행 | 0. 단, 레지스트리 없이 커밋만 묶으면 스키마 변경 때 발생 | <span class="pill ok">커밋을 묶어도 0</span> | 3장, 5장 |
| 1단계: 레지스트리 | 커밋 안에서 파티션을 옮긴 id | 이전 위치를 지우지 못함 | <span class="pill ok">position delete로 정확히 삭제</span> | 5장 |
| 1단계: 쓰기·커밋 분리 | 실패 시 고아 파일 | 남음 | <span class="pill ok">abort로 정리</span> | 5장 |
| 2단계: coordinator | 테이블당 커밋 | 5분에 최대 3회 | <span class="pill ok">1회</span> | 6장 |
| 2단계: coordinator | 전달 보장 | at-least-once + upsert 멱등성 | <span class="pill ok">exactly-once</span> 스냅샷에 offset 기록 | 6장 |
| 커밋 간격 | mirror 최신성 | 수 초 (경합이 없을 때) | <span class="pill warn">5분 이내</span> 배치 방식(15~25분)보다는 빠름 | 7장 |

</div>

> **측정 방법**
>
> 같은 커밋 안의 중복은 mirror 테이블에서 `GROUP BY id HAVING count(*) > 1`로 바로 확인할 수 있습니다. 1단계를 적용한 뒤 스키마 변경이 있었던 날을 골라 이 쿼리가 0건인지 보면, 레지스트리가 제대로 동작하는지 검증됩니다.
{: .callout .info}

* * *

## 9. 참고 {#refs}

- [토스증권 Iceberg 적용기 #1: CDC 환경은 왜 제대로 동작하지 않을까? (toss.tech, 2025-09-10)](https://toss.tech/article/iceberg-cdc-1)
- [Apache Iceberg Spec: Row-level Deletes (sequence number 적용 규칙)](https://iceberg.apache.org/spec/#row-level-deletes)
- [databricks/iceberg-kafka-connect](https://github.com/databricks/iceberg-kafka-connect) (토스가 사용한 커넥터, 구 tabular-io, 유지보수 중단)
- [apache/iceberg: kafka-connect](https://github.com/apache/iceberg/tree/main/kafka-connect) · [문서](https://iceberg.apache.org/docs/latest/kafka-connect/)
- 연관 문서: [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html) · [Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html)
