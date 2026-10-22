---
layout: post
title: Column-oriented Database
tags: [Database, Parquet, Iceberg]
author: caimask
---

"대용량 분석에 좋다는 컬럼 DB는 무엇일까?"라는 질문만 적고 참고 링크로 끝났던 2020년 글을 채웠습니다. 행 저장과 컬럼 저장의 차이, 컬럼 저장이 분석에서 빠른 이유와 대가, 그리고 그 대가가 데이터 레이크(Parquet, Iceberg)에서 작은 파일·삭제 파일·컴팩션 문제로 어떻게 나타나는지 정리했습니다.

원문 2020-08-27 · 보강 2026-10-21 · 연관: [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#deletes) · [Buffer Writer 배치 적재 리뷰](/2026/10/15/buffer-writer-batch-review.html)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **차이는 디스크에 값을 놓는 순서입니다.** 행 저장은 한 행의 컬럼들을 붙여 놓고, 컬럼 저장은 한 컬럼의 값들을 붙여 놓습니다.
2. **분석 쿼리는 많은 행의 적은 컬럼을 읽습니다.** 컬럼 저장은 필요한 컬럼만 읽고, 같은 종류의 값이 모여 있어 압축이 잘 되며, 블록별 통계로 읽을 필요 없는 블록을 건너뜁니다.
3. **대가는 행 단위 변경입니다.** 한 행을 고치려면 여러 컬럼 블록을 다시 써야 합니다. 그래서 컬럼 저장소는 수정 대신 "새로 쓰고 나중에 합치는" 방식을 씁니다.
4. **HBase는 컬럼 저장소가 아닙니다.** 이름이 비슷한 wide-column 저장소로, 행 키 기준 조회에 맞춰져 있습니다.
5. **데이터 레이크에서는 이 대가가 운영 문제로 돌아옵니다.** 자주 커밋하면 작은 Parquet 파일이, upsert하면 삭제 파일이 쌓이고, 이것을 컴팩션으로 다시 합쳐야 합니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [행 저장과 컬럼 저장](#layout)
2. [컬럼 저장이 분석에서 빠른 이유](#fast)
3. [컬럼 저장의 대가](#cost)
4. [헷갈리기 쉬운 것: wide-column 저장소](#wide)
5. [Parquet 파일 구조](#parquet)
6. [Iceberg에서 다시 만난 컬럼 저장](#iceberg)
7. [참고](#refs)

</div>

## 1. 행 저장과 컬럼 저장 {#layout}

같은 주문 테이블을 두 방식으로 디스크에 놓으면 이렇게 됩니다.

```
주문 테이블
| id | user | amount | status |
|  1 | kim  |  3000  | DONE   |
|  2 | lee  |  5000  | READY  |
|  3 | park |  1000  | DONE   |

행 저장 (MySQL InnoDB 등)
[1, kim, 3000, DONE][2, lee, 5000, READY][3, park, 1000, DONE]

컬럼 저장 (Parquet, ClickHouse, Redshift, BigQuery 등)
id:     [1, 2, 3]
user:   [kim, lee, park]
amount: [3000, 5000, 1000]
status: [DONE, READY, DONE]
```

<div class="table-wrap" markdown="1">

|   | 행 저장 | 컬럼 저장 |
| --- | --- | --- |
| 잘하는 쿼리 | `WHERE id = 2`로 한 행 전체 조회 | `SELECT sum(amount) WHERE status = 'DONE'` |
| 한 행 쓰기·수정 | 한 곳만 씀 | 컬럼 수만큼 여러 곳을 씀 |
| 대량 집계 | 필요 없는 컬럼까지 모두 읽음 | 필요한 컬럼만 읽음 |
| 압축 | 행 안의 값들이 서로 다른 타입이라 낮음 | 같은 타입의 비슷한 값이 모여 높음 |
| 대표 용도 | OLTP (서비스 DB) | OLAP (분석, 데이터 웨어하우스) |

</div>

## 2. 컬럼 저장이 분석에서 빠른 이유 {#fast}

1. **필요한 컬럼만 읽습니다(column pruning).** 컬럼이 100개인 테이블에서 3개만 쓰는 쿼리는 데이터의 약 3%만 읽습니다.
2. **압축이 잘 됩니다.** `status`처럼 종류가 적은 값은 dictionary 인코딩(값을 작은 번호로 바꿈)이나 run-length 인코딩(같은 값의 반복을 개수로 표현)으로 크게 줄어듭니다. 읽을 바이트가 줄면 I/O가 줄어듭니다.
3. **블록을 통째로 건너뜁니다(predicate pushdown).** 블록마다 컬럼별 최솟값과 최댓값을 기록해 두면, `amount > 10000` 조건에서 최댓값이 5000인 블록은 읽지 않습니다.
4. **CPU가 한 번에 여러 값을 처리합니다(vectorized execution).** 같은 타입의 값이 연속으로 놓여 있어 한 번에 묶어 계산하기 좋습니다.

> **건너뛰기는 정렬에 달려 있습니다**
>
> 최솟값·최댓값 통계는 값이 블록마다 고르게 섞여 있으면 쓸모가 없습니다. 모든 블록의 범위가 비슷해지기 때문입니다. 자주 거는 조건 컬럼으로 정렬하거나 파티션을 나눠 쓰면 건너뛰는 비율이 크게 오릅니다.
{: .callout .info}

## 3. 컬럼 저장의 대가 {#cost}

한 행을 고치려면 그 행의 값이 흩어져 있는 모든 컬럼 블록을 고쳐야 하고, 압축된 블록은 부분 수정이 어렵습니다. 그래서 컬럼 저장소는 대부분 **불변 파일 + 나중에 합치기** 구조를 씁니다.

<div class="table-wrap" markdown="1">

| 방식 | 하는 일 | 예 |
| --- | --- | --- |
| 새 파일에 쓰고 백그라운드에서 병합 | insert는 작은 파트로 쓰고 주기적으로 합침 | ClickHouse MergeTree |
| 삭제 표시 + 나중에 정리 | 지울 행을 별도로 기록하고 읽을 때 걸러냄 | Iceberg delete 파일, Delta deletion vector |
| 파일 통째로 다시 쓰기 | 바뀐 행이 든 파일을 새로 씀 | Iceberg·Delta copy-on-write |

</div>

어느 방식이든 비용이 사라지지는 않고, 쓰기 시점에 낼지 읽기와 정리 시점에 낼지를 고르는 것입니다.

## 4. 헷갈리기 쉬운 것: wide-column 저장소 {#wide}

HBase나 Cassandra도 흔히 "컬럼 DB"로 묶여 불리지만, 저장 방식과 잘하는 일이 다릅니다.

<div class="table-wrap" markdown="1">

|   | 컬럼 저장 (columnar) | wide-column 저장소 |
| --- | --- | --- |
| 예 | Parquet, ClickHouse, Redshift | HBase, Cassandra, Bigtable |
| 물리 배치 | 컬럼 하나의 값들을 모음 | 행 키 순으로 정렬, column family 단위로 파일을 나눔 |
| 잘하는 일 | 대량 스캔과 집계 | 행 키로 빠른 조회와 쓰기 |
| "컬럼"의 의미 | 테이블 스키마의 컬럼 | 행마다 다를 수 있는 key-value 쌍 |

</div>

HBase 위에 SQL을 얹는 Phoenix도 HBase의 행 키 조회 성능을 활용하는 것이지, 컬럼 저장의 스캔 성능을 주는 것은 아닙니다.

## 5. Parquet 파일 구조 {#parquet}

데이터 레이크에서 가장 많이 쓰는 컬럼 저장 파일 형식입니다.

```
Parquet 파일
├── Row Group 1 (보통 128MB 안팎)
│   ├── Column Chunk: id      → Page, Page, ...  (min/max, null 수)
│   ├── Column Chunk: user    → Page, Page, ...
│   └── Column Chunk: amount  → Page, Page, ...
├── Row Group 2
│   └── ...
└── Footer: 스키마, 각 row group·column chunk의 위치와 통계
```

- 읽는 쪽은 footer부터 읽어 필요한 row group과 column chunk만 골라 읽습니다.
- row group이 너무 작으면 통계와 메타데이터 비중이 커지고 압축률이 떨어집니다. **작은 파일이 많으면 느린 이유**가 여기 있습니다.

## 6. Iceberg에서 다시 만난 컬럼 저장 {#iceberg}

Iceberg는 Parquet 파일 위에 "어떤 파일들이 지금 테이블인가"를 관리하는 테이블 계층을 얹습니다. manifest에 파일마다 컬럼별 최솟값·최댓값을 기록해, 파일을 열기도 전에 건너뜁니다. 컬럼 저장의 장점을 파일 단위로 한 번 더 쓰는 셈입니다.

그리고 3장의 대가가 CDC 적재에서 그대로 나타났습니다.

<div class="table-wrap" markdown="1">

| 컬럼 저장의 성질 | CDC → Iceberg 적재에서 생긴 일 | 정리한 문서 |
| --- | --- | --- |
| 작은 파일은 비효율적 | 몇 초마다 커밋하자 커밋마다 작은 data·delete·manifest 파일이 생김 | [커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#ops) |
| 행 수정은 삭제 표시로 처리 | upsert마다 equality delete가 쌓여, 파티션 없는 테이블에서는 읽을 때마다 전체에 적용됨 | [커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#deletes) |
| 파일 다시 쓰기는 비쌈 | 파티션 없는 copy-on-write mirror 테이블이 `MERGE` 때마다 바뀐 행이 든 파일을 모두 다시 씀 | [Buffer Writer 리뷰](/2026/10/15/buffer-writer-batch-review.html) |
| 통계로 건너뛰려면 정렬·파티션이 필요 | bucket 파티션과 증분 컴팩션으로 삭제와 다시 쓰기의 범위를 줄이는 안 | [커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html#plan) |

</div>

> **배운 점**
>
> 컬럼 저장은 "한 번 쓰고 여러 번 크게 읽는" 데이터에 맞춰져 있습니다. 변경이 잦은 CDC 데이터를 담으려면, 변경 이력은 append로 쌓고 최신 상태는 주기적으로 합치는 식으로 **변경을 모아서 처리하는 구조**를 만들어야 컬럼 저장의 장점이 살아납니다.
{: .callout .info}

* * *

## 7. 참고 {#refs}

- [Apache Parquet: File Format](https://parquet.apache.org/docs/file-format/)
- [Apache Iceberg Spec: Manifests (컬럼 통계)](https://iceberg.apache.org/spec/#manifests)
- [What is a Column Store Database?](https://database.guide/what-is-a-column-store-database/)
- Daniel Abadi 외, [The Design and Implementation of Modern Column-Oriented Database Systems](https://doi.org/10.1561/1900000024) (2013)
- [컬럼 기반 DB 설명 (네이버 블로그)](https://m.blog.naver.com/PostView.nhn?blogId=kmk1030&logNo=221238376662)
