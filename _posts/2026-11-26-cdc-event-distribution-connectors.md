---
layout: post
title: CDC 이벤트 분배·트리거·지연 측정 커넥터 설계와 회고
tags: [Kafka Connect, Kafka, CDC, Debezium, gRPC]
author: caimask
---

하나의 CDC 토픽을 여러 하류가 쓰도록 만든 Kafka Connect 플러그인 세 종류(이벤트 분배 커넥터, 내부 API 트리거 2종, 수집 지연 측정 커넥터)를 설계·개발해 운영한 기록입니다. 각 커넥터가 무엇을 하고 왜 그렇게 만들었는지 정리하고, 이 글을 쓰며 코드를 다시 읽다가 찾은 결함을 회고로 적었습니다.

작성 2026-11-26 · 역할: 세 커넥터 모두 직접 설계·개발 · 분배 커넥터 2024-01 ~ 2025-09, 정합성 보정 트리거 2024-01 ~ 2025-09, 콘텐츠 보강 트리거 2024-04 ~ 2025-09, 지연 측정 커넥터 2025-06 ~ · Kafka Connect(Confluent Platform 7.2.6) · Debezium MongoDB 2.0.1
{: .meta}

이 글을 포함하는 사례: [이벤트 기반 데이터 파이프라인 구축](/portfolio/event-driven-pipeline)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **분배 커넥터**는 CDC 토픽에서 "실제로 바뀐" 생성·수정만 골라, 하류마다 정한 테이블과 키 기준으로 용도별 토픽에 다시 발행합니다. 분배 대상은 설정 몇 줄로 늘리고, 한 분배의 설정 오류가 다른 분배를 막지 않게 했습니다.
2. **트리거 2종**은 분배된 토픽을 소비해 내부 gRPC API를 레코드마다 호출합니다. 호출 실패는 처음에는 로그만 남기고 넘겼고, 2025-01부터 DLQ 토픽에 남겨 다시 처리할 수 있게 바꿨습니다.
3. **지연 측정 커넥터**는 원본 DB 변경 시각과 Debezium 처리 시각에서 Kafka 기록까지 걸린 시간을, 파티션·DB별 10분 구간으로 집계해 CloudWatch Logs에 남깁니다.
4. **회고**: 분배 커넥터는 비동기 발행이 끝났는지 확인하지 않고 offset을 커밋하게 둘 수 있는 구조였고(E1), 트리거는 API 실패만 DLQ로 보내고 키 변환 실패에서는 task가 멈췄습니다(E2). 둘 다 이 글을 쓰며 찾았고 적용하지 않았습니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [배경과 구조](#background)
2. [분배 커넥터](#distributor)
3. [트리거 2종](#trigger)
4. [수집 지연 측정 커넥터](#latency)
5. [당시 가정과 확인한 사실](#assumptions)
6. [회고: 다시 읽고 찾은 결함](#retro)
7. [성과와 기대 효과](#outcome)
8. [남은 과제](#next)
9. [참고](#refs)

</div>

## 1. 배경과 구조 {#background}

원본 MongoDB의 변경은 Debezium source 커넥터가 CDC 토픽(`cdc.events`) 하나에 발행합니다. 이 토픽은 데이터 레이크 적재([별도 사례](/portfolio/cdc-iceberg-ingestion))가 소비하고 있었습니다. 그런데 내부 서비스 두 곳도 "어떤 엔티티가 바뀌었는지"를 알아야 했습니다. 하나는 엔티티의 정합성을 다시 맞추는 API이고, 다른 하나는 엔티티 콘텐츠를 보강하는 API입니다. 두 API가 원하는 대상 테이블과 식별 키가 서로 달랐습니다.

```
MongoDB ──(Debezium)──▶ cdc.events
                          │
         ┌────────────────┼──────────────────────────┐
         ▼                ▼                          ▼
   분배 커넥터        지연 측정 커넥터            (데이터 레이크 적재)
   (tasks 10)        (tasks 10)
     │                 └─▶ 10분 구간 지표 → CloudWatch Logs
     ├─▶ event.content ──▶ 정합성 보정 트리거 (tasks 6)  ──gRPC──▶ 보정 API
     │                        └─ 실패 → dlq.content
     └─▶ event.enhance ──▶ 콘텐츠 보강 트리거 (tasks 72) ──gRPC──▶ 보강 API
                              └─ 실패 → dlq.enhance
```

<div class="table-wrap" markdown="1">

| 커넥터 | 하는 일 | 시기 |
| --- | --- | --- |
| 분배 커넥터 | CDC 이벤트를 걸러 용도별 토픽에 하류 키로 다시 발행 | 2024-01 시작, 2024-04 다중 분배, 2024-09 설정 오류 격리 |
| 정합성 보정 트리거 | `event.content`를 소비해 보정 API 호출 | 2024-01 시작, 2025-01 DLQ |
| 콘텐츠 보강 트리거 | `event.enhance`를 소비해 보강 API 호출 | 2024-04 시작, 2025-01 DLQ |
| 지연 측정 커넥터 | CDC 토픽의 기록 지연을 10분 구간으로 집계 | 2025-06 시작 |

</div>

분배 커넥터와 트리거 2종은 2025-09에 운영을 종료하고 설정과 코드를 저장소의 archive로 옮겼습니다. 지연 측정 커넥터는 계속 운영합니다.

## 2. 분배 커넥터 {#distributor}

### 2.1 발행하는 이벤트는 "무엇이 바뀌었나"뿐 {#event-model}

분배 커넥터가 발행하는 메시지에는 문서 내용이 없습니다. Avro 스키마의 필드는 네 개뿐입니다.

<div class="table-wrap" markdown="1">

| 필드 | 내용 |
| --- | --- |
| 엔티티 종류 | 바뀐 문서의 타입 이름 |
| 고유 키 집합 | 그 엔티티를 찾는 키 (원본 문서의 `_id`에서 꺼냄) |
| 작업 코드 | `c`(생성) 또는 `m`(실제 수정) |
| 작업 시각 | 원본 DB에서 변경이 일어난 시각 (`source.ts_ms`) |

</div>

메시지에는 식별 정보만 실리므로 원본 문서의 크기와 관계없이 작게 유지됩니다.

### 2.2 "실제로 바뀐 수정"만 골라내기 {#modified}

Debezium의 update 이벤트(`op=u`)에는 부분 수정(update)과 문서 교체(replace)가 함께 들어오고, 내용이 바뀌지 않은 쓰기도 update로 나올 수 있습니다. 원본 문서에는 "수정 시각"과 "동기화 시각" 필드가 따로 있었습니다. 실제 데이터가 바뀐 경우에만 하류에 알리려고 자체 작업 코드 `m`을 두고, 다음 기준을 만족할 때만 update를 `m`으로 바꿨습니다.

```
op = u 이고
  updateDescription 있음 (부분 수정) → 변경 필드 목록에 "수정 시각" 필드가 있을 때
  updateDescription 없음 (문서 교체) → 문서의 "수정 시각" == "동기화 시각" 일 때
→ op = m (실제 수정)
그 밖의 update → u 그대로 (분배 설정이 c,m 이므로 발행되지 않음)
after가 없는 이벤트 (delete, tombstone) → 발행하지 않음
```

이 기준은 원본이 "데이터가 실제로 바뀔 때 수정 시각을 갱신한다"는 규칙을 지킨다는 전제 위에 있습니다. 분배 대상 두 개 모두 `c,m`만 받도록 설정했습니다.

### 2.3 분배 대상은 설정으로 {#multi-way}

처음(2024-01)에는 출력 토픽이 하나였습니다. 2024-04에 보강 API가 붙으면서 분배 대상을 이름별 설정 묶음으로 바꿨습니다.

```
distribute.list = content, enhance

distribute.config.content.topic          = event.content
distribute.config.content.table.regex    = (<엔티티A 테이블들>|<엔티티B 테이블들>)
distribute.config.content.operation.type = c,m
distribute.config.content.key.field      = <정규식1>:<키 필드1>,<정규식2>:<키 필드2>,...

distribute.config.enhance.topic          = event.enhance
distribute.config.enhance.table.regex    = (<엔티티A>|<엔티티B>|<엔티티C> 테이블들)
distribute.config.enhance.operation.type = c,m
distribute.config.enhance.key.field      = _id
```

- **분배마다 producer 하나**를 두고, 이벤트 하나를 모든 분배에 차례로 넘깁니다. 각 분배는 테이블 정규식과 작업 코드로 자기 대상인지 판단합니다.
- **키 필드를 테이블마다 다르게** 지정할 수 있게 했습니다. `테이블 정규식:필드` 형식을 쉼표로 나열하면 처음 맞는 규칙의 필드를 키로 씁니다. 운영 설정에서 보정용 분배는 테이블마다 다른 상위 엔티티 id를, 보강용 분배는 문서 `_id`를 키로 썼습니다.
- 메시지 키는 `{"id": <하류 키 값>}` 형태의 JSON 문자열입니다. 같은 하류 키의 이벤트는 분배 토픽에서 같은 파티션에 모입니다.

새 분배 대상은 이 설정 묶음 하나를 추가하는 것으로 붙였고, 코드는 고치지 않았습니다.

### 2.4 한 분배의 설정 오류가 다른 분배를 막지 않게 {#isolation}

여러 분배를 한 task에서 돌리면, 키는 이벤트 하나에 대해 모든 분배의 것을 한 번에 만듭니다. 그래서 한 분배의 키 설정이 문서와 맞지 않을 때 다른 분배의 발행까지 영향을 받을 수 있는 구조였습니다. 2024-09에 이렇게 바꿨습니다.

- 키를 만들 때 문서에 키 필드가 없는 분배는 키 목록에서 빼기만 합니다.
- 발행 단계에서 키가 없는 분배는 "키 없음"으로 로그를 남기고 건너뜁니다. 다른 분배는 그대로 발행합니다.

필터는 테이블 → 작업 코드 → 키 존재 순서로 적용하고, 건너뛴 이유를 분배 이름과 함께 로그로 남겨 어느 설정 때문에 빠졌는지 추적할 수 있게 했습니다.

### 2.5 실패 처리 {#distributor-failure}

<div class="table-wrap" markdown="1">

| 실패 | 처리 |
| --- | --- |
| 이벤트 변환 실패 (예상과 다른 형태의 레코드) | 운영 설정 `exception.message.skippable=true`. 오류 로그와 레코드를 남기고 버림 |
| `send()` 호출 자체의 예외 (직렬화, 버퍼 등) | 300·600·1,200ms 간격으로 3회 재시도, 그래도 실패하면 task 실패 |
| 브로커 쓰기 실패 (비동기) | 콜백에서 예외를 던지게 했으나 task에 전달되지 않음 → 회고 [E1](#e1) |

</div>

producer는 `acks=all`로 설정했습니다. 처음 배포할 때 과거 이벤트를 다시 분배하지 않도록 consumer의 `auto.offset.reset`은 `latest`로 두었습니다.

## 3. 트리거 2종 {#trigger}

두 트리거는 호출하는 API만 다르고 구조가 같습니다. 보강 트리거는 엔티티 한 종류를 더 처리합니다.

```
put(records):
  for record in records:
    event = 변환(record)               # 실패 시 skippable 설정에 따라 버림
    if event.작업코드 not in (c, m): continue
    요청 키 = 고유 키 집합 → gRPC 요청 키   # 키 형태 두 가지 중 하나로
    try:
      API 호출 (blocking, deadline 적용)
    except gRPC 실패:
      DLQ로 보고 (ErrantRecordReporter)
```

<div class="table-wrap" markdown="1">

| 항목 | 정합성 보정 트리거 | 콘텐츠 보강 트리거 |
| --- | --- | --- |
| 소비 토픽 | `event.content` | `event.enhance` |
| `tasks.max` | 6 | 72 (2025-01에 12에서 증설) |
| 호출 deadline | 5초 (기본값 1초) | 120초 |
| 호출 방식 | 레코드마다 동기 호출 (blocking stub) | 같음 |
| 전송 | 포트 443이면 TLS, 아니면 평문 | 같음 |

</div>

**보강 트리거의 120초 deadline**: 보강 API는 요청 하나에 상당한 작업 시간이 걸릴 수 있다는 API 개발자의 의견을 받아, 처리 도중 요청이 끊기지 않도록 deadline을 충분히 길게 잡았습니다.

**보강 트리거의 병렬도 증설(2025-01)**: 레코드마다 응답을 기다리는 구조라 처리량은 병렬도로 확보했습니다. 소비 토픽의 파티션을 늘리고 `tasks.max`를 12에서 72로 늘렸습니다. 파티션 수를 바꾸면 키가 배정되는 파티션이 달라지므로, 증설 시점 전후로 같은 키의 이벤트가 서로 다른 파티션에 놓여 그 사이의 처리 순서가 보장되지 않습니다. 이 일시적인 순서 문제는 서비스 운영 측과 상의해 받아들일 수 있다고 판단했습니다.

**gRPC 연결 유지**: 2024-05에 채널에 keep-alive(5분 간격, 20초 대기)를 고정값으로 넣었다가, 2024-12에 켜고 끄는 여부와 두 값을 커넥터 설정으로 뺐습니다. 기본값은 꺼짐입니다.

**호출 실패 처리의 변화**: 처음에는 gRPC 호출이 실패하면 오류 로그만 남기고 다음 레코드로 넘어갔습니다. 실패한 레코드의 offset도 그대로 커밋되어, 실패한 변경은 로그에만 남았습니다. 2025-01에 다음처럼 바꿨습니다.

- API 클라이언트가 gRPC 실패를 전용 예외로 감싸 던집니다.
- task는 이 예외만 잡아 Kafka Connect의 `ErrantRecordReporter`로 해당 레코드를 DLQ 토픽에 보고하고, 다음 레코드를 처리합니다.
- 커넥터 설정은 `errors.tolerance=all`, DLQ 토픽 지정, `errors.deadletterqueue.context.headers.enable=true`로 둡니다. Connect 런타임의 변환 단계 실패도 같은 DLQ로 가고, 오류 맥락은 헤더에 남습니다.

레코드 하나의 호출 실패 때문에 task 전체를 멈추지 않는다는 원칙은 그대로 두고, 실패한 레코드를 로그가 아니라 토픽에 남겨 다시 처리할 수 있게 했습니다. 다만 이 경로는 gRPC 실패에만 적용했습니다(회고 [E2](#e2)).

## 4. 수집 지연 측정 커넥터 {#latency}

"이벤트가 늦게 온다"는 말이 나올 때 원본에서 늦는지, 그 뒤에서 늦는지 가를 숫자가 필요했습니다. CDC 토픽을 별도 커넥터(별도 consumer group)로 소비해 지연을 측정했습니다.

<div class="table-wrap" markdown="1">

| 지표 | 계산 | 보는 것 |
| --- | --- | --- |
| 원본 기준 지연 | Kafka 레코드 timestamp − `source.ts_ms` (원본 DB 변경 시각) | 원본 변경에서 Kafka 기록까지 전체 |
| Debezium 기준 지연 | Kafka 레코드 timestamp − `ts_ms` (Debezium 처리 시각) | Debezium이 이벤트를 만든 뒤 Kafka 기록까지 |

</div>

두 지표를 나눠 두면, 원본 기준 지연만 크고 Debezium 기준 지연이 작을 때 원인이 change stream을 읽는 쪽에 있다고 좁힐 수 있습니다.

- **집계 단위**: task에 할당된 파티션마다 집계기를 두고, 그 안에서 원본 DB별로 평균·최대·최소·건수를 계산합니다.
- **구간**: 10분(`rotate.interval.ms=600000`). UTC 0시부터 10분 단위로 나누고, 구간 판단에는 벽시계가 아니라 **레코드의 Kafka timestamp**를 씁니다. 그래서 밀린 이벤트를 나중에 읽어도 원래 구간으로 집계됩니다.
- **출력**: 다음 구간의 첫 레코드가 들어오면 이전 구간의 결과를 worker 로그와 CloudWatch Logs에 한 줄로 남깁니다.
- 처음 배포할 때 과거 이벤트로 지표를 만들지 않도록 `auto.offset.reset=latest`로 시작합니다.

## 5. 당시 가정과 확인한 사실 {#assumptions}

<div class="table-wrap" markdown="1">

| 당시 가정 | 확인한 사실 | 그래서 |
| --- | --- | --- |
| `flush()`는 offset을 커밋하는 단계라 비워 둬도 런타임이 처리한다 (커넥터 README에도 그렇게 적음) | 런타임은 `flush()`가 **정상 반환되면** 그때까지 받은 레코드를 처리 완료로 보고 offset을 커밋합니다. 플러그인이 비동기로 내보낸 것이 끝났는지는 플러그인이 `flush()`에서 확인해야 합니다 | E1 |
| `send()`를 재시도로 감싸면 발행 실패에 대비된다 | `send()`는 비동기라 재시도 루프는 호출 자체의 예외만 잡습니다. 브로커 쓰기 실패는 콜백으로만 옵니다 | E1 |
| 콜백에서 예외를 던지면 task가 실패한다 | 콜백은 producer의 I/O 스레드에서 실행됩니다. 예외는 그 스레드에서 로그로 남을 뿐 task로 전달되지 않습니다 | E1 |
| `errors.tolerance=all`이면 레코드 처리 실패가 모두 DLQ로 간다 | 이 설정은 런타임의 변환·transform 단계에만 적용됩니다. `put()` 안의 실패는 플러그인이 `ErrantRecordReporter`로 직접 보고해야 합니다 | E2 |

</div>

## 6. 회고: 다시 읽고 찾은 결함 {#retro}

<div class="findings" markdown="1">

<div class="finding" markdown="1">

### <span class="pill bad">높음</span> <span class="id">E1</span> 분배 커넥터의 비동기 발행 실패가 offset 커밋을 막지 못합니다 {#e1}

분배 task (발행·flush 경로)
{: .where}

`put()`은 분배마다 producer의 `send()`를 비동기로 호출하고 돌아오며, `flush()`는 비어 있습니다. 런타임은 `flush()`가 반환되면 그때까지의 offset을 커밋하므로, 브로커가 아직 확인하지 않은 메시지의 offset도 커밋될 수 있습니다. 브로커 쓰기가 최종 실패하면 콜백이 예외를 던지지만 I/O 스레드에서 끝나고, `acks=all`로 설정했어도 그 메시지는 다시 처리되지 않습니다. producer의 `flush()`는 task가 멈출 때만 호출했습니다.

**개선 방향** `flush()`(또는 `preCommit()`)에서 모든 producer의 `flush()`를 호출해 미확인 메시지를 기다립니다. 콜백은 예외를 던지지 않고 첫 실패를 필드에 기록해 두고, `flush()`나 다음 `put()`에서 그 실패를 예외로 올려 offset 커밋을 막습니다. **상태** 이 글을 쓰며 찾았습니다. 분배 커넥터 운영을 종료(2025-09)해 적용하지 않았습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill warn">중간</span> <span class="id">E2</span> 트리거는 API 실패만 격리하고, 키 변환 실패에서는 멈춥니다 {#e2}

트리거 task (`put()`)
{: .where}

DLQ 보고는 gRPC 호출을 감싼 `try`에만 있습니다. 그 앞의 키 변환(고유 키 집합 → 요청 키)에서 필드가 없거나 숫자로 바뀌지 않으면 예외가 `put()` 밖으로 나가 task가 실패합니다. "레코드 하나 때문에 전체를 멈추지 않는다"는 원칙이 API 실패에만 적용된 셈입니다. 또 DLQ가 설정되지 않은 환경에서는 실패를 아무 기록 없이 넘깁니다.

**개선 방향** 레코드 하나를 처리하는 전체(변환, 키 변환, 호출)를 하나의 `try`로 감싸고, 실패하면 `ErrantRecordReporter`로 보고합니다. reporter가 없으면 task를 실패시켜 설정 누락을 바로 드러냅니다. **상태** 적용하지 않았습니다.

</div>

<div class="finding" markdown="1">

### <span class="pill info">낮음</span> <span class="id">E3</span> 지연 측정은 다음 이벤트가 와야 구간을 닫습니다 {#e3}

지연 측정 task (구간 출력)
{: .where}

구간 종료를 다음 구간의 레코드로 판단하므로, 이벤트가 뜸한 파티션은 마지막 구간의 결과가 늦게, 혹은 다음 이벤트가 올 때까지 나오지 않습니다. 결과가 파티션별로 따로 남아서, DB 전체의 지연을 보려면 파티션별 줄을 다시 합쳐야 합니다. 결과가 지표가 아니라 로그 한 줄이라 경보를 걸려면 metric filter를 따로 만들어야 합니다.

**개선 방향** 빈 `put()` 호출 때 벽시계로 지난 구간을 닫고, 결과를 CloudWatch 지표(`PutMetricData`, 차원: DB)로 바로 기록합니다. **상태** 적용하지 않았습니다.

</div>

</div>

> **배운 점**
>
> Kafka Connect 플러그인을 만들면 offset 관리와 오류 처리를 런타임에 맡길 수 있다고 생각했습니다. 하지만 런타임은 `flush()`의 반환을 "여기까지 처리 끝"으로 믿을 뿐이고, `errors.tolerance`도 플러그인 바깥 단계에만 적용됩니다. 비동기로 내보낸 것의 완료와 `put()` 안의 실패는 플러그인이 책임져야 했습니다. 런타임에 맡긴 것과 플러그인이 지켜야 하는 것의 경계를 먼저 그었어야 했습니다.
{: .callout .info}

## 7. 성과와 기대 효과 {#outcome}

### 운영에 적용한 것

<div class="table-wrap" markdown="1">

| 무엇이 바뀌었나 | 이전 | 적용 후 |
| --- | --- | --- |
| 하류 API에 전달되는 변경 | — (새로 만듦) | 실제로 바뀐 생성·수정만, 하류 키로 묶여 용도별 토픽에 (2024-01) |
| 분배 대상 추가 | 출력 토픽 하나 | 설정 묶음 하나 추가 (2024-04) |
| 한 분배의 키 설정 오류 | 다른 분배에 영향 가능 | 그 분배만 건너뜀 (2024-09) |
| 보강 트리거 병렬도 | task 12 | 파티션 증설과 함께 task 72. 증설 시점의 같은 키 순서 어긋남은 운영 측과 합의해 감수 (2025-01) |
| 트리거 호출 실패 | 로그만 남고 레코드는 지나감 | DLQ 토픽에 레코드와 오류 맥락 보관 (2025-01) |
| 수집 지연 | 측정 수단 없음 | 파티션·DB별 10분 구간의 평균·최대·최소·건수, 원본 기준과 Debezium 기준으로 분리 (2025-06) |

</div>

모두 코드와 변경 이력으로 확인한 내용입니다. 적용 전후의 실측값(호출 수, 실패 건수, 지연 수치)은 이 기록에 없습니다.

### 회고 항목을 적용하면 얻는 것

아래는 적용하지 않았습니다. 효과는 코드 구조로 판단한 예상입니다.

<div class="table-wrap" markdown="1">

| 개선 작업 | 무엇이 바뀌나 | 지금 | 적용 후 (예상) | 근거 |
| --- | --- | --- | --- | --- |
| **`flush()`에서 발행 완료를 기다리고 실패를 올리기** (E1) | 브로커 쓰기 실패 시 이벤트 유실 | offset이 커밋되어 다시 처리되지 않음 | 커밋이 막히고 task가 재처리 | 6장 E1 |
| **레코드 처리 전체를 DLQ 경로로** (E2) | 키가 이상한 레코드 하나의 영향 | task 정지, 해당 토픽 전체 지연 | 그 레코드만 DLQ로 | 6장 E2 |
| **구간을 벽시계로 닫고 지표로 기록** (E3) | 이벤트가 뜸할 때 지표 공백, 경보 설정 | 다음 이벤트까지 대기, metric filter 필요 | 10분마다 지표, 바로 경보 가능 | 6장 E3 |

</div>

## 8. 남은 과제 {#next}

- **검증할 항목: 동기 호출과 poll 간격**. 보강 트리거는 레코드마다 최대 120초를 기다리는 동기 호출을 합니다. 한 번에 받은 레코드 여러 개가 느려지면 다음 poll까지의 간격이 consumer의 `max.poll.interval.ms`(기본 5분)를 넘을 수 있습니다. 넘으면 consumer group에서 빠지고 리밸런싱이 일어납니다. 운영 중 이런 리밸런싱이 있었는지는 이 기록으로 확인하지 못했습니다. 확인하려면 worker 로그에서 poll 간격 초과 경고를 찾고, 대응으로는 `consumer.override.max.poll.records`를 낮추거나 호출을 비동기로 바꾸는 방법이 있습니다.
- **분배 토픽의 순서 범위**. 분배 토픽 안에서는 같은 하류 키가 같은 파티션에 모이지만, 원본 토픽의 서로 다른 파티션에서 온 이벤트(예: 같은 상위 엔티티에 속한 하위 문서들) 사이의 순서는 보장되지 않습니다. 하류 API가 원본을 다시 조회해 처리하는 방식이라면 문제가 되지 않지만, 이벤트 내용에 의존하는 하류가 붙는다면 이 전제를 다시 봐야 합니다.

## 9. 참고 {#refs}

- 같은 사례의 다른 글: [쿠버네티스 위 Kafka Connect 클러스터 구성과 오픈소스 패치](/2026/10/23/kafka-connect-platform.html)
- [Apache Kafka: SinkTask (flush, preCommit)](https://kafka.apache.org/documentation/#connect_developing)
- [KIP-610: Error Reporting in Sink Connectors (ErrantRecordReporter)](https://cwiki.apache.org/confluence/display/KAFKA/KIP-610%3A+Error+Reporting+in+Sink+Connectors)
- [Kafka Connect: errors.tolerance와 DLQ 설정](https://kafka.apache.org/documentation/#sinkconnectorconfigs)
- [Apache Kafka: Producer 설정 (acks, enable.idempotence)](https://kafka.apache.org/documentation/#producerconfigs)
- [Apache Kafka: Consumer 설정 (max.poll.interval.ms)](https://kafka.apache.org/documentation/#consumerconfigs)
- [Debezium MongoDB connector: change event (op, updateDescription, source.ts_ms)](https://debezium.io/documentation/reference/stable/connectors/mongodb.html)
- [gRPC: Deadlines](https://grpc.io/docs/guides/deadlines/) · [Keepalive](https://grpc.io/docs/guides/keepalive/)
