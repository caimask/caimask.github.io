---
layout: post
title: CDC 이벤트를 Iceberg에 적재하는 파이프라인
date: 2026-10-22
tags: [Iceberg, Kafka Connect, CDC, Spark, Debezium, Kubernetes]
author: caimask
---

MongoDB CDC 이벤트를 Kafka에서 Apache Iceberg로 적재하는 Kafka Connect sink를 직접 만들어 운영했습니다. 적재가 느려지자 원인을 Iceberg의 커밋 경쟁으로 짚었고, buffer sink 커넥터와 Spark 배치로 적재 구조를 바꿔 해소했습니다. 이후 바꾼 구조도 다시 검토해 다음 과제를 정리했습니다. 이 파이프라인이 도는 Kafka Connect 클러스터를 EKS에 구성하고, 수집 구간에서 설정으로 풀리지 않던 문제는 오픈소스 코드를 고쳐 이미지에 넣었습니다.

역할: 커넥터 설계·개발(2023-08 ~ 2024-12), buffer sink 커넥터 + Spark 배치 설계·개발과 전환, Kafka Connect 클러스터(EKS·Helm) 구성과 오픈소스 패치, 회고 문서 작성(2026-10)
{: .meta}

기술: Kafka Connect · Debezium(MongoDB) · Schema Registry · Apache Iceberg(Glue Catalog) · Spark · S3 · Kyuubi · Kubernetes(EKS) · Helm
{: .meta}

<div class="summary" markdown="1">

## 한눈에 보기 {#tldr}

- **문제**: 같은 Iceberg 테이블에 writer 최대 8개가 몇 초마다 커밋하면서 재시도가 폭주했고, 작은 파일과 delete 파일이 쌓여 최적화 작업이 따라가지 못했습니다.
- **판단**: lock 문제가 아니라 낙관적 동시성 제어(OCC)의 커밋 경쟁이었습니다. 해법은 lock을 세밀하게 만드는 것이 아니라 **테이블당 커밋 주체와 커밋 횟수를 줄이는 것**이었습니다.
- **결과**: 테이블당 커밋 주체 8 → 1, 커밋 몇 초마다 → 10분에 2~3회, equality delete 0. 대가로 최신성이 수 초에서 15~25분이 됐습니다.
- **운영 기반**: Schema Registry 일시 실패로 task가 내려가던 문제와 Debezium 필드 필터의 제약을 오픈소스 코드 수정으로 풀었습니다.

</div>

## 문제 {#problem}

변경을 빨리 반영하려고 커넥터가 `put()`이 호출될 때마다 테이블별로 커밋하도록 만들었습니다. 그런데 주요 mirror 테이블에는 두 커넥터의 task가 함께 쓰고 있어서, 한 테이블에 최대 8개 writer가 몇 초 간격으로 커밋을 다퉜습니다. 커밋에서 진 writer는 메타데이터를 다시 읽고 다시 쓰고 기다리기를 반복했고, 커밋마다 작은 data 파일과 equality delete 파일이 생겼습니다.

## 핵심 판단 {#decisions}

<div class="table-wrap" markdown="1">

| 당시 가정 | 확인한 사실 | 그래서 한 일 |
| --- | --- | --- |
| table lock 때문에 느리고, "Iceberg도 row lock이 된다"면 업그레이드로 풀린다 | Iceberg는 lock 없이 OCC로 커밋하고, 커밋은 테이블당 한 번에 하나씩 반영됩니다. row-level concurrency는 Delta Lake 기능이고, 커밋 직렬화를 없애지 않습니다 | 커밋 경쟁 자체를 없애는 방향으로 설계를 바꿈 |
| 커밋을 묶으면 중간 변경 이력이 사라진다 | 변경 이력은 스냅샷이 아니라 log 테이블의 **행**으로 보존됩니다. 커밋 주기와 상관이 없습니다 | 커밋 주기를 10분으로 늘림 |
| 처리량을 높이려면 커밋을 자주 해야 한다 | 커밋은 이미 쓴 파일 목록을 등록하는 단계라, 처리량이 아니라 **최신성의 하한**을 정합니다 | 쓰기는 병렬로, 커밋은 한 곳에서 |

</div>

이 판단을 바탕으로 **"append 먼저, merge는 나중에"** 구조로 옮겼습니다. 이벤트를 10분 단위 Parquet 파일로 S3에 버퍼링하고, Spark 배치 job 하나가 log 테이블에는 append하고 mirror 테이블에는 `MERGE`로 반영합니다.

옮기기 전에 공개된 외부 사례(토스증권의 Iceberg CDC 적재기)와 비교했습니다. 그 결과, 커넥터를 유지한 채 커밋만 묶으면 스키마 변경 때 같은 커밋 안에서 중복 행이 생긴다는 점을 적용 전에 찾았습니다.

## 결과 {#result}

<div class="table-wrap" markdown="1">

| 항목 | 이전 (upsert 커넥터) | 전환 후 (buffer + Spark 배치) | 평가 |
| --- | --- | --- | --- |
| 테이블당 커밋 주체 | 최대 8개 task | 배치 job 1개 | <span class="pill ok">개선</span> |
| 커밋 빈도 | 몇 초마다 | 10분에 테이블당 2~3회 | <span class="pill ok">개선</span> |
| equality delete | 레코드마다 | 0 | <span class="pill ok">개선</span> |
| 재처리 | upsert 멱등성에 의존 | 결정적 파일명 + 파티션 덮어쓰기 | <span class="pill ok">개선</span> |
| 원본 보존 | 없음 | S3 버퍼에 Parquet로 보관 | <span class="pill ok">개선</span> |
| 최신성 | 수 초 (경합이 없을 때) | 15~25분 | <span class="pill warn">감수</span> 커밋 경합을 없앤 대가 |

</div>

## 운영 기반: Kafka Connect 클러스터와 오픈소스 패치 {#platform}

위 커넥터들은 공용 EKS에 직접 구성한 Kafka Connect 클러스터(distributed 모드)에서 돌았습니다. 클러스터는 Helm chart로 구성하고 환경마다 values 파일만 달리 뒀습니다.

### 이미지와 플러그인을 나눈 구조 {#image}

직접 만든 커넥터는 자주 고쳐 배포하지만, 런타임과 그 라이브러리는 거의 바뀌지 않습니다. 그래서 공식 이미지(`cp-kafka-connect` 7.2)를 다시 빌드할 때는 런타임 수정만 넣고, 커넥터 플러그인은 pod가 뜰 때 설치하게 했습니다.

```
[이미지]  cp-kafka-connect 7.2
          + JSON 로그 layout jar (log4j)
          + 재빌드한 Avro Converter jar        ← 공식 jar 자리에 덮어씀
          + 재빌드한 Debezium MongoDB zip      ← Confluent Hub zip 형식 그대로

[pod 시작]
  init container ─ Confluent Hub 플러그인 설치 + 사내 Maven 저장소에서 커넥터 jar 내려받기
                   └─ 공유 볼륨 → worker의 plugin.path
```

커넥터를 새로 배포할 때 이미지를 다시 빌드하지 않습니다. 이 밖에 worker 설정은 values에서 환경 변수로 주입하고, 비밀 값은 Secret 볼륨과 `DirectoryConfigProvider`로 커넥터 설정 밖에 두고, log4j 설정은 ConfigMap으로, 노드 교체 중 가용성은 PodDisruptionBudget으로 다뤘습니다.

### 설정으로 풀리지 않은 문제 두 가지 {#patches}

<div class="table-wrap" markdown="1">

| 문제 | 원인 | 수정 |
| --- | --- | --- |
| Schema Registry에 일시적으로 접근하지 못하면 task가 `FAILED`로 내려가 사람이 재시작 | Connect 런타임은 converter 예외가 `RetriableException`일 때만 `errors.retry.timeout` 동안 재시도하는데, Avro Converter는 직렬화 실패를 모두 `DataException`으로 던짐 | 같은 버전 tag에서 `AvroConverter`의 예외를 `RetriableException`으로 바꿔 재빌드. 커넥터에 재시도 타임아웃 10초 지정 |
| 큰 본문 필드 때문에 CDC 메시지가 커지는데, `@`로 시작하는 필드는 `field.exclude.list`에 넣으면 설정 검증에서 실패 | MongoDB는 필드명에 `@`를 허용하지만, Debezium의 설정 검증 정규식이 허용하지 않음 | 정규식의 필드명 문자 집합에 `@` 추가 |
| 필드를 제외하면 그 필드가 바뀌어도 `updatedFields`에서 함께 빠짐. 하류는 값이 아니라 **바뀌었다는 사실**이 필요 | 필터가 문서 본문과 변경 필드 목록에 같은 제거 규칙을 적용 | 새 옵션 `custom.empty.updated.fields`를 켜면 제외 필드도 변경 목록에 이름을 남기고 값은 빈 문자열로 바꿈. 끄면 기존 동작 |

</div>

재빌드한 jar는 공식 이미지의 같은 경로에 같은 버전 번호로 덮어썼고, Debezium jar는 Confluent Hub의 같은 버전 zip에 넣어 다시 묶었습니다. 다른 jar와의 호환을 지키고 차이를 수정 몇 줄로 한정하려는 선택이었습니다. 대신 버전 번호만으로는 공식 jar와 구분되지 않으므로, 패치마다 원본 저장소·tag·수정한 파일과 메서드·수정 전후 코드·빌드 절차를 이미지 저장소의 README에 남겼습니다.

재시도 타임아웃을 다 쓰고도 실패하면 레코드를 건너뛰지 않고 task가 실패하도록 `errors.tolerance`는 기본값(`none`)으로 뒀습니다. 스키마가 실제로 맞지 않는 영구 실패도 10초 동안 재시도한 뒤 실패하는데, 이 지연은 감수할 만하다고 판단했습니다.

## 다음 과제 {#next}

바꾼 구조를 다시 검토해 찾은 항목입니다. 수정안을 정리했고, 아직 적용하지 않았습니다.

- **넘겨받는 지점의 누락 위험**: 배치가 벽시계 시간으로 윈도우 완료를 판단해, 늦게 도착한 버퍼 파일이 처리되지 않을 수 있습니다. → 윈도우 단위가 아니라 파일 단위로 증분 처리합니다.
- **정합성**: mirror `MERGE`의 원본 선택 쿼리와 변경 순서 값의 인코딩에서 변경이 빠지거나 순서가 뒤바뀔 수 있습니다. → 키별 최신 1건을 고르고 버전을 비교하며, 순서 값을 다시 인코딩합니다.
- **패치 이식**: Connect 런타임과 Debezium을 올리려면 두 패치를 새 버전에 다시 적용해야 합니다. 상위 버전에서 같은 문제가 고쳐졌는지 먼저 보고, 남는 패치만 옮기는 것이 순서입니다.
- **시간이 갈수록 느려지는 구조**: `MERGE`의 log 전체 스캔, 파티션 없는 copy-on-write mirror, 10분 단위 파티션. → 윈도우 범위만 읽고, 일 파티션과 merge-on-read를 검토합니다.

> **배운 점**
>
> 느려지는 원인은 한 번의 무거운 작업이 아니라, 가벼워 보이는 커밋이 "writer 수 × 몇 초마다"로 곱해지는 데 있었습니다. 코드를 볼 때는 무엇을 하느냐보다 **언제, 몇 번 실행되느냐**를 먼저 봅니다.
{: .callout .info}

## 상세 문서 {#docs}

<div class="table-wrap" markdown="1">

| 문서 | 내용 |
| --- | --- |
| [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html) | 커넥터가 느렸던 원인, OCC와 lock의 차이, 삭제 방식 비교, 테이블 운영 최적화, 직접 구현한 이유 |
| [Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html) | 전환한 구조의 성과와 자기 검토로 찾은 결함(B1~B6, S1~S10), 개선안과 우선순위 |
| [토스 Iceberg CDC 사례 비교](/2026/10/20/toss-iceberg-cdc-compare.html) | 외부 사례와 비교, 커밋을 묶을 때 생기는 정합성 문제와 구현 경로 |

</div>
