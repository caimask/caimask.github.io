# Iceberg 커밋 경합 리뷰

October 21, 2026

MongoDB CDC 이벤트를 Kafka에서 Iceberg로 적재하던 자체 개발 Sink Connector가 느렸던 원인을 짚고, “Iceberg도 이제 row lock으로 처리된다”는 정보가 맞는지, 그리고 대용량 데이터를 빠르게 적재하려면 어떤 방식을 써야 하는지 정리한 문서입니다.

작성 2026-10-08 · 대상 커넥터 v1.3.1 · Iceberg 1.4.3 · Glue Catalog

## 요약

1. **당시 “table lock” 현상의 정체**: Iceberg는 lock을 쓰지 않고 낙관적 동시성 제어(OCC)로 동작합니다. 한 테이블의 커밋은 한 번에 하나씩만 반영되며, 경쟁에서 진 writer는 재시도합니다. 커넥터가 **몇 초마다, 테이블당 최대 8개 writer에서** 커밋하면서 재시도가 폭주한 것이 원인입니다. 커밋 단계만 보면 table lock과 같은 효과이고, 충돌이 잦을 때는 재작업과 backoff 때문에 lock보다 오히려 느릴 수 있습니다.
2. **row lock은 Iceberg 공통 스펙이 아닙니다**: Databricks가 발표한 row-level concurrency는 Delta Lake 기능입니다. 이 기능도 커밋 직렬화를 없애지 않고, 충돌을 행 단위로 판정해 **실패를 줄여 줄 뿐**입니다. Iceberg v3(deletion vector, row lineage)도 커밋 모델은 그대로입니다.
3. **커밋을 묶어도 변경 이력은 사라지지 않습니다**: 지금도 한 배치 안의 중간 변경은 mirror 테이블에서 사라집니다. 변경 이력은 스냅샷이 아니라 log 테이블의 **행**으로 보존해야 하며, 현재 log 테이블은 이미 그렇게 설계되어 있습니다.
4. **처리량과 커밋 빈도는 별개입니다**: 대용량 적재의 표준은 **병렬 writer + 테이블당 단일 committer + 1\~5분 커밋**입니다. 커밋 직렬화가 정하는 것은 데이터 최신성의 하한이지 처리량이 아닙니다.
5. **최적화가 감당이 안 됐던 이유**: 파일 수는 커밋 횟수에 비례하므로 커밋을 묶으면 크게 줄어듭니다. 하지만 근본 원인은 **equality delete가 파티션 없는 mirror 테이블 전체에 걸리는 것**이라, bucket 파티션이나 MERGE 구조 전환과 증분 컴팩션 운영이 함께 필요합니다.
6. **삭제 방식은 테이블 성격에 맞춰 고릅니다**: equality delete는 쓰기가 빠른 대신 읽기와 컴팩션 비용이 크고, position delete(v3에서는 deletion vector)와 copy-on-write는 쓰기 시점에 비용을 더 냅니다. 현재 커넥터는 신규 insert에도 equality delete를 쓰고 있어, log 테이블에서는 대부분 불필요한 비용입니다.

### 목차

1. [기존 커넥터 평가](#code)
2. [table lock과 row lock의 실체](#lock)
3. [커밋을 묶으면 변경 이력이 사라지는가](#history)
4. [Databricks row-level concurrency와 Iceberg의 차이](#databricks)
5. [대용량 실시간 적재 권장 방식](#scale)
6. [Iceberg 삭제 방식 비교](#deletes)
7. [테이블 운영 최적화](#ops)
8. [이 프로젝트 적용안](#plan)
9. [참고 자료](#refs)

## 1. 기존 커넥터 평가

커넥터는 `put()`이 호출될 때마다(poll 1회, 최대 3000건) 레코드를 테이블별로 묶어 writer로 쓰고, 그 자리에서 `rowDelta.commit()`을 호출합니다. upsert 모드에서는 `BaseEqualityDeltaWriter`로 equality delete를 함께 씁니다.

### 운영 설정에서 확인된 구조

| 커넥터 | tasks | 대상 테이블 | 모드 |
| --- | --- | --- | --- |
| `log-datalake-sync-upsert` | 3 | `repo_cdc_mirror_*` (major 4종 제외) | upsert, 키 `_oid` |
| `log-datalake-sync-upsert-major` | 3 | `repo_cdc_mirror_{product,property,room_type,rate_plan}` | upsert, 키 `_oid` |
| `snapshot-datalake-sync-upsert` | 5 | `repo_cdc_mirror_{product,property,room_type,rate_plan}` | upsert, 키 `_oid` |
| `log-datalake-sync-append-major` | 3 | `repo_cdc_log_*` (major 4종) | 키 `_oid, __source_ts_ms_ord`, 일 파티션 |

> **major mirror 테이블 4종은 writer가 최대 8개**
> 
> upsert-major(3)와 snapshot(5) 커넥터가 같은 `repo_cdc_mirror_*` 테이블에 각자 몇 초 간격으로 커밋합니다. 커밋 경합이 가장 심했던 지점입니다.

### 성능 문제

| 위치 | 내용 |
| --- | --- |
| `IcebergUtil.java:106-109` | `put()`마다 테이블별로 즉시 커밋합니다. 커밋이 몇 초 간격으로 일어납니다. |
| `tasks.max=3` + `topics.regex` | 한 토픽의 파티션이 여러 task에 나뉘어, 같은 테이블에 task 3개가 동시에 커밋합니다. |
| `IcebergUtil.java:55-60` | 스키마 비교를 레코드마다 하고, 스키마 확장도 별도 커밋이라 경합을 키웁니다. |
| `IcebergSinkTask.java:96` | 매 `put()`마다 `loadTable`을 호출해 Glue GetTable과 metadata.json 읽기를 반복합니다. |
| MOR + equality delete | 커밋마다 작은 data/delete/manifest 파일이 생깁니다. 컴팩션이 없으면 커밋과 읽기가 함께 느려집니다. |

### 그 밖의 평가

- 양호 같은 배치 안의 중복 키를 position delete로 올바르게 처리합니다.
- 양호 실패 시 예외로 offset 커밋을 막아 at-least-once를 보장하고, upsert라 재처리해도 결과가 같습니다.
- 개선 실패 시 writer를 `abort()`하지 않아 orphan 파일이 남습니다.
- 개선 한 `put()`에서 여러 테이블 커밋이 원자적이지 않습니다. upsert 테이블은 괜찮지만 append 계열은 확인이 필요합니다.
- 개선 `RuntimeException`을 던지면 task가 FAILED로 멈춥니다. `errors.retry.timeout`은 converter와 SMT에만 적용되고 `put()`에는 적용되지 않습니다.
- 개선 `iceberg-spark-runtime`을 통째로 포함해 jar가 무겁습니다. Iceberg 1.4.3도 오래된 버전입니다.
- 확인 snapshot 커넥터 prod 설정의 warehouse 경로가 `.../pros/data/iceberg`로 오타로 보입니다.

## 2. table lock과 row lock의 실체

**질문** 당시 table lock 때문에 적재가 느렸는데, 요즘은 Iceberg가 row lock으로 처리한다고 들었다. 개선이 가능한가?

Iceberg는 처음부터 lock을 쓰지 않습니다. 데이터 파일과 메타데이터를 먼저 쓰고, 마지막에 카탈로그(Glue)의 metadata 포인터를 원자적으로 교체하는 방식으로 커밋합니다.

```
writer A: [데이터 파일 쓰기]──┐ commit(base=v10 → v11)  성공
writer B: [데이터 파일 쓰기]──┴ commit(base=v10 → v11)  실패: 이미 v11 있음
                                 └ v11 다시 읽기 → 변경 재적용 → commit(v11 → v12)  성공
```

- 데이터 파일 쓰기는 완전히 병렬로 일어납니다.
- 커밋 순간만 직렬화됩니다. 경쟁에서 진 쪽은 실패로 끝나지 않고 자동으로 재시도합니다(`commit.retry.num-retries`, 지수 backoff).
- 이 커넥터의 `RowDelta`는 충돌 검증을 하지 않으므로 데이터 충돌로 실패한 적은 없습니다. 원인은 순수하게 **커밋 순번을 잡기 위한 재시도 폭주**였습니다. 재시도할 때마다 메타데이터 재로딩, manifest 재작성, backoff 대기가 반복되어 task가 멈춘 것처럼 보였습니다.

### 낙관적 동시성 제어(OCC)란

여러 트랜잭션이 같은 데이터를 동시에 수정할 때, 충돌이 드물다고 가정하고 미리 lock을 걸지 않는 방식입니다. 반대 개념은 수정 전에 lock을 잡는 비관적 lock입니다.

1. **읽기**: 데이터를 읽을 때 현재 버전(숫자, 타임스탬프, 해시 등)을 함께 읽습니다.
2. **작업**: 읽은 데이터를 바탕으로 변경 내용을 만듭니다. 이 동안 아무것도 잠그지 않습니다.
3. **검증과 커밋**: 저장 직전에 “내가 읽은 버전 = 현재 버전”인지 확인합니다. 같으면 저장하고 버전을 올립니다. 다르면 다른 트랜잭션이 먼저 바꾼 것이므로 실패 처리하고 재시도합니다.

|  | OCC (낙관적) | 비관적 lock |
| --- | --- | --- |
| 충돌이 드물 때 | 빠름 대기 없음 | 보통 lock 획득과 해제 비용 |
| 충돌이 잦을 때 | 느림 실패와 재작업 반복 | 보통 순서대로 대기 후 진행 |
| 데드락 | 없음 | 발생 가능 |
| 적합한 환경 | 읽기가 많고 같은 대상에 대한 동시 수정이 드문 경우 | 짧은 시간에 많은 트랜잭션이 같은 대상을 다투는 경우 (재고 차감, 선착순 예약 등) |

### Iceberg에 대입하면

| OCC 개념 | Iceberg에서의 대상 |
| --- | --- |
| 보호하는 데이터 | 행이 아니라 **테이블의 현재 metadata 포인터** 하나 |
| 버전 | 현재 metadata.json 위치 (Glue에서는 테이블 버전으로 조건부 업데이트) |
| 작업 단계 | data/delete 파일, manifest, 새 metadata.json 쓰기 (병렬, 무잠금) |
| 검증과 커밋 | “base 버전이 여전히 현재인가”를 조건으로 포인터 교체 |
| 재시도 | Iceberg 라이브러리가 자동 수행 (`commit.retry.num-retries`, 지수 backoff). 데이터 충돌 검증에 실패하면(`ValidationException`) 재시도 없이 실패 |

일반적인 OCC 설명에서는 “충돌 시 애플리케이션이 재시도를 직접 구현해야 한다”고 하지만, Iceberg는 커밋 재시도를 라이브러리가 대신 해 줍니다. 대신 보호 대상이 테이블 포인터 하나뿐이라, **같은 테이블에 커밋하는 모든 writer가 같은 대상을 두고 다툽니다.** OCC가 피해야 하는 환경으로 꼽는 “많은 트랜잭션이 같은 대상을 짧은 간격으로 다투는 경우”가 바로 이 커넥터의 상황이었습니다.

### 그렇다면 결국 table lock 효과가 아닌가

**질문** table lock이 아니라 테이블 단위 커밋 경합이라고 했지만, 결국 table lock의 효과를 내는 것 아닌가?

결과만 보면 맞습니다. 커밋 단계만 놓고 보면 테이블 단위 lock과 같은 효과가 납니다. 한 테이블에 동시에 반영될 수 있는 커밋은 하나뿐이고, 나머지는 기다렸다가 다시 해야 합니다. 체감 현상을 “table lock 때문에 느리다”고 표현해도 틀리지 않습니다. 다만 동작 방식이 달라서 대응 방법도 달라집니다.

|  | 테이블 lock (비관적) | Iceberg OCC (낙관적) |
| --- | --- | --- |
| 막히는 구간 | lock을 잡은 순간부터 트랜잭션 끝까지 | 마지막 포인터 교체 순간만 |
| 데이터 파일 쓰기 | lock을 기다리느라 직렬로 진행되기 쉬움 | 완전히 병렬 |
| 진 쪽의 행동 | 줄 서서 대기하다가 차례가 오면 바로 커밋 | 이미 한 작업을 버리고 다시 함 (메타데이터 재로딩, manifest 재작성) 후 재시도 |
| 대기 방식 | 앞 작업이 끝나는 즉시 진행 | 지수 backoff로 정해진 시간만큼 대기 (앞 작업이 끝났어도) |
| 읽기 | 구현에 따라 막힐 수 있음 | 막히지 않음 |
| writer가 죽으면 | lock이 남아 정리가 필요할 수 있음 | 남는 것 없음 (고아 파일만) |

> **경합이 심하면 lock보다 느릴 수 있습니다**
> 
> lock이었다면 writer 8개가 줄을 서서 하나씩 커밋하고 끝났을 것입니다. OCC에서는 진 writer 7개가 각자 메타데이터를 다시 읽고 manifest를 다시 쓴 뒤 다시 경쟁하고, 그 사이 또 누군가 커밋하면 다시 집니다. **커밋 순서 대기 + 버려지는 재작업 + backoff 공백**이 모두 쌓여, 단순히 lock을 기다리는 것보다 낭비가 큽니다.

그래서 대응 방향이 다릅니다. lock 문제라면 “lock을 짧게, 더 세밀하게”가 해법이고, row lock에 대한 기대가 여기서 나옵니다. OCC 문제는 **“충돌이 드물다”는 전제를 되살리는 것**이 해법입니다. 커밋 횟수를 줄이고 테이블당 커밋 주체를 하나로 만들면 재시도가 사라지고, 쓰기 병렬과 읽기 무차단이라는 OCC의 장점만 남습니다.

> **결론**
> 
> Iceberg 공식 스펙에서는 v3를 포함해 지금도 **한 테이블의 커밋은 한 번에 하나씩** 반영됩니다. 다만 동시에 커밋을 시도할 수는 있고, 충돌이 없으면 재시도로 결국 성공합니다. 테이블이 다르면 커밋은 완전히 병렬입니다.

### Iceberg v3에서 바뀐 점

- **Deletion Vectors**: position delete를 데이터 파일당 bitmap(Puffin) 하나로 관리합니다. 삭제를 쓰고 읽는 효율이 좋아집니다.
- **Row Lineage**: `_row_id`, `_last_updated_sequence_number`로 행 단위 변경을 추적합니다.
- 커밋은 여전히 테이블 단위 OCC이고, 충돌 검사는 파일과 스냅샷 단위입니다. 이 커넥터의 upsert는 equality delete를 쓰므로 deletion vector의 이점도 제한적입니다.

## 3. 커밋을 묶으면 변경 이력이 사라지는가

**질문** 커밋을 묶으면 변경 이력이 커밋 단위로만 남아 중간 변경이 사라지지 않나? 그래서 커밋을 자주 했다.

Iceberg 스냅샷 이력이 커밋 단위로만 남는 것은 맞습니다. 하지만 지금 구조에서는 커밋을 자주 해도 그 이력이 보존되지 않습니다.

### 지금도 배치 안의 중간 변경은 사라집니다

`DeltaTaskWriter`는 upsert일 때 `delete(record)` 후 `write(record)`를 호출합니다. `BaseEqualityDeltaWriter`는 같은 writer 안에서 이미 insert한 키가 다시 오면 이전 행을 position delete로 지웁니다.

```
같은 put() 안: _oid=A  v1 → v2 → v3
mirror 테이블:  v3만 남음 (v1, v2는 같은 커밋 안에서 삭제)
스냅샷 이력:    v1, v2는 어느 스냅샷에도 보이지 않음
```

- mirror 테이블의 스냅샷 이력은 변경 이력이 아니라 **poll 배치 단위로 샘플링한 상태**입니다. 커밋 주기를 늘리면 샘플링 간격이 넓어질 뿐입니다.
- snapshot 커넥터는 `history.expire.max-snapshot-age-ms=86400000`(1일)으로 스냅샷을 지우고 있어, 스냅샷은 애초에 이력 저장소로 쓰이지 않고 있습니다.

### 변경 이력은 log 테이블이 행으로 보존합니다

`repo_cdc_log_*`는 키에 `__source_ts_ms_ord`가 들어 있어 변경 하나하나가 서로 다른 행이 됩니다. 커밋을 1분이든 10분이든으로 묶어도 사라지는 변경이 없고, 스냅샷 만료나 컴팩션과도 무관합니다.

| 테이블 | 목적 | 커밋을 묶으면 |
| --- | --- | --- |
| `repo_cdc_mirror_*` | 최신 상태 | 잃는 것은 스냅샷이 찍히는 간격뿐 |
| `repo_cdc_log_*` | 변경 이력 | 손실 없음 |

### 커밋을 묶을 때 주의할 점

- **메모리**: writer를 오래 열어 두면 `insertedRowMap`이 커밋 전까지 모든 키를 메모리에 들고 있습니다. 시간 조건과 건수 조건을 함께 거세요.
- **가시성 지연**: 커밋 주기만큼 쿼리에 늦게 보입니다. 하위 소비자의 SLA에 맞춰 주기를 정하세요.
- **offset 정합성**: `preCommit()`에서 Iceberg 커밋에 성공한 offset만 반환해야 합니다.
- **키 유일성**: `__source_ts_ms_ord`가 같은 `_oid` 안에서 정말 유일해야 이력이 보장됩니다. converter에서 확인이 필요합니다.

## 4. Databricks row-level concurrency와 Iceberg의 차이

**질문** Databricks 발표에서 row 단위로 처리할 수 있다고 해서 Iceberg 공통 스펙인 줄 알았다. Iceberg 공식으로는 아직 테이블 단위 동시 커밋이 불가능한가?

Databricks의 row-level concurrency는 **Delta Lake 테이블 기능**입니다(DBR 14.3+, deletion vectors 활성화, 파티션 없는 테이블). 문서에 managed Iceberg 테이블에 적용된다는 내용은 없습니다. 그리고 Delta도 커밋은 테이블당 한 번에 하나입니다. `_delta_log`에 다음 버전 파일을 먼저 만든 쪽이 이깁니다.

| 방식 | 경쟁에서 진 writer가 하는 일 |
| --- | --- |
| 기존 Delta (파일 단위) | 같은 **파일**을 건드렸으면 `ConcurrentModificationException`으로 실패 |
| Delta + row-level concurrency | 같은 **행**이 아니면 deletion vector를 합쳐 재시도 성공 |
| Iceberg (OCC) | 검증 조건(파일, 파티션 단위)에 걸리지 않으면 재시도 성공 |

> **정리**
> 
> row-level concurrency는 MERGE/UPDATE끼리 같은 파일을 건드려 생기는 **실패**를 줄이는 기능이지, 커밋 직렬화를 없애는 기능이 아닙니다. 이 커넥터는 데이터 충돌이 아니라 커밋 순번 경쟁이 문제였으므로, Delta로 옮겼어도 같은 증상이 났을 것입니다. 해법은 포맷과 상관없이 **커밋 횟수와 테이블당 커밋 주체 수를 줄이는 것**입니다.

## 5. 대용량 실시간 적재 권장 방식

**질문** 테이블 단위 커밋 lock이 있는 이상 Iceberg는 대량 데이터의 빠른 적재와 거리가 있지 않나? 더 나은 권장 방식이 있나?

커밋은 데이터를 쓰는 단계가 아니라, 이미 써 둔 파일 목록을 메타데이터에 등록하는 단계입니다. 그래서 커밋 직렬화는 처리량이 아니라 **최신성의 하한**을 정합니다.

```
[Kafka 파티션 1..N] → [writer 1..N: Parquet 병렬 쓰기]    ← 처리량 결정 (수평 확장)
                                   ↓ 파일 목록만 전달
                       [committer 1개: 1~5분마다 테이블당 1회 커밋]  ← 메타데이터만
```

- 1분에 한 번 커밋해도 그 한 번에 수천만 건(수 GB)을 등록할 수 있습니다. writer를 늘리면 처리량이 늘고 커밋 횟수는 그대로입니다.
- lock은 테이블 단위라 컬렉션별 테이블끼리는 커밋이 병렬입니다.
- 기존 커넥터는 writer마다 각자 몇 초 간격으로 커밋해서, 처리량이 커밋 경쟁에 묶였습니다.

### 병렬 writer + 단일 committer 구현체

| 도구 | writer | committer | 커밋 주기 |
| --- | --- | --- | --- |
| Flink Iceberg Sink | 병렬 task | `IcebergFilesCommitter` (parallelism 1) | checkpoint마다 |
| Iceberg Kafka Connect Sink (공식, 1.7+) | 각 task | coordinator 1개 (control topic) | `iceberg.control.commit.interval-ms`, 기본 5분 |
| Spark Structured Streaming | executor | driver | micro-batch마다 |

일반적인 운영값은 커밋 주기 1\~5분입니다. 몇 초 단위로 줄이면 스냅샷과 작은 파일이 급증해 유지보수 비용이 커집니다.

> **공식 Kafka Connect Sink 도입 전 확인**
> 
> 원본인 Tabular 버전은 upsert 모드를 명시적으로 지원했지만, Apache 버전 문서에서는 upsert 지원 여부를 확인하지 못했습니다. 배포할 버전에서 upsert와 delete 동작을 꼭 검증하세요.

### 공식 Kafka Connect Sink 관련 링크

| 구분 | 링크 | 설명 |
| --- | --- | --- |
| 공식 문서 (안정판) | [iceberg.apache.org/docs/latest/kafka-connect](https://iceberg.apache.org/docs/latest/kafka-connect/) | 최신 릴리스 기준 문서. 운영 도입 시 기준 |
| 공식 문서 (nightly) | [iceberg.apache.org/docs/nightly/kafka-connect](https://iceberg.apache.org/docs/nightly/kafka-connect/) | main 브랜치 기준 문서. 아직 릴리스되지 않은 변경이 포함될 수 있음 |
| **공식 소스 코드** | [apache/iceberg › kafka-connect](https://github.com/apache/iceberg/tree/main/kafka-connect) | 현재 관리되는 코드. `kafka-connect`, `kafka-connect-events`, `kafka-connect-transforms`, `kafka-connect-runtime`(배포용 패키지) 모듈로 구성 |
| 이전 저장소 (유지보수 중단) | [databricks/iceberg-kafka-connect](https://github.com/databricks/iceberg-kafka-connect) | Tabular가 만든 원본 커넥터(`io.tabular.iceberg.connect.*`). README에 “코드를 Apache Iceberg에 기증해 더 이상 유지보수하지 않는다”고 명시. 이전 주소는 `tabular-io/iceberg-kafka-connect` |

두 버전은 커넥터 클래스 이름(`org.apache.iceberg.connect.IcebergSinkConnector`와 `io.tabular.iceberg.connect.IcebergSinkConnector`)과 설정 키가 일부 다릅니다. 인터넷의 예제를 참고할 때는 어느 버전 기준인지 확인하세요. 새로 도입한다면 Apache 버전을 기준으로 합니다.

### CDC(upsert) 권장 패턴

1. **Append 먼저, Merge는 나중에** (가장 일반적): Kafka에서 변경 로그 테이블로 append하고 1\~5분마다 커밋합니다. mirror 테이블은 Spark나 Databricks의 `MERGE INTO` 단일 job으로 5\~15분마다 반영합니다.
2. **Flink upsert sink + 주기적 컴팩션**: 분 단위 최신성이 필요할 때 씁니다. equality delete가 쌓이므로 컴팩션이 필수입니다.
3. **테이블 유지보수 자동화**: `rewrite_data_files`, `rewrite_position_delete_files`, `expire_snapshots`, `rewrite_manifests`. Glue와 S3 Tables의 자동 컴팩션이나 Databricks Predictive Optimization을 활용할 수 있습니다. 컴팩션도 커밋이므로 `partial-progress.enabled`를 켜고 실행 간격을 조율하세요.

### 초 단위 최신성이 필요하다면

Iceberg는 분 단위의 근실시간 분석 저장소입니다. 초 단위 upsert 최신성이 필요하면 다른 계층을 함께 검토합니다. 제품마다 성숙도 차이가 크니 PoC로 검증하세요.

- **Apache Paimon**: LSM 트리 기반으로, Flink에서 primary key upsert를 고빈도로 처리하도록 설계됐습니다.
- **Apache Hudi 1.x**: MOR 테이블에서 여러 writer가 동시에 쓰는 non-blocking concurrency control을 지원합니다.
- **조회 계층 분리**: 최신 몇 분은 Kafka나 원본 DB에서, 그 이전은 Iceberg에서 읽습니다.

## 6. Iceberg 삭제 방식 비교

upsert는 “기존 행 삭제 + 새 행 추가”이므로, 삭제를 어떤 방식으로 기록하느냐가 쓰기 속도, 읽기 속도, 운영 부담을 함께 결정합니다. Iceberg v2부터 지원하는 행 단위 삭제는 크게 Copy-on-Write와 Merge-on-Read로 나뉘고, Merge-on-Read는 다시 position delete와 equality delete로 나뉩니다.

```
Iceberg 행 단위 삭제
 ├── Copy-on-Write (CoW)  삭제 대상이 든 data 파일을 통째로 새로 씀
 └── Merge-on-Read (MoR)  원본은 두고 delete 파일만 기록, 조회 시 병합
      ├── Position delete  (파일 경로, 행 번호)로 기록   → v3: deletion vector
      └── Equality delete  (컬럼 값)으로 기록, 예: _oid = 'abc'
```

### Equality delete

- **동작**: 삭제할 행의 키 값만 equality delete 파일에 기록합니다. 대상 행이 어느 파일에 있는지 찾지 않습니다.
- **장점**: 사전 스캔이 없어 쓰기와 커밋 지연이 짧습니다. 스트리밍 CDC에 맞는 방식입니다.
- **단점**: 조회할 때 data 파일마다 해당 equality delete의 키 집합과 대조해야 해서 읽기 부하가 큽니다. 컴팩션으로 data 파일에 반영하기 전까지 비용이 계속 쌓입니다.
- **주요 사용처**: Flink upsert sink, Kafka Connect upsert 같은 실시간 CDC 적재. 이 커넥터가 쓰는 방식입니다.

### Position delete

- **동작**: 삭제할 행의 정확한 위치 `(file_path, row_position)`를 기록합니다.
- **장점**: 조회 시 해당 파일의 특정 행만 건너뛰면 되므로 equality delete보다 읽기 저하가 훨씬 적습니다.
- **단점**: 기록 전에 대상 행이 어느 파일의 몇 번째 행인지 찾는 스캔이 필요합니다.
- **주요 사용처**: Spark, Trino의 `MERGE`/`UPDATE`/`DELETE` (MoR 모드). 같은 writer가 방금 쓴 행을 다시 지울 때도 쓰입니다(이 커넥터의 배치 내 중복 처리).

### Copy-on-Write

- **동작**: 삭제 대상이 포함된 data 파일을 읽어, 대상 행을 뺀 새 파일로 다시 씁니다.
- **장점**: delete 파일이 없어 조회가 가장 빠릅니다.
- **단점**: 한 건만 바뀌어도 그 행이 든 파일 전체(수백 MB)를 다시 써서 쓰기 증폭이 큽니다. 업데이트가 여러 파일에 흩어지면 비용이 급격히 커집니다.
- **주요 사용처**: 읽기가 쓰기보다 압도적으로 많은 테이블, 물리적 즉시 삭제가 필요한 경우(GDPR 등).

### 비교

| 항목 | Copy-on-Write | Position delete (MoR) | Equality delete (MoR) |
| --- | --- | --- | --- |
| 기록 대상 | 재작성한 data 파일 | `(파일 경로, 행 위치)` | 키 컬럼 값 |
| 쓰기 비용 | 높음 파일 전체 재작성 | 보통 사전 스캔 | 낮음 값만 기록 |
| 읽기 비용 | 낮음 병합 없음 | 보통 위치 기반 필터 | 높음 키 대조 |
| 사전 스캔 | 필요 | 필요 | 불필요 |
| 삭제 영향 범위 | 해당 파일 | 해당 파일의 특정 행 | 같은 파티션의 이전 data 파일 전체 |
| 컴팩션 부담 | 거의 없음 | 보통 (delete 파일 병합) | 큼 (영향 범위 전체 재작성) |
| v3에서의 변화 | 없음 | deletion vector로 대체, data 파일당 1개 | 그대로 유지 |
| 주요 워크로드 | 대규모 배치, 물리 삭제 | Spark/Trino 배치, 마이크로 배치 MERGE | Flink/Kafka Connect 실시간 CDC |

### 보충과 정정

- **equality delete의 범위는 “전체 파일”이 아니라 “같은 파티션의 이전 파일”입니다.** equality delete는 같은 파티션에서 자기보다 data sequence number가 낮은 data 파일에만 적용됩니다. 그래서 파티션이 없는 mirror 테이블에서는 사실상 테이블 전체가 대상이 되고, `bucket(_oid)` 같은 파티션을 두면 범위가 줄어듭니다. 7장의 bucket 파티션 제안이 이 원리를 이용합니다.
- **컴팩션은 delete를 position delete로 “변환”하지 않습니다.** `rewrite_data_files`는 data 파일에 delete를 반영해 delete가 없는 새 data 파일을 씁니다. 더 이상 가리킬 data 파일이 없는 delete 파일은 이후 `rewrite_position_delete_files`와 `expire_snapshots`로 정리됩니다.
- **v3의 deletion vector는 position delete만 대체합니다.** equality delete는 v3에서도 그대로이고, 커뮤니티는 v4에서 equality delete를 줄이는 방향을 논의하고 있습니다. 스트리밍 upsert를 계속 equality delete로 하면 v3로 올려도 이 부분의 비용은 같습니다.
- **“쓰기 비용이 낮다”는 커밋 1회 기준입니다.** 읽기와 컴팩션에서 나중에 치르는 비용까지 합치면, 업데이트가 많은 테이블에서는 equality delete가 가장 비싼 방식이 될 수 있습니다.

### 이 커넥터에 대입하면

`DeltaTaskWriter.write()`는 upsert 모드에서 operation 종류와 상관없이 항상 `delete(record)`를 먼저 호출합니다. 신규 insert에도 equality delete 행이 하나씩 생깁니다.

| 테이블 | 현재 삭제 방식 | 평가 |
| --- | --- | --- |
| `repo_cdc_mirror_*` | equality delete + 배치 내 중복은 position delete | 최신 상태 유지에 필요하지만, 파티션이 없어 영향 범위가 테이블 전체 |
| `repo_cdc_log_*` | equality delete (키에 `__source_ts_ms_ord` 포함) | 키가 매 변경마다 달라서 지울 대상이 거의 없습니다. 재처리 시 중복 방지 역할만 하는, **대부분 불필요한 비용**입니다. |

> **log 테이블 개선 선택지**
> 
> exactly-once 적재(공식 Iceberg Kafka Connect Sink 등)로 바꾸면 재처리 중복이 생기지 않으므로 log 테이블을 순수 append(`upsert=false`)로 둘 수 있습니다. at-least-once를 유지해야 한다면 append로 쓰고 조회나 MERGE 단계에서 `(_oid, __source_ts_ms_ord)`로 중복을 제거하는 방식이 equality delete보다 저렴합니다.

### 테이블 성격별 권장

| 테이블 성격 | 권장 방식 |
| --- | --- |
| 변경 로그 (log) | append only. 삭제 기록 없음 |
| 최신 상태 (mirror), 분 단위 최신성 필요 | 스트리밍 equality delete + `bucket` 파티션 + 1\~2시간 증분 컴팩션 |
| 최신 상태 (mirror), 5\~15분 지연 허용 | log 테이블에서 Spark `MERGE` (MoR position delete, v3라면 deletion vector) |
| 읽기가 압도적으로 많고 업데이트가 적은 차원 테이블 | Copy-on-Write `MERGE` |

## 7. 테이블 운영 최적화

**질문** upsert로 삭제와 추가를 반복하고 매번 커밋해서 파일이 많고, 주기적으로 최적화를 돌려도 감당이 안 됐다. 커밋을 줄이면 같이 해결되는가, 아니면 최적화를 실행하는 더 나은 방식이 있는가?

커밋을 줄이면 상당 부분 해결되지만 전부는 아닙니다. 파일 개수는 커밋 횟수에 비례해 줄어듭니다. 하지만 최적화 범위가 넓어지는 근본 원인은 equality delete가 파티션 없는 mirror 테이블 전체에 걸린다는 점이라, 구조도 함께 바꿔야 합니다.

### 파일이 폭증하고 최적화가 감당이 안 되는 이유

```
커밋 1회 × task × 파티션 마다:
  data 파일 ≥1 + equality delete 파일 ≥1 (+ position delete)
  + manifest + manifest list + metadata.json
```

- **equality delete의 적용 범위가 넓습니다.** 같은 파티션 안에서 자기보다 먼저 들어온 모든 data 파일에 적용됩니다. mirror 테이블은 파티션이 없으므로 테이블 전체가 대상입니다.
- **컴팩션이 매번 테이블 전체를 다시 씁니다.** equality delete는 `rewrite_data_files`로 data 파일에 반영해야만 없어지는데, 적용 대상이 전체 파일이기 때문입니다. 그동안에도 커넥터는 몇 초마다 새 delete를 쌓습니다.
- **컴팩션도 커밋이라 커넥터와 경합합니다.** 큰 컴팩션일수록 커밋 직전에 실패할 확률이 높습니다.

### 커밋을 줄이면 해결되는 것과 안 되는 것

| 항목 | 커밋 묶기 효과 |
| --- | --- |
| data/delete/manifest/metadata 파일 개수 | 감소 커밋 주기에 비례 (몇 초 → 5분이면 수십\~수백 분의 1) |
| delete 행 수 | 감소 같은 `_oid`의 연속 변경이 하나로 합쳐짐 |
| 컴팩션과 커넥터의 커밋 충돌 | 감소 커넥터 커밋이 드물어짐 |
| equality delete가 테이블 전체에 걸리는 문제 | 해결 안 됨 |
| 업데이트가 테이블 전반에 흩어지는 문제 | 해결 안 됨 데이터 특성 |

### 최적화 범위를 줄이는 구조 변경

**A. mirror 테이블에 bucket 파티션 추가** (커넥터를 유지하는 경우)

```
ALTER TABLE glue.yanolja_mdp_prod.repo_cdc_mirror_xxx
ADD PARTITION FIELD bucket(16, _oid);
```

- equality delete가 자기 bucket 안에서만 적용되어 영향 범위가 1/16로 줄어듭니다.
- 컴팩션을 bucket 단위로 나눠 병렬로, 부분적으로 실행할 수 있습니다.
- 기존 데이터는 이전 스펙으로 남으므로, 변경 후 한 번 전체 `rewrite_data_files`로 옮겨야 합니다.
- bucket 하나가 수백 MB\~수 GB가 되도록 bucket 수를 정합니다.

**B. mirror 테이블에서 equality delete 없애기** (권장)

- log 테이블에 append하고 Spark `MERGE INTO`로 mirror에 반영하면, MERGE는 대상 파일을 이미 찾았으므로 equality delete 대신 **position delete**를 씁니다. 영향 범위가 특정 파일의 특정 행으로 좁혀집니다.
- **Iceberg v3의 deletion vector가 여기서 효과를 냅니다.** data 파일당 delete 정보가 하나(DV)로 유지되어 delete 파일이 끝없이 쌓이는 구조가 사라집니다.
- 업데이트가 아주 많은 테이블은 copy-on-write(`write.merge.mode=copy-on-write`)도 선택지입니다. 쓰기는 무겁지만 delete 파일이 생기지 않습니다.

### 필요한 운영 작업과 실행 순서

| 순서 | 작업 | 목적 |
| --- | --- | --- |
| 1 | `rewrite_data_files` | 작은 파일 병합, equality/position delete를 data 파일에 반영 |
| 2 | `rewrite_position_delete_files` | position delete 파일 병합, 무효가 된(dangling) delete 제거 |
| 3 | `rewrite_manifests` | manifest 병합으로 커밋과 쿼리 계획 속도 개선 |
| 4 | `expire_snapshots` | 오래된 스냅샷과 컴팩션으로 대체된 파일의 실제 삭제 |
| 5 | `remove_orphan_files` | 실패한 writer가 남긴 고아 파일 정리 (현재 커넥터는 `abort()`가 없어 필요) |

> **컴팩션과 스냅샷 만료는 세트입니다**
> 
> `expire_snapshots`를 하지 않으면 컴팩션을 해도 이전 파일이 S3에 그대로 남습니다.

테이블 속성도 함께 정리하세요. 현재는 snapshot 커넥터에만 들어가 있습니다.

```
write.metadata.delete-after-commit.enabled = true
write.metadata.previous-versions-max       = 50
write.target-file-size-bytes               = 268435456   # 256MB 전후
```

### 최적화 작업을 실행하는 권장 방식

핵심은 **“하루 한 번 크게”가 아니라 “자주, 작게, 바뀐 곳만”** 실행하는 것입니다.

```
-- 1) 바뀐 범위만, delete가 붙은 파일 위주로, 부분 커밋하면서
CALL glue.system.rewrite_data_files(
  table    => 'yanolja_mdp_prod.repo_cdc_mirror_xxx',
  strategy => 'binpack',
  -- where => '...',   -- bucket/일 파티션이 있으면 최근 변경 범위만 지정
  options  => map(
    'delete-file-threshold',              '2',
    'min-input-files',                    '5',
    'partial-progress.enabled',           'true',
    'partial-progress.max-commits',       '10',
    'max-concurrent-file-group-rewrites', '8',
    'max-file-group-size-bytes',          '10737418240'
  )
);

CALL glue.system.rewrite_position_delete_files(
  table   => 'yanolja_mdp_prod.repo_cdc_mirror_xxx',
  options => map('partial-progress.enabled', 'true')
);

CALL glue.system.rewrite_manifests('yanolja_mdp_prod.repo_cdc_mirror_xxx');

CALL glue.system.expire_snapshots(
  table       => 'yanolja_mdp_prod.repo_cdc_mirror_xxx',
  older_than  => current_timestamp() - INTERVAL 1 DAY,
  retain_last => 20
);

-- 주 1회 정도, 진행 중인 쓰기 보호를 위해 3일 이상 여유
CALL glue.system.remove_orphan_files(
  table      => 'yanolja_mdp_prod.repo_cdc_mirror_xxx',
  older_than => current_timestamp() - INTERVAL 3 DAYS
);
```

- **`partial-progress.enabled`는 필수입니다.** 큰 컴팩션을 여러 개의 작은 커밋으로 나눠, 도중에 충돌이 나도 끝난 그룹은 유지됩니다. “감당이 안 됐던” 상황을 완화하는 가장 직접적인 옵션입니다.
- **스트리밍 writer와의 충돌**: 기본값 `use-starting-sequence-number=true` 덕분에 컴팩션 도중 들어온 equality delete는 새 파일에도 계속 적용되어 충돌하지 않습니다. 충돌은 컴팩션 대상 파일에 position delete가 새로 붙을 때 납니다. 이 커넥터의 position delete는 같은 배치에서 새로 쓴 파일만 가리키므로 충돌이 적은 편입니다.
- **주기**: 커밋을 5분 단위로 묶었다면, 테이블별로 1\~2시간마다 증분 컴팩션, 하루 1회 스냅샷 만료가 무난합니다. 테이블 크기가 아니라 **delete 파일 비율**을 보고 조정하세요.
- **실행 환경**: 커넥터와 분리된 Spark job(EMR, Glue ETL, Databricks)으로 테이블별 병렬 실행합니다. AWS Glue Data Catalog의 자동 테이블 최적화를 쓰면 스케줄 관리 부담이 줄지만, MOR 테이블의 equality delete 처리 지원 범위는 사용 중인 버전에서 확인이 필요합니다.

### 현재 상태를 측정하는 쿼리

```
-- content: 0=data, 1=position delete, 2=equality delete
SELECT content, count(*) AS files,
       round(sum(file_size_in_bytes)/1024/1024) AS mb,
       round(avg(file_size_in_bytes)/1024/1024, 1) AS avg_mb
FROM glue.yanolja_mdp_prod.repo_cdc_mirror_xxx.files
GROUP BY content;

-- 시간대별 커밋 빈도
SELECT date_trunc('hour', committed_at) AS h, count(*) AS commits
FROM glue.yanolja_mdp_prod.repo_cdc_mirror_xxx.snapshots
GROUP BY 1 ORDER BY 1 DESC LIMIT 24;
```

equality delete 파일(content=2) 수가 data 파일 수에 가깝거나 더 많고 평균 파일 크기가 수 MB 이하라면, 지금 겪은 상황과 정확히 일치합니다.

### 단계별 조치

| 단계 | 조치 | 효과 |
| --- | --- | --- |
| 1 | 커밋 묶기 (5분) | 파일 수 수십\~수백 분의 1, 컴팩션 충돌 감소 |
| 2 | 증분 컴팩션 + `partial-progress` + 스냅샷 만료 세트 운영 | 감당 가능한 작업 단위로 분할 |
| 3 | mirror 테이블에 `bucket(_oid)` 파티션 추가 | equality delete 범위와 컴팩션 범위 축소 |
| 4 | mirror를 log 기반 `MERGE`로 전환하고 v3 적용 | equality delete 제거, delete 파일 누적 구조 해소 |

1\~2단계만으로도 체감이 크게 달라질 것입니다. 3\~4단계는 테이블 규모와 업데이트 비율을 측정한 뒤 결정합니다.

## 8. 이 프로젝트 적용안

단기

### 커밋 묶기

`put()`은 쓰기만 하고 `preCommit()`에서 1\~5분 또는 N건마다 테이블당 1회 커밋합니다. 테이블 캐시와 `refresh()`를 쓰고 스키마 비교는 배치 단위로 합니다.

단기

### writer 분리

major mirror 테이블에 같이 쓰는 snapshot 커넥터와 upsert-major 커넥터를 시간 또는 대상 테이블로 분리합니다.

단기

### 증분 유지보수

`partial-progress`를 켠 증분 컴팩션을 1\~2시간마다, 스냅샷 만료를 하루 1회 실행합니다. 7장 참고.

중장기

### 적재 구조 전환

log 테이블은 공식 Iceberg Kafka Connect Sink 또는 Flink로 append합니다. mirror 테이블은 log 테이블에서 주기적으로 `MERGE`하는 단일 job으로 바꾸고 v3를 적용합니다.

중기

### bucket 파티션

커넥터 upsert를 유지하는 동안에는 mirror 테이블에 `bucket(_oid)` 파티션을 추가해 equality delete 범위를 줄입니다.

먼저 하위 소비자에게 분 단위 최신성이면 충분한지 확인해야 합니다. 그 답에 따라 Iceberg 단독으로 갈지, 실시간 계층을 함께 둘지가 정해집니다.

---

## 9. 참고 자료

- [Apache Iceberg: Kafka Connect (latest)](https://iceberg.apache.org/docs/latest/kafka-connect/) · [nightly](https://iceberg.apache.org/docs/nightly/kafka-connect/)
- [apache/iceberg: kafka-connect 소스 (공식, 현재 관리)](https://github.com/apache/iceberg/tree/main/kafka-connect)
- [Wikipedia: Optimistic concurrency control](https://en.wikipedia.org/wiki/Optimistic_concurrency_control)
- [Apache Iceberg: Reliability (optimistic concurrency)](https://iceberg.apache.org/docs/1.7.1/reliability)
- [AWS: Manage concurrent write conflicts in Apache Iceberg on the Glue Data Catalog](https://aws.amazon.com/blogs/big-data/manage-concurrent-write-conflicts-in-apache-iceberg-on-the-aws-glue-data-catalog)
- [Databricks: Row-level concurrency](https://docs.databricks.com/aws/en/optimizations/isolation/row-level-concurrency)
- [Databricks: Deep dive on row-level concurrency](https://databricks.com/blog/deep-dive-how-row-level-concurrency-works-out-box)
- [Databricks: Apache Iceberg v3 public preview](https://www.databricks.com/blog/next-era-open-lakehouse-apache-icebergtm-v3-public-preview-databricks)
- [AWS: Working with Apache Iceberg V3](https://docs.aws.eu/AmazonS3/latest/userguide/working-with-apache-iceberg-v3.html)
- [Apache Iceberg Spec: Row-level Deletes](https://iceberg.apache.org/spec/#row-level-deletes)
- [Apache Iceberg: Spark Procedures (유지보수 프로시저)](https://iceberg.apache.org/docs/latest/spark-procedures/)
- [Apache Iceberg: Maintenance](https://iceberg.apache.org/docs/latest/maintenance/)
- [databricks/iceberg-kafka-connect](https://github.com/databricks/iceberg-kafka-connect) (구 tabular-io, 유지보수 중단)
- [Why Iceberg V4 wants to retire equality deletes](https://datalakehousehub.com/blog/equality-deletes-iceberg-v4)
- [Surviving commit conflicts when dozens of writers hit the same Iceberg table](https://datalakehousehub.com/blog/iceberg-concurrent-commits-agents)

Tags[Iceberg](https://caimask.github.io/tags#Iceberg)[Kafka Connect](https://caimask.github.io/tags#Kafka+Connect)