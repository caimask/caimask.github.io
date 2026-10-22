---
layout: post
title: Logstash로 MySQL 데이터를 Kafka로 보내기
tags: [Logstash, MySQL, Kafka, CDC]
author: caimask
---

MySQL 테이블을 5초마다 조회해 Kafka 토픽으로 보내던 Logstash 구성을 정리한 2020년 글을 보강했습니다. 당시 "나중에 binlog 기반으로 바꿔 봐야겠다"고 적어 둔 숙제가 이후 CDC 파이프라인 작업으로 이어졌기 때문에, 이 방식의 동작과 한계를 로그 기반 CDC와 비교해 정리했습니다.

원문 2020-08-06 · 보강 2026-10-21 · Logstash jdbc input · Kafka output · 연관: [Apache Kafka](/2020/08/20/post-kafka.html) · [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **쿼리 기반 수집입니다.** 마지막으로 읽은 `id`를 기억해 두고, 5초마다 그보다 큰 행을 최대 1만 건씩 조회해 Kafka로 보냅니다. 설정만으로 만들 수 있어 빠르게 붙이기 좋았습니다.
2. **insert만 있는 로그성 테이블에서만 맞는 방식입니다.** 증가하는 `id`를 기준으로 삼기 때문에 update와 delete는 보이지 않습니다.
3. **처리량 상한이 설정에 박혀 있습니다.** 5초에 1만 건이면 초당 2,000건이 최대이고, 그보다 많이 쌓이면 지연이 계속 늘어납니다.
4. **로그 기반 CDC(binlog → Debezium)가 이 한계를 모두 없앱니다.** 변경 종류와 이전 값까지 전달되고, 원본 DB에 조회 부하를 주지 않습니다. 대신 하류 저장소가 upsert와 delete를 감당해야 하는 문제가 새로 생깁니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [당시 구성](#setup)
2. [동작 방식](#how)
3. [이 방식의 한계](#limits)
4. [로그 기반 CDC와 비교](#cdc)
5. [binlog 이후에 만난 문제](#after)
6. [참고](#refs)

</div>

## 1. 당시 구성 {#setup}

이벤트 로그가 쌓이는 MySQL 테이블을 실시간 처리 쪽으로 넘기기 위해, Logstash 하나로 MySQL → Kafka 경로를 만들었습니다.

```
MySQL (이벤트 로그 테이블, insert only)
  │  Logstash jdbc input: 5초마다 SELECT ... WHERE id > :sql_last_value LIMIT 10000
  ▼
Logstash
  │  kafka output: message_key = table_id, 본문 = event JSON 그대로
  ▼
Kafka 토픽 → 스트림 처리
```

아래는 당시 설정입니다. 접속 주소와 계정 정보는 지우고 자리표시로 바꿨습니다.

```
input {
  jdbc {
    jdbc_driver_library => "/etc/logstash/mysql-connector-java-5.1.39-bin.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://<db-host>:3306/<database>?useUnicode=true&characterEncoding=utf8"
    jdbc_user => "${JDBC_USER}"
    jdbc_password => "${JDBC_PASSWORD}"
    use_column_value => true
    tracking_column => "id"
    schedule => "*/5 * * * * *"
    statement => "SELECT id, json_extract(`event`, '$') as event, json_unquote(json_extract(`event`,'$.table.id')) as table_id FROM hand WHERE id > :sql_last_value limit 10000"
    last_run_metadata_path => "/etc/logstash/last_sql_index"
  }
}
output {
  kafka {
    topic_id => "<topic>"
    message_key => "%{table_id}"
    codec => plain {
      format => "%{event}"
    }
    bootstrap_servers => "<broker-1>:6667,<broker-2>:6667"
  }
}
```

> **비밀번호는 설정 파일에 쓰지 않습니다**
>
> Logstash는 `${VAR}` 형태로 환경 변수나 [keystore](https://www.elastic.co/guide/en/logstash/current/keystore.html) 값을 읽습니다. 계정 정보는 keystore에 두고 설정 파일에는 이름만 남깁니다.
{: .callout .warn}

## 2. 동작 방식 {#how}

<div class="table-wrap" markdown="1">

| 설정 | 의미 |
| --- | --- |
| `schedule => "*/5 * * * * *"` | 초 필드가 있는 6자리 cron. 5초마다 실행 |
| `use_column_value`, `tracking_column => "id"` | 마지막 실행 시각이 아니라 **마지막으로 읽은 `id` 값**을 기준으로 삼음 |
| `:sql_last_value` | 직전 실행에서 읽은 마지막 `id`. 쿼리에 그대로 치환됨 |
| `last_run_metadata_path` | `:sql_last_value`를 파일로 저장. Logstash를 재시작해도 이어서 읽음 |
| `limit 10000` | 한 번에 가져오는 최대 건수 |
| `message_key => "%{table_id}"` | 이벤트 안의 `table.id`를 Kafka 키로 사용. 같은 `table_id`의 이벤트는 같은 파티션에 순서대로 들어감 |

</div>

`message_key`를 정한 것이 이 설정에서 가장 중요한 부분이었습니다. 하류에서 같은 `table_id` 단위로 이벤트 순서가 지켜져야 했기 때문입니다. 키와 파티션, 순서의 관계는 [Apache Kafka](/2020/08/20/post-kafka.html#ordering) 글에 정리했습니다.

## 3. 이 방식의 한계 {#limits}

<div class="table-wrap" markdown="1">

| 한계 | 원인 | 영향 |
| --- | --- | --- |
| <span class="pill bad">미감지</span> update, delete | 증가하는 `id`만 봄 | 행이 바뀌거나 지워져도 하류는 모름 |
| <span class="pill warn">누락 가능</span> 늦게 커밋된 행 | auto increment 값은 커밋 순서가 아니라 할당 순서. 큰 `id`가 먼저 커밋되면 `:sql_last_value`가 앞서 나감 | 그 사이에 커밋된 작은 `id`의 행은 다시 조회되지 않음 |
| <span class="pill warn">중복 가능</span> 재시작 | Kafka 전송 후 `last_sql_index` 저장 전에 죽으면 같은 구간을 다시 읽음 | at-least-once. 하류가 중복을 견뎌야 함 |
| <span class="pill warn">처리량 상한</span> | 5초 × 1만 건 | 초당 2,000건을 넘으면 지연이 계속 쌓임 |
| <span class="pill warn">원본 부하</span> | 5초마다 조회 | `id` 인덱스가 있어도 운영 DB에 주기적인 읽기 부하 |
| <span class="pill info">지연</span> | 폴링 주기 | 최소 0~5초 지연 |

</div>

두 번째 항목은 당시에는 몰랐던 문제입니다. 트랜잭션 A가 `id=100`을 받고 아직 커밋하지 않은 사이 트랜잭션 B가 `id=101`을 받아 먼저 커밋하면, Logstash는 101까지 읽고 기준값을 101로 올립니다. 이후 A가 커밋해도 `id > 101` 조건에 걸리지 않아 100은 영영 빠집니다. 쓰기가 몰리는 테이블일수록 잘 생깁니다.

> **쿼리 기반 수집이 맞는 경우**
>
> insert만 있고, 쓰기 동시성이 낮거나 약간의 누락을 사후 보정할 수 있으며, 초 단위 지연이 괜찮은 로그성 테이블입니다. 그 밖의 경우에는 로그 기반 CDC를 씁니다.
{: .callout .info}

## 4. 로그 기반 CDC와 비교 {#cdc}

MySQL은 모든 변경을 binlog에 커밋 순서대로 기록합니다. 로그 기반 CDC는 DB를 조회하는 대신 이 binlog를 replica처럼 받아 읽습니다. 대표적인 도구가 Kafka Connect source 커넥터인 [Debezium](https://debezium.io/)입니다.

<div class="table-wrap" markdown="1">

|   | 쿼리 기반 (Logstash jdbc) | 로그 기반 (Debezium) |
| --- | --- | --- |
| 변경 종류 | insert만 (기준 컬럼에 따라 update 일부) | insert, update, delete 모두 |
| 이전 값 | 없음 | `before`, `after` 모두 전달 |
| 순서 | `id` 순서 (커밋 순서와 다를 수 있음) | 커밋 순서 |
| 누락 | 늦게 커밋된 행, 삭제 | binlog 보존 기간 안이면 없음 |
| 원본 DB 부하 | 주기적인 조회 | binlog 읽기만 |
| 원본 준비 사항 | 기준 컬럼과 인덱스 | `binlog_format=ROW`, 복제 권한, 충분한 binlog 보존 기간 |
| 위치 저장 | 마지막 `id` (파일) | binlog 파일과 위치, 또는 GTID (Kafka offset) |
| 스키마 변경 | 쿼리를 고쳐야 함 | 스키마 변경 이벤트로 전달 |

</div>

Debezium 이벤트는 대략 이런 모양입니다. 하류는 `op`를 보고 insert, update, delete를 구분합니다.

```
{
  "before": { "id": 100, "status": "READY" },
  "after":  { "id": 100, "status": "DONE"  },
  "op": "u",                     // c=insert, u=update, d=delete, r=스냅샷
  "source": { "file": "mysql-bin.000123", "pos": 4567, "ts_ms": 1600000000000 }
}
```

## 5. binlog 이후에 만난 문제 {#after}

CDC로 바꾸면 수집 쪽 문제는 대부분 사라집니다. 대신 문제가 하류로 옮겨 갑니다. 쿼리 기반일 때는 insert만 append하면 됐지만, CDC 이벤트를 받으면 하류 테이블이 **같은 키의 update와 delete를 순서대로 반영**해야 합니다.

이후 MongoDB CDC 이벤트를 Iceberg에 적재하는 작업에서 바로 이 부분이 어려웠습니다.

- 변경을 바로바로 반영하려고 몇 초마다 커밋하자, 같은 테이블에 여러 writer의 커밋이 몰려 경합이 생겼습니다. ([Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html))
- 커밋을 묶으면 한 커밋 안에서 같은 키의 변경을 정확히 지우는 방법이 필요했습니다. ([토스 Iceberg CDC 사례 비교](/2026/10/20/toss-iceberg-cdc-compare.html))
- 변경 이력(log)과 최신 상태(mirror)를 나눠, 이력은 append로 쌓고 최신 상태는 배치 `MERGE`로 만드는 구조로 옮겼습니다. ([Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html))

> **배운 점**
>
> 수집 방식을 고르는 일은 "어떤 변경을 하류에 넘길 것인가"를 정하는 일입니다. insert만 넘기면 수집이 어렵고, 모든 변경을 넘기면 반영이 어렵습니다. 둘 중 어느 쪽 비용을 낼지 먼저 정해야 합니다.
{: .callout .info}

* * *

## 6. 참고 {#refs}

- [Logstash Jdbc input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html)
- [Logstash Kafka output plugin](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-kafka.html)
- [Secrets keystore for secure settings](https://www.elastic.co/guide/en/logstash/current/keystore.html)
- [Debezium connector for MySQL](https://debezium.io/documentation/reference/stable/connectors/mysql.html)
