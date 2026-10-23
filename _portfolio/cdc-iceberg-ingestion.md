---
layout: post
title: CDC 이벤트를 Iceberg에 적재하는 파이프라인
date: 2026-10-22
tags: [Iceberg, Kafka Connect, CDC, Spark]
author: caimask
---

MongoDB CDC 이벤트를 Kafka에서 Apache Iceberg로 적재하는 Kafka Connect sink를 직접 만들어 운영했습니다. 적재가 느려지자 원인을 Iceberg의 커밋 경쟁으로 짚었고, buffer sink 커넥터와 Spark 배치로 적재 구조를 바꿔 해소했습니다. 이후 바꾼 구조도 다시 검토해 다음 과제를 정리했습니다.

역할: 커넥터 설계·개발(2023-08 ~ 2024-12), buffer sink 커넥터 + Spark 배치 설계·개발과 전환, 회고 문서 작성(2026-10)
{: .meta}

기술: Kafka Connect · Debezium(MongoDB) · Apache Iceberg(Glue Catalog) · Spark · S3 · Kyuubi
{: .meta}

<div class="summary" markdown="1">

## 한눈에 보기 {#tldr}

- **문제**: 같은 Iceberg 테이블에 writer 최대 8개가 몇 초마다 커밋하면서 재시도가 폭주했고, 작은 파일과 delete 파일이 쌓여 최적화 작업이 따라가지 못했습니다.
- **판단**: lock 문제가 아니라 낙관적 동시성 제어(OCC)의 커밋 경쟁이었습니다. 해법은 lock을 세밀하게 만드는 것이 아니라 **테이블당 커밋 주체와 커밋 횟수를 줄이는 것**이었습니다.
- **결과**: 테이블당 커밋 주체 8 → 1, 커밋 몇 초마다 → 10분에 2~3회, equality delete 0. 대가로 최신성이 수 초에서 15~25분이 됐습니다.

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

| 항목 | 이전 (upsert 커넥터) | 전환 후 (buffer + Spark 배치) |
| --- | --- | --- |
| 테이블당 커밋 주체 | 최대 8개 task | <span class="pill ok">배치 job 1개</span> |
| 커밋 빈도 | 몇 초마다 | <span class="pill ok">10분에 테이블당 2~3회</span> |
| equality delete | 레코드마다 | <span class="pill ok">0</span> |
| 재처리 | upsert 멱등성에 의존 | <span class="pill ok">결정적 파일명 + 파티션 덮어쓰기</span> |
| 원본 보존 | 없음 | <span class="pill ok">S3 버퍼에 Parquet로 보관</span> |
| 최신성 | 수 초 (경합이 없을 때) | <span class="pill warn">15~25분</span> 대가로 내준 것 |

</div>

## 다음 과제 {#next}

바꾼 구조를 다시 검토해 찾은 항목입니다. 수정안을 정리했고, 아직 적용하지 않았습니다.

- **넘겨받는 지점의 누락 위험**: 배치가 벽시계 시간으로 윈도우 완료를 판단해, 늦게 도착한 버퍼 파일이 처리되지 않을 수 있습니다. → 윈도우 단위가 아니라 파일 단위로 증분 처리합니다.
- **정합성**: mirror `MERGE`의 원본 선택 쿼리와 변경 순서 값의 인코딩에서 변경이 빠지거나 순서가 뒤바뀔 수 있습니다. → 키별 최신 1건을 고르고 버전을 비교하며, 순서 값을 다시 인코딩합니다.
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
