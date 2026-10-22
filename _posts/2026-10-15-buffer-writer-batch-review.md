---
layout: post
title: Buffer Writer 배치 적재 리뷰
tags: [Iceberg, Kafka Connect, Spark]
author: caimask
---

Kafka Connect upsert 커넥터의 커밋 경합을 피하려고 전환한 구조, 즉 이벤트를 10분 단위 Parquet 파일로 S3에 버퍼링하고 Spark 배치로 Iceberg log/mirror 테이블에 적재하는 방식을 검토한 문서입니다. 「Iceberg 커밋 경합 리뷰」 문서의 후속편입니다.

작성 2026-10-15 · buffer_writer v1.0.1 · Parquet 1.13.1 · mdp-batch-writer · PySpark 3.5.0 · Kyuubi
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **방향은 맞습니다.** 테이블당 writer가 배치 job 하나로 줄었고, 커밋은 10분에 테이블당 2~3회로 줄었습니다. equality delete도 사라졌습니다. 앞 문서에서 권장한 "append 먼저, merge는 나중에" 패턴과 같은 구조입니다.
2. **가장 큰 위험은 두 컴포넌트가 넘겨받는 지점에 있습니다.** 배치는 벽시계 시간이 지나면 윈도우가 끝났다고 보고 처리한 뒤 checkpoint를 넘깁니다. 하지만 버퍼 파일은 "다음 윈도우의 레코드가 도착해야" 완성되고, 지연된 이벤트는 이미 처리된 과거 윈도우 경로에 나중에 쓰입니다. 이렇게 늦게 도착한 파일은 **다시 처리되지 않아 Iceberg에서 조용히 누락**될 수 있습니다.
3. **정합성 버그가 세 가지 있습니다.**
   - mirror `MERGE`의 원본을 고르는 쿼리가 컬럼별 `max`를 따로 구해서, 변경이 반영되지 않고 빠질 수 있습니다.
   - `__source_ts_ms_ord = ts_ms + ord`는 같은 초 안에서 ord가 1000을 넘으면 순서가 뒤바뀌거나 값이 겹칩니다. Java 커넥터에도 같은 계산이 있습니다.
   - 버퍼 writer는 타임스탬프 역전이 있으면 아직 S3에 올리지 않은 레코드의 offset까지 커밋합니다.
4. **시간이 갈수록 느려지는 구조가 있습니다.**
   - mirror `MERGE`가 log 테이블 전체를 스캔합니다.
   - mirror가 파티션 없는 copy-on-write 테이블이라, 바뀐 행이 든 파일을 매번 다시 씁니다.
   - log 테이블 파티션이 10분 단위라 1년이면 5만 개를 넘습니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [전환된 구조](#flow)
2. [전환 방향 평가](#eval)
3. [핵심 위험: 윈도우 완료 판단](#handoff)
4. [buffer_writer 검토](#buffer)
5. [mdp-batch-writer 검토](#batch)
6. [개선안과 코드 예시](#fix)
7. [우선순위](#plan)
8. [성과와 기대 효과](#outcome)
9. [참고](#refs)

</div>

## 1. 전환된 구조 {#flow}

```
Kafka mdp.private.cdc.accommodation
  │  BufferWriterConnector (prod: tasks 5, rotate.mode=record, rotate.interval.ms=600000)
  │  Kafka 레코드 timestamp 기준 10분 윈도우 × Kafka 파티션마다 파일 1개
  ▼
S3 prod/data/EventBuffer/{topic}/{yyyyMMddHHmm}/{topic}+{partition}+{windowStartMs}.gz.parquet
  │  CronJob "5/10 * * * *" → Kyuubi spark-submit → batch_monitor 폴링
  ▼
main.py  checkpoint 읽기 → 다음 윈도우 경로 읽기 → Python RDD 변환 → 테이블별 병렬 처리
  ├─ log 테이블:    overwritePartitions(__p_id = 윈도우)          커밋 1
  └─ mirror 테이블: MERGE INTO … USING (log 중 윈도우의 키별 최신)  커밋 1
  → checkpoint 쓰기 (S3 json)
```

운영 설정 기준입니다. dev 기본값은 1시간 윈도우(`%Y%m%d%H`, `3600000`)이고, prod 버퍼 커넥터는 10분(`YYYYMMddHHmm`, `600000`)입니다. 배치의 prod 값은 별도 저장소(`ckp-yanolja-mdp-eks-manifest`)에 있어 확인하지 못했습니다. 두 설정의 윈도우 단위와 포맷이 반드시 같아야 합니다.

## 2. 전환 방향 평가 {#eval}

<div class="table-wrap" markdown="1">

| 항목 | 이전: Iceberg upsert 커넥터 | 현재: buffer + Spark 배치 |
| --- | --- | --- |
| 테이블당 커밋 주체 | 최대 8개 task | <span class="pill ok">1</span> 배치 job (CronJob `Forbid`) |
| 커밋 빈도 | 몇 초마다 | <span class="pill ok">10분마다</span> 테이블당 2~3회 |
| 삭제 방식 | equality delete | <span class="pill ok">없음</span> log는 append, mirror는 CoW `MERGE` |
| 재처리 멱등성 | upsert에 의존 | <span class="pill ok">양호</span> 파일명 고정 + OVERWRITE, log는 파티션 단위 덮어쓰기 |
| 최신성 | 수 초 | <span class="pill warn">15~25분</span> 윈도우 10분 + 대기 5분 + 실행 시간 |
| 데이터 완전성 | offset 커밋으로 보장 | <span class="pill bad">취약</span> 윈도우 완료를 벽시계로 판단 (3장) |
| 운영 구성요소 | 커넥터 1종 | <span class="pill warn">증가</span> 커넥터, S3 버퍼, CronJob, Kyuubi, checkpoint 파일 |

</div>

### 잘 된 점

- **버퍼 파일명이 결정적입니다.** `topic+partition+windowStartMs`를 OVERWRITE 모드로 쓰므로, 재처리해도 같은 파일을 같은 내용으로 덮어씁니다.
- **offset은 파일이 S3에 올라간 뒤에만 커밋합니다.** `preCommit()`에서 완료된 파일 기준 offset만 반환하므로 기본적으로 at-least-once입니다. 예외는 4장의 B2입니다.
- **업로드 중 실패하거나 종료되면 multipart upload를 abort해서** 반쪽짜리 파일이 남지 않습니다.
- **log 테이블을 `overwritePartitions`로 써서** 같은 윈도우를 다시 돌려도 중복이 생기지 않습니다.
- **checkpoint보다 뒤처졌으면 루프를 돌며 따라잡고,** 백필 경로도 분리되어 있습니다.
- **Debezium `capture.mode=change_streams_update_full`이라 update 이벤트의 `after`가 전체 문서입니다.** 그래서 mirror에서 모든 컬럼을 `UPDATE SET`하는 방식이 맞습니다. log-collector 운영 설정에서 확인했습니다.

## 3. 핵심 위험: 윈도우 완료 판단 {#handoff}

두 컴포넌트가 "윈도우가 끝났다"를 서로 다른 기준으로 판단합니다.

<div class="table-wrap" markdown="1">

| 컴포넌트 | 윈도우 W 파일을 완성하거나 처리하는 시점 |
| --- | --- |
| buffer_writer (`record` 모드) | 그 Kafka 파티션에 **W 다음 윈도우의 레코드가 도착했을 때** 파일을 닫고 업로드 |
| mdp-batch-writer | **벽시계가 W 종료 + 약 5분을 지났을 때** 경로를 읽고 checkpoint를 W로 전진 |

</div>

```
10:10~10:20 윈도우(W)
  partition 0: 10:21 레코드 도착 → W 파일 업로드 (10:21)
  partition 3: 트래픽 없음 … 10:47 레코드 도착 → W 파일 업로드 (10:47)
배치: 10:25 실행 → W 경로 읽기 (partition 0 파일만 있음) → checkpoint = W
      10:35 실행 → W+1 처리 …  partition 3의 W 파일은 다시 읽히지 않음
```

같은 일이 다음 상황에서도 생깁니다.

- **버퍼 커넥터의 지연이나 재시작**: 컨슈머 lag이 쌓이면, 과거 timestamp의 레코드가 이미 처리된 윈도우 경로에 나중에 써집니다.
- **버퍼 커넥터 장애**: 해당 시간대 경로가 없으면 `runBatchJob`이 "Partition is not Exist"로 건너뛰고, checkpoint는 그대로 전진합니다(`main.py:93-97`).
- **Kafka 파티션 간 진행 차이**: 한 파티션만 파일을 올려도 경로가 존재하므로, 배치는 윈도우를 완료로 간주합니다.

> **영향**
>
> 누락이 생겨도 오류가 나지 않습니다. 버퍼 S3에는 파일이 있지만 Iceberg log/mirror에는 없는 상태가 됩니다. mirror는 그 키의 다음 변경이 올 때까지 이전 상태로 남고, delete 이벤트가 누락되면 지워진 문서가 계속 남습니다.
{: .callout .bad}

### 누락 여부 확인 방법

```sql
-- 1) 버퍼 원본 건수 (윈도우 하나)
SELECT count(*) FROM parquet.`s3a://yanolja-mdp-prod-bucket/prod/data/EventBuffer/mdp.private.cdc.accommodation/202610150910/`;

-- 2) 같은 윈도우의 log 테이블 건수 (모든 컬렉션 테이블 합계)
SELECT count(*) FROM yanolja_mdp_prod.<prefix>_log_<db>_<collection> WHERE __p_id = '202610150910';
```

S3에서 파일의 `LastModified`가 "해당 윈도우를 배치가 처리한 시각"보다 늦은 파일을 찾으면, 실제로 누락된 파일 목록을 바로 얻을 수 있습니다.

## 4. buffer_writer 검토 {#buffer}

<div class="findings" markdown="1">

<div class="finding" markdown="1">

### <span class="pill bad">높음</span> <span class="id">B1</span> record 모드는 쉬고 있는 파티션의 파일을 닫지 않습니다

writer/TopicPartitionWriter.java:231-258
{: .where}

`writeByRecordTsMs()`는 다음 윈도우의 레코드가 와야만 `commitFile()`을 호출합니다. 트래픽이 끊긴 파티션의 마지막 윈도우 파일은 무기한 열려 있고, 그동안 offset도 커밋되지 않습니다. 3장 누락 문제의 직접 원인입니다. `fusion` 모드에는 빈 `put()` 호출 때 오래된 윈도우를 닫는 로직이 있지만, 운영 설정은 `record`입니다.

</div>

<div class="finding" markdown="1">

### <span class="pill bad">높음</span> <span class="id">B2</span> 타임스탬프 역전 시 업로드 전 레코드까지 offset이 커밋됩니다

writer/TopicPartitionWriter.java:131-165, 260-273
{: .where}

`writeBuffer()`는 레코드마다 현재 윈도우를 그 레코드의 timestamp로 다시 계산합니다. 그래서 timestamp가 이전 윈도우로 돌아간 레코드가 오면 윈도우별 writer가 동시에 여러 개 열립니다. 이 상태에서 `commitFile()`은 한 윈도우의 파일만 닫으면서 `offsetToCommit = currentOffset + 1`로 설정합니다. 이 값은 아직 열려 있는 다른 윈도우 writer에 들어 있는 레코드의 offset보다 클 수 있습니다. 그 뒤 재시작하거나 리밸런싱되면 열린 writer는 abort되고, offset은 이미 넘어갔으므로 그 레코드들은 유실됩니다.

Kafka 레코드 timestamp는 producer가 정하는 CreateTime이라, 같은 파티션 안에서도 역전이 생길 수 있습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">B3</span> `close()`가 아직 할당된 파티션의 writer까지 지웁니다

BufferWriterTask.java:112-122
{: .where}

회수된 파티션만 닫은 뒤 `topicPartitionWriters.clear()`로 전체를 비웁니다. cooperative 리밸런싱처럼 일부 파티션만 회수되는 경우, 남은 파티션의 `put()`에서 `NullPointerException`이 납니다. 지금은 eager 방식이라 드러나지 않을 수 있지만, 회수된 파티션만 제거하도록 바꾸는 것이 안전합니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">B4</span> 운영 설정이 `auto.offset.reset=latest`입니다

kafka/connectors/mdp-repo-log-buffer-sink/config.prod.json
{: .where}

커넥터를 오래 멈춰 컨슈머 그룹 offset이 만료되거나(기본 보존 7일), 커넥터 이름을 바꿔 새 그룹이 되면 그 사이 데이터를 건너뜁니다. 버퍼는 원본 보존 계층이므로 `earliest`가 맞습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">B5</span> 열린 writer마다 메모리를 크게 씁니다

writer/ParquetRecordWriter.java:37-47, writer/S3OutputStream.java:44
{: .where}

writer 하나가 Parquet row group(기본 128MB까지)과 S3 part 버퍼(5MB 이상)를 메모리에 들고 있습니다. 메모리 사용량은 task 수 × 파티션 수 × 열린 윈도우 수만큼 늘어납니다. B1, B2로 열린 writer가 쌓이면 worker heap을 압박합니다. row group 크기를 파일 크기에 맞게 줄이는 것을 권장합니다.

</div>

<div class="finding" markdown="1">

### <span class="pill info">낮음</span> <span class="id">B6</span> 그 밖의 개선점

- GZIP 압축은 CPU 비용이 큽니다. `ZSTD`나 `SNAPPY`를 권장합니다. 지금은 파일 확장자가 GZIP으로 고정되어 있어 함께 바꿔야 합니다.
- `start()`에서 `open(context.assignment())`를 직접 호출하고, 프레임워크도 `open()`을 다시 호출합니다. 지금은 해가 없지만 중복입니다.
- README는 SimpleDateFormat 패턴을 안내하지만 코드는 Joda-Time입니다. `Y`의 의미가 서로 다르므로(SimpleDateFormat에서는 week-year, Joda에서는 year-of-era) `yyyy`로 통일하세요.
- 버퍼 레코드에 Kafka partition과 offset이 없습니다. 이 둘을 넣으면 하류에서 정확한 중복 제거 키로 쓸 수 있습니다(6장).

</div>

</div>

## 5. mdp-batch-writer 검토 {#batch}

<div class="findings" markdown="1">

<div class="finding" markdown="1">

### <span class="pill bad">높음</span> <span class="id">S1</span> 없는 윈도우를 건너뛰고 checkpoint를 전진시킵니다

main.py:69-91, 93-97
{: .where}

3장의 배치 쪽 원인입니다. 경로가 없으면 처리하지 않지만 checkpoint는 그 윈도우로 넘어가고, 늦게 생긴 파일은 다시 보지 않습니다. 경로가 있어도 모든 Kafka 파티션의 파일이 모였는지는 확인하지 않습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill bad">높음</span> <span class="id">S2</span> MERGE 원본 쿼리가 변경을 빠뜨리고, log 전체를 스캔합니다

src/spark_client.py:117-126
{: .where}

```
WHERE (id_id, __source_ts_ms_ord, __kafka_ts_ms, __event_ts_ms) IN (
  SELECT id_id, max(__source_ts_ms_ord), max(__kafka_ts_ms), max(__event_ts_ms)
  FROM log WHERE __p_id = '{partition}' GROUP BY id_id)
```

- **변경 누락**: 세 컬럼의 `max`를 따로 구하므로, 세 값이 한 행에서 나온다는 보장이 없습니다. 예를 들어 재전송된 이벤트처럼 `__source_ts_ms_ord`가 가장 큰 행이 `__kafka_ts_ms`는 가장 크지 않으면, 조건에 맞는 행이 없어 그 키의 변경이 mirror에 반영되지 않습니다.
- **전체 스캔**: 바깥 `SELECT DISTINCT * FROM log`에 `__p_id` 조건이 없어 log 테이블 전체를 읽습니다. log가 쌓일수록 실행 시간이 계속 늘어납니다.
- **중복 매칭**: 다른 윈도우에 같은 튜플을 가진 행이 있고 다른 컬럼 값이 조금이라도 다르면, 원본 행 여러 개가 같은 대상에 매칭되어 MERGE가 실패합니다.

</div>

<div class="finding" markdown="1">

### <span class="pill bad">높음</span> <span class="id">S3</span> `__source_ts_ms_ord = ts_ms + ord`가 순서와 유일성을 깨뜨립니다

src/record_converter.py:78 · iceberg_connector/…/MongoEventToRecordConverter.java:249
{: .where}

Debezium MongoDB 커넥터의 `source.ts_ms`는 oplog 타임스탬프의 초 값을 ms로 바꾼 것이라 항상 1000의 배수이고, `ord`는 같은 초 안의 증가 번호입니다. 두 값을 더하면 ord가 1000 이상일 때 다음 초의 값과 겹치거나 순서가 뒤집힙니다.

```
초 S, ord 1500  → S*1000 + 1500 = (S+1)*1000 + 500
초 S+1, ord 3   → (S+1)*1000 + 3          ← 실제로는 더 나중인데 값이 더 작음
```

bulk 작업이나 snapshot처럼 초당 1000건이 넘는 변경이 있으면 실제로 발생합니다. 이 값은 log 테이블 키, mirror 최신 판정, 앞 문서에서 이력 보존의 근거로 삼은 키에 모두 쓰입니다. 버퍼 데이터에서 `max(source.ord)`를 조회하면 실제로 1000을 넘는 경우가 있는지 바로 확인할 수 있습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill bad">높음</span> <span class="id">S4</span> 늦게 들어온 과거 변경이 mirror의 최신 상태를 덮어씁니다

src/spark_client.py:118-125, LOG_TABLE_ONLY_COLUMN
{: .where}

`WHEN MATCHED` 절에 버전 비교가 없습니다. 백필이나 3장의 지연 데이터로 과거 윈도우를 나중에 처리하면, 오래된 상태가 최신 상태를 덮어씁니다. mirror 테이블은 `__source_ts_ms_ord`를 저장하지 않으므로(log 전용 컬럼) 지금 구조로는 비교할 방법도 없습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">S5</span> Python 타입으로 스키마를 추론해서 타입이 섞이면 실패합니다

src/spark_client.py:45-63, 79-81
{: .where}

- 값이 `None`이면 `StringType`으로 잡힙니다. 그래서 같은 필드가 어떤 문서에서는 null, 어떤 문서에서는 int이면 `(f, String)`과 `(f, Long)`이 둘 다 set에 들어가 같은 이름의 컬럼이 두 개 생깁니다.
- 테이블이 이미 있으면 기존 스키마로 `createDataFrame`을 호출합니다. 이때 타입이 다른 값이 오면 "LongType can not accept object 'abc'" 같은 오류로 job 전체가 실패합니다.
- MongoDB는 스키마가 없어서 이런 타입 혼재가 흔합니다. 한 컬렉션의 문제로 같은 실행의 모든 테이블이 멈춥니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">S6</span> mirror CoW MERGE의 비용이 테이블 크기에 비례해 커집니다

mirror 테이블 생성 (파티션 없음, 기본 write.merge.mode = copy-on-write)
{: .where}

10분마다 바뀐 `id_id`가 들어 있는 data 파일을 통째로 다시 씁니다. 키가 테이블 전체에 흩어져 있으면 거의 전체를 재작성하게 됩니다. 앞 문서 6장의 쓰기 증폭 문제입니다. merge-on-read(v2 position delete, v3 deletion vector)와 주기적 컴팩션, 또는 `bucket(id_id)` 파티션을 검토하세요.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">S7</span> log 테이블 파티션이 10분 단위입니다

src/spark_client.py:84-88
{: .where}

`partitionedBy(__p_id)`는 연간 약 52,000개 파티션을 만들고, 파티션마다 작은 파일이 생깁니다. 쿼리 계획과 컴팩션 비용이 커집니다. `days(__p_time)`로 파티셔닝하고, 윈도우 단위 멱등성은 `overwrite(__p_id = …)` 조건 덮어쓰기로 유지하는 것을 권장합니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">S8</span> 배치가 겹쳐 실행될 수 있습니다

helm_chart/templates/cronjob.yaml, src/batch_monitor.py
{: .where}

`concurrencyPolicy: Forbid`는 제출용 pod만 보호하고, Spark 앱 자체는 막지 못합니다.

- `batch_monitor.py`는 Kyuubi API 호출이 한 번만 실패해도 예외로 종료됩니다.
- `restartPolicy: OnFailure`라서 pod가 재시작되면 `submit_batch_job.sh`를 다시 실행해 Spark 앱을 또 제출합니다.
- 그러면 같은 checkpoint를 읽는 driver 두 개가 같은 테이블에 MERGE하게 됩니다. 앞 문서의 커밋 경합이 다시 생기고, CoW MERGE 검증 충돌로 실패할 수 있습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">S9</span> 토픽이 여러 개면 일부만 처리합니다

main.py:75-91
{: .where}

- 토픽 하나가 최신이면 `return False`로 `main()` 전체를 끝내서, 뒤 순서의 토픽은 처리되지 않습니다.
- `latest_checkpoint`는 반복마다 새 dict로 만들어서, 이번에 처리한 토픽만 기록하고 나머지 토픽의 checkpoint를 지웁니다.
- 지금은 토픽이 하나라 문제가 드러나지 않습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill info">낮음</span> <span class="id">S10</span> 그 밖의 개선점

- 백필 실행 시 `BACKFILL_END_PARTITION`이 없으면 정의되지 않은 `latest_partition`을 참조합니다(`main.py:45`). 제출 스크립트가 빈 문자열을 넘기므로 `int('')` 오류도 날 수 있습니다.
- `buffer_df.count()`는 로그 출력만을 위해 버퍼 전체를 한 번 더 읽습니다.
- Python RDD `map`으로 레코드를 변환하면 JVM과 Python 사이 직렬화 비용이 큽니다. 처리량이 늘면 DataFrame 함수(`from_json` 등)나 Scala UDF로 옮기는 것을 검토하세요.
- `getLastPartition()`은 실제로는 가장 오래된 파티션을 돌려줍니다. 이름을 바꾸고, `list_objects` 페이지네이션(1000개 제한)도 처리하세요.
- Kyuubi 계정과 비밀번호가 `values.yaml`에 평문(`admin/admin`)으로 있습니다. Secret으로 옮기세요.
- README는 `write.*.mode=copy-on-write`를 설정한다고 하지만 코드에는 없습니다. 기본값이 CoW라 동작은 같지만, 명시하는 편이 좋습니다.

</div>

</div>

## 6. 개선안과 코드 예시 {#fix}

### 윈도우 단위 처리에서 파일 단위 증분 처리로

3장 문제를 근본적으로 없애려면, 배치가 "어느 윈도우까지 처리했는가"가 아니라 **"어느 파일까지 처리했는가"**를 기록해야 합니다.

1. **버퍼 레코드에 `kafka_partition`, `kafka_offset`을 추가합니다.** (topic, partition, offset)은 Kafka 레코드의 유일 키입니다.
2. **배치는 지난 실행 이후 새로 생기거나 바뀐 파일을 처리합니다.** S3 `LastModified`를 기준으로 하거나, S3 이벤트를 SQS로 받습니다. 윈도우 경로와 상관없이 늦게 온 파일도 처리됩니다.
3. **log 테이블 쓰기는 (topic, partition, offset) 기준 `MERGE … WHEN NOT MATCHED THEN INSERT`로 멱등하게 합니다.** 같은 파일을 다시 처리해도 중복이 생기지 않습니다.
4. **mirror `MERGE`는 버전 비교 조건(아래)으로 늦게 온 과거 변경을 무시합니다.** 처리 순서가 결과에 영향을 주지 않게 됩니다.

윈도우 방식을 유지해야 한다면 최소한 다음이 필요합니다.

- buffer_writer가 쉬고 있는 파티션의 파일을 시간 기준으로 닫도록 합니다(`fusion` 모드의 빈 배치 처리를 `record` 모드에도 적용).
- 배치가 윈도우 W를 처리하기 전에, 모든 파티션의 버퍼 커넥터 커밋 offset이 W 종료 이후 레코드까지 진행됐는지 확인합니다.
- 경로가 없을 때 checkpoint를 넘기지 말고 경보를 냅니다.

### B2: offset은 열린 writer 중 가장 작은 offset 직전까지만

```java
// ParquetRecordWriter: 처음 쓴 레코드의 offset 보관
private long firstOffset = -1L;
public void write(CdcRecordValueEnvelope record, String fileName, long offset) {
  if (firstOffset < 0) firstOffset = offset;
  ...
}

// TopicPartitionWriter.commitFile(): 남은 writer가 있으면 그중 최소 offset까지만 커밋
timePartitionWriterMapper.remove(timePartition);
long safe = timePartitionWriterMapper.values().stream()
    .mapToLong(ParquetRecordWriter::getFirstOffset).min()
    .orElse(currentOffset + 1);
offsetToCommit = safe;
```

### S2, S4: 키별 최신 1건과 버전 비교

```sql
MERGE INTO {mirror} t
USING (
  SELECT * FROM (
    SELECT *, row_number() OVER (
             PARTITION BY id_id
             ORDER BY __source_ts_ms_ord DESC, __kafka_ts_ms DESC) AS rn
    FROM {log}
    WHERE __p_id = '{partition}'          -- 이번 윈도우만 읽기
  ) WHERE rn = 1
) s
ON t.id_id = s.id_id
WHEN MATCHED AND s.__op = 'delete'
     AND s.__source_ts_ms_ord >= t.__source_ts_ms_ord THEN DELETE
WHEN MATCHED AND s.__op != 'delete'
     AND s.__source_ts_ms_ord >= t.__source_ts_ms_ord THEN UPDATE SET …
WHEN NOT MATCHED AND s.__op != 'delete' THEN INSERT …
```

- mirror 테이블에 `__source_ts_ms_ord`를 남겨야 합니다. `LOG_TABLE_ONLY_COLUMN`에서 빼세요.
- 삭제된 행은 버전 정보도 함께 사라지므로, 삭제 뒤에 그보다 오래된 create가 늦게 오면 되살아날 수 있습니다. 이 경우까지 막으려면 행을 지우지 않고 삭제 표시만 남기는 soft delete(`__deleted` 플래그)로 바꿔야 합니다.

### S3: 순서 값 인코딩

```java
# Python (record_converter.py)
row_dict['__source_ts_ms_ord'] = int(source.ts_ms) * 1_000_000 + int(source.ord)

// Java (MongoEventToRecordConverter.java)
sourceTsMsOrd = sourceTsMs * 1_000_000L + sourceOrd;   // 약 1.8e18, long 범위 이내
```

인코딩을 바꾸면 기존 행과 비교할 수 없게 됩니다. 새 컬럼(예: `__source_seq`)으로 추가하고 기존 데이터를 백필한 뒤 전환하세요. 또는 `__source_ts_ms`, `__source_ord` 두 컬럼으로 나눠 정렬 키로 함께 쓰는 방법도 있습니다.

### S5: 타입 혼재 대응

- 필드별로 관측된 타입이 둘 이상이면 `StringType`으로 통일하고 값도 문자열로 변환합니다. `None`은 타입 추론에서 빼세요.
- 원본 문서 전체를 JSON 문자열 컬럼(`__doc`)으로 함께 저장하면, 승격된 컬럼의 타입 문제가 생겨도 원본을 잃지 않습니다.
- Iceberg v3의 `VARIANT` 타입은 이런 반정형 문서를 위한 기능입니다. 읽는 엔진이 지원하는지 확인한 뒤 중장기 선택지로 검토하세요.
- 한 테이블이 실패해도 다른 테이블은 계속 처리하도록 실패를 테이블 단위로 격리하고, 실패한 테이블만 다시 처리할 수 있게 checkpoint를 테이블별로 둡니다.

### S8: 중복 실행 방지

- 제출하기 전에 Kyuubi API로 같은 이름의 앱이 `RUNNING`인지 확인하고, 있으면 제출하지 않습니다.
- `batch_monitor.py`의 API 호출에 재시도를 넣고, 모니터 실패와 Spark 앱 실패를 구분합니다.
- checkpoint를 쓸 때 S3 조건부 쓰기(`If-Match` ETag)를 사용하면, 동시에 두 실행이 진행되어도 하나만 checkpoint를 갱신합니다.

## 7. 우선순위 {#plan}

<div class="steps" markdown="1">

<div class="step" markdown="1">

<span class="when">즉시 확인</span>

### 누락 점검

3장의 쿼리로 버퍼와 log 테이블 건수를 윈도우별로 비교합니다. 버퍼 데이터의 `max(source.ord)`도 조회해 S3 버그가 실제로 발생했는지 확인합니다.

</div>

<div class="step" markdown="1">

<span class="when">단기</span>

### 정합성 수정

S2 MERGE 원본 쿼리, B2 offset 커밋, S1 "경로 없으면 checkpoint 정지 + 경보", B4 `earliest`로 변경.

</div>

<div class="step" markdown="1">

<span class="when">단기</span>

### 운영 안정화

B1 쉬는 파티션 파일 닫기, S8 중복 실행 방지, S5 타입 통일, S10 Secret 이전.

</div>

<div class="step" markdown="1">

<span class="when">중기</span>

### 구조 개선

파일 단위 증분 처리, S3 순서 값 재인코딩과 백필, S4 버전 비교, S7 log 일 파티션.

</div>

<div class="step" markdown="1">

<span class="when">중장기</span>

### mirror 쓰기 비용

S6 merge-on-read 전환 + 컴팩션(앞 문서 7장), 또는 `bucket(id_id)`. 테이블 크기와 MERGE 실행 시간 추이를 보고 결정합니다.

</div>

</div>

> **대안 검토**
>
> 이 구조가 직접 해결하려는 문제, 즉 단일 committer, 주기적 커밋, exactly-once는 공식 Iceberg Kafka Connect Sink가 기본으로 제공합니다. 버퍼와 배치 두 컴포넌트를 계속 유지보수하는 비용과 비교해 보세요. log 테이블 적재를 공식 Sink로 바꾸고 mirror만 Spark `MERGE` 배치로 두면, 3장의 넘겨받기 문제와 B 계열 이슈가 함께 사라집니다. 이 경우 Sink의 upsert 지원 여부와는 무관합니다. log 테이블은 append만 하기 때문입니다.
{: .callout .info}

## 8. 성과와 기대 효과 {#outcome}

### 기존 구현의 성과

커밋 경합을 없애려고 적재 구조를 바꾼 결과입니다.

<div class="table-wrap" markdown="1">

| 구분 | 이전 (upsert 커넥터) | 전환 후 (buffer + Spark 배치) |
| --- | --- | --- |
| 테이블당 커밋 주체 | 최대 8개 task | <span class="pill ok">1</span> 배치 job |
| 커밋 빈도 | 몇 초마다 | <span class="pill ok">10분에 테이블당 2~3회</span> |
| equality delete | 레코드마다 | <span class="pill ok">0</span> log는 append, mirror는 CoW `MERGE` |
| 재처리 | upsert 멱등성에 의존 | <span class="pill ok">결정적 파일명 + 파티션 덮어쓰기</span> 같은 윈도우를 다시 돌려도 중복 없음 |
| 원본 보존 | 없음 | <span class="pill ok">S3 버퍼</span> Iceberg와 별개로 원본 이벤트를 Parquet로 보관. 백필과 누락 점검의 기준 |
| 장애 처리 | 반쪽 파일, 고아 파일 | <span class="pill ok">multipart upload abort</span> checkpoint보다 뒤처지면 따라잡기, 백필 경로 분리 |

</div>

앞 문서에서 권장한 "append 먼저, merge는 나중에" 구조를 실제로 운영에 옮겼고, 커밋 경합과 equality delete 누적이라는 두 문제를 함께 없앴습니다.

### 개선을 적용하면 얻는 것

아래 수치는 코드와 운영 설정에서 계산한 예상치입니다. 적용 후 측정값으로 바꿉니다.

<div class="table-wrap" markdown="1">

| 개선 | 지표 | 지금 | 적용 후 (예상) | 근거 |
| --- | --- | --- | --- | --- |
| 파일 단위 증분 처리 (또는 B1 + S1) | 늦게 도착한 버퍼 파일의 누락 | 오류 없이 누락 가능 | <span class="pill ok">0</span> 버퍼와 log 건수 일치 | 3장 |
| B2 offset 커밋 수정 | 타임스탬프 역전 시 유실 | 재시작·리밸런싱 때 발생 가능 | <span class="pill ok">0</span> | 4장 |
| S2 MERGE 원본 쿼리 | mirror에 반영되지 않는 변경 | 세 컬럼 `max`가 엇갈리면 누락 | <span class="pill ok">0</span> 키별 `row_number` 1건 | 5장 |
| S2 MERGE 원본 쿼리 | MERGE가 읽는 log 범위 | 누적 전체, 시간이 갈수록 증가 | <span class="pill ok">10분 윈도우 하나</span> 실행 시간이 운영 기간과 무관해짐 | 5장 |
| S3 순서 값 재인코딩 | 같은 초에 ord ≥ 1000일 때 순서 역전·값 충돌 | 대량 변경 시 발생 | <span class="pill ok">0</span> | 5장 |
| S4 버전 비교 | 늦게 온 과거 변경이 최신 상태를 덮어씀 | 백필·지연 시 발생 | <span class="pill ok">0</span> 처리 순서와 무관한 결과 | 5장 |
| S5 타입 통일 + 테이블 단위 격리 | 한 컬렉션의 타입 혼재로 멈추는 범위 | 같은 실행의 모든 테이블 | <span class="pill ok">그 테이블 하나</span> | 5장 |
| S7 일 파티션 | log 테이블 파티션 수 | 연간 약 52,000개 | <span class="pill ok">연간 365개</span> | 5장 |
| S8 중복 실행 방지 | 같은 테이블에 MERGE하는 driver | pod 재시작 시 2개 가능 | <span class="pill ok">항상 1개</span> | 5장 |
| log 적재를 공식 Sink로 (대안) | 운영 구성요소 | 커넥터, S3 버퍼, CronJob, Kyuubi, checkpoint | <span class="pill ok">Sink + mirror MERGE 배치</span> 넘겨받기 문제 자체가 사라짐 | 7장 |
| 최신성 | mirror 반영 지연 | 15~25분 | <span class="pill warn">그대로</span> 이번 개선의 목표가 아님 | 2장 |

</div>

> **측정 방법**
>
> 3장의 버퍼 대비 log 건수 비교를 윈도우마다 자동으로 돌리면, 누락이 0인지를 매일 확인하는 지표가 됩니다. 여기에 버퍼 데이터의 `max(source.ord)`, MERGE 실행 시간 추이, 실행별 실패 테이블 수를 함께 기록합니다.
{: .callout .info}

* * *

## 9. 참고 {#refs}

- 앞 문서: [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html)
- [Apache Iceberg: Spark Writes (MERGE INTO, overwritePartitions)](https://iceberg.apache.org/docs/latest/spark-writes/)
- [Apache Iceberg: Kafka Connect (latest)](https://iceberg.apache.org/docs/latest/kafka-connect/) · [nightly](https://iceberg.apache.org/docs/nightly/kafka-connect/)
- [apache/iceberg: kafka-connect 소스 (공식, 현재 관리)](https://github.com/apache/iceberg/tree/main/kafka-connect)
- [databricks/iceberg-kafka-connect](https://github.com/databricks/iceberg-kafka-connect) (구 tabular-io, 유지보수 중단)
- [Debezium MongoDB connector (source.ts_ms, ord, capture.mode)](https://debezium.io/documentation/reference/stable/connectors/mongodb.html)
- [Kafka Connect: SinkTask preCommit / open / close](https://kafka.apache.org/documentation/#connect_developing)
