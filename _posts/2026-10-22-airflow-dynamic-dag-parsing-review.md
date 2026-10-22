---
layout: post
title: Airflow 동적 DAG 파싱 과부하 리뷰
tags: [Airflow, AWS Glue, Data Pipeline]
author: caimask
---

MySQL에 등록된 로그 설정 한 row마다 수집(intake) DAG 하나를 만들도록 제가 설계·개발한 코드를 직접 다시 검토한 문서입니다. 설정이 늘어날수록 DAG을 만드는 과정 자체가 무거워져, 스케줄 주기 안에 처리를 끝내지 못할 수 있는 문제가 있었습니다. 원인이 어디서 몇 번 반복되는지 짚고, 개선안과 적용 순서를 정리했습니다. 개선안은 아직 적용하지 않았습니다. "DAG 하나짜리 파일 300개로 나누면 가벼워진다"는 당시 가정도 함께 검토했습니다.

작성 2026-10-22 · 역할 DAG 생성 코드 설계·개발, 이후 자기 검토 · 대상 설정 기반 동적 DAG 생성 코드(intake) · Airflow 2.x · AWS Glue 4.0 · 규모 약 300 DAG
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **원인은 연산이 아니라 파싱입니다.** Airflow는 DAG 파일을 주기적으로 다시 실행합니다. 그래서 파일 최상단과 `create_dag()` 본문에 있는 MySQL 조회, SSM 호출, pool 조회·생성이 **파싱할 때마다** 실행됩니다.
2. **비용이 세 겹으로 곱해집니다.** 설정 row 수(N)만큼 반복되고, 스케줄러가 파싱할 때마다 반복되고, 워커에서 태스크 하나가 시작될 때마다 또 반복됩니다. 태스크 하나를 돌리려고 DAG 300개를 다시 만듭니다.
3. **부하와 별개로 장애 격리가 없었습니다.** MySQL 조회가 실패하거나 파싱이 `dagbag_import_timeout`을 넘으면 파일 import가 실패하고, **intake DAG 300개가 함께 UI에서 빠집니다.** 자기 검토로 이 결함을 포함해 네 가지를 찾았고, 각각의 수정안을 정리했습니다(미적용).
4. **개선은 "파싱 경로에서 외부 호출 제거"가 먼저라고 판단했습니다.** pool 조회·생성과 SSM 호출을 빼고, 설정은 캐시에서 읽고, 워커에서는 필요한 DAG 하나만 만드는 계획입니다(미적용).
5. **파일 300개로 나누는 것은 그다음입니다.** 파일을 나누면 파싱이 병렬화되고 장애가 DAG 단위로 격리됩니다. 하지만 파일마다 드는 고정 비용이 크면 오히려 300배 나빠지므로, 순서를 바꾸면 안 됩니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [기존 구성](#setup)
2. [Airflow는 DAG 파일을 언제 실행하는가](#parsing)
3. [파싱 1회당 비용](#cost)
4. [부하와 별개로 발견한 문제](#bugs)
5. [개선안](#fix)
6. [파일 하나 vs 파일 300개](#files)
7. [다른 선택지](#alternatives)
8. [적용 순서](#plan)
9. [성과와 기대 효과](#outcome)
10. [참고](#refs)

</div>

## 1. 기존 구성 {#setup}

로그 수집 대상이 MySQL의 `source_registry` 테이블에 등록되면, DAG 파일 하나가 이 테이블을 조회해 row마다 수집 DAG을 만듭니다. 수집 대상을 추가할 때 코드를 고치지 않고 row만 넣으면 되도록 설계했습니다.

```
MySQL source_registry (row 1개 = 수집 대상 1개)
   │  DAG 파일 최상단: load_source_config()로 전체 조회
   ▼
for item in items:
    globals()[dag_id] = create_dag(dag_id, default_args, item)   ← row 수만큼 반복
```

DAG 하나는 태스크 9개로 이루어집니다.

```
task_pre (선행 DAG 실행·대기. hook을 쓰는 유형에서만 동작)
  └▶ task_pre_py (Branch: 입수할 파일 크기로 Glue 리소스 계산, 실시간 파티션 관리)
       ├▶ submit_glue_job (AWS Glue 4.0, pool로 동시 실행 20개 제한)
       │     ├▶ task_success (파일 크기, Trino ANALYZE, 통계 저장, 감사 로그) ─┐
       │     └▶ task_fail    (감사 로그) ──────────────────────────────────────┤
       └▶ task_pre_skip (입수할 데이터가 없으면 Glue 생략) ────────────────────┤
                                                                               ▼
           task_post (후행 DAG 실행·대기. 위와 같음) ▶ task_post_py (건수 이상 탐지) ▶ task_final
```

설정이 늘어나면서 DAG 반영이 느려지고, 스케줄 주기 안에 처리를 끝내지 못할 수 있는 상황이 생겼습니다. 아래 코드 조각은 검토에 필요한 부분만 남기고, 버킷과 계정, 내부 주소는 자리표시로 바꿨습니다.

## 2. Airflow는 DAG 파일을 언제 실행하는가 {#parsing}

DAG 파일은 정의 파일이 아니라 **실행되는 파이썬 코드**입니다. Airflow는 이 파일을 다음 시점마다 처음부터 끝까지 실행해 DAG 객체를 얻습니다.

<div class="table-wrap" markdown="1">

| 시점 | 누가 | 빈도 |
| --- | --- | --- |
| 주기적 파싱 | 스케줄러의 DAG processor | 파일마다 `min_file_process_interval`(기본 30초)마다 |
| 태스크 실행 | 워커 | **태스크 인스턴스 하나가 시작될 때마다** 그 DAG 파일 전체를 다시 실행 |
| 수동 명령 | `airflow dags list`, `airflow tasks test` 등 | 실행할 때마다 |

</div>

그리고 파싱 단위는 **파일**입니다.

- `parsing_processes`(기본 2)를 늘려도 파일 하나는 프로세스 하나가 처리합니다. DAG 300개가 든 파일 하나는 병렬화되지 않습니다.
- 파일 하나의 실행 시간이 `dagbag_import_timeout`(기본 30초)을 넘으면 import 오류가 되고, 그 파일의 DAG이 전부 UI에서 사라집니다.

> **놓치기 쉬운 것: 워커도 파싱합니다**
>
> 스케줄러 파싱만 생각하고 `min_file_process_interval`을 늘리면 부하가 줄어든 것처럼 보입니다. 하지만 워커는 태스크를 시작할 때마다 그 파일을 다시 실행합니다. 태스크 하나를 돌리려고 DAG 300개를 만들고 MySQL과 SSM을 다시 호출합니다.
{: .callout .warn}

## 3. 파싱 1회당 비용 {#cost}

`create_dag()`에서 런타임에 실행되는 것은 안쪽 함수(`do_pre`, `do_success` 등)의 본문뿐입니다. **`create_dag()`의 나머지 본문은 파싱할 때 실행됩니다.**

```python
# 모듈 최상단: 파싱마다 1회
AUDIT_PASSWORD = get_secret("/<env>/audit/password")   # AWS SSM API 호출
items = load_source_config()                           # MySQL 조회

def create_dag(dag_id, default_args, cfg):
    ...
    pool = ensure_pool(cfg["group"], slots=MAX_GLUE_RUNS)       # 메타 DB에서 조회, 없으면 생성. 파싱마다 N회
    dag = DAG(dag_id, schedule=get_schedule(), ...)             # 현지 시간 → UTC 계산만. 파싱마다 N회
    with dag:
        pre = TriggerAndWait(task_id="task_pre", ...)           # Operator 생성자도 파싱마다 N회
        ...
```

<div class="table-wrap" markdown="1">

| 위치 | 코드 | 파싱 1회당 | 심각도 |
| --- | --- | --- | --- |
| `create_dag` 내부 | `ensure_pool(...)` | **N회**. 메타 DB에서 pool을 조회하고 없으면 생성(조회 + 쓰기) | <span class="pill bad">매우 높음</span> |
| `create_dag` 내부 | `get_schedule()` | N회. 배치가 돌기를 기대하는 현지 시간대를 UTC 기준 스케줄로 바꾸는 계산만 함. 외부 호출 없음 | <span class="pill info">낮음</span> |
| 모듈 최상단 | `get_secret(...)` | 1회. AWS SSM API 호출 | <span class="pill warn">높음</span> |
| 모듈 최상단 | `load_source_config()` | 1회. 서브쿼리가 있는 MySQL 조회 | <span class="pill warn">높음</span> |
| 모듈 최상단 | 공통 모듈 import (`trino`, DB 헬퍼 등) | 1회 | <span class="pill info">모듈에 따라 다름</span> |

</div>

Airflow는 UTC 기준으로 동작하므로 `get_schedule()`은 현지 시간 스케줄을 UTC로 바꿔 줍니다. 계산만 하므로 비용은 작지만, 파싱마다 N회 호출되므로 모듈 레벨에서 한 번 계산해도 됩니다. `TriggerAndWait`(선행·후행 DAG 실행 오퍼레이터)와 `AwsGlueJobOperator`의 생성자도 파싱마다 N회 실행되므로, 생성자는 설정값만 저장하고 외부 호출은 실행 시점에만 하도록 유지해야 합니다.

이 비용이 얼마나 반복되는지 따져 보면 이렇습니다.

```
스케줄러:  30초마다 × (SSM 1 + MySQL 1 + pool 조회·생성 N + DAG 생성 N)
워커:      DAG 1회 실행에서 실제로 도는 태스크 7개 안팎 × 같은 비용
           → N=300이면 DAG 한 번 돌 때 DAG 객체 약 2,100개를 만들고 ensure_pool을 약 2,100번 호출
           → DAG 300개가 하루 한 번씩 돈다고 하면 워커 쪽에서만 하루 약 63만 번
```

설정이 늘어날수록 부하가 선형으로 커지는 직접적인 원인은 `create_dag()` 안의 외부 호출, 그중에서도 DAG마다 메타 DB를 조회·쓰기 하는 `ensure_pool`입니다. 개선 전후는 다음 명령으로 비교할 계획입니다.

```bash
airflow dags report                                              # 파일별 파싱 시간
python -m cProfile -s cumtime dags/ingest_dag.py 2>&1 | head -40
```

프로파일 결과에서는 `ensure_pool`, `get_secret`, `db.query`, 각 Operator의 `__init__`이 차지하는 시간을 봅니다.

## 4. 부하와 별개로 발견한 문제 {#bugs}

코드를 다시 보면서 부하와 별개로 찾은 결함입니다. 각 결함의 원인과 개선 방향을 정리했고, 아직 적용하지 않았습니다.

<div class="table-wrap" markdown="1">

| 문제 | 원인 | 영향 | 개선 방향 · 상태 |
| --- | --- | --- | --- |
| <span class="pill bad">전체 소실</span> DB 장애 시 DAG 300개가 빠짐 | `load_source_config()`가 예외를 삼키고 `None`을 반환하게 만든 것 → `for item in items`에서 `TypeError` | 파일 import 실패. MySQL이 잠깐 흔들려도 모든 intake DAG이 UI에서 빠짐 | 실패 시 마지막 캐시로 대체(5장 ④). 미적용 |
| <span class="pill warn">개선</span> 실패 콜백이 다시 실패할 수 있음 | 실패 콜백에서 context에 `exception`이 없으면 예외 메시지 변수가 정의되지 않음 | `UnboundLocalError`로 실패 알림이 안 감 | 변수를 기본값으로 먼저 초기화. 미적용 |
| <span class="pill warn">개선</span> Airflow 1 스타일 import | `python_operator`, `dummy_operator`, `provide_context` | 버전 업그레이드 시 깨짐. `get_parsing_context()` 사용 가능 여부도 버전에 달림 | Airflow 2 경로로 교체. 미적용 |
| <span class="pill info">참고</span> 오류 메시지 불일치 | MySQL을 조회하면서 "pymssql 연결 오류"를 로그로 남김 | 장애 분석 시 혼란 | 메시지 수정. 미적용 |

</div>

첫 번째가 가장 위험합니다. 부하 문제는 처리가 늦어지는 것으로 끝나지만, 이 문제는 MySQL 장애 동안 수집 전체를 멈춥니다. 그래서 적용 순서(8장)에서 가장 앞에 두었습니다.

이 코드는 hook을 쓰지 않는 유형의 DAG을 만드는 코드라 pre/post hook 컬럼을 읽지 않습니다. hook을 쓰는 유형은 다른 생성 코드가 pre_hook·post_hook 정보를 읽어 태스크에 넘깁니다.

## 5. 개선안 {#fix}

제가 정리한 개선 계획입니다. 상태는 모두 미적용이고, 효과가 큰 순서로 적었습니다.

### ① `create_dag()`에서 외부 호출 제거

pool은 DAG을 만들 때마다 메타 DB에서 조회하고 생성할 필요가 없습니다. 이름만 쓰고, 생성은 배포 단계나 별도 DAG에서 한 번만 하기로 했습니다. `get_schedule()`은 계산만 하지만 DAG마다 같은 값을 다시 계산하므로 모듈 레벨로 옮깁니다.

```python
# 모듈 레벨에서 1회만
POOL_NAME = "intake_group_pool"   # 이름만 사용. 생성은 배포 단계에서 한 번
SCHEDULE = get_schedule()         # 현지 시간 → UTC 계산. N회 → 1회

def create_dag(dag_id, default_args, cfg):
    ...
    # pool = ensure_pool(cfg["group"], ...)   ← 삭제
    dag = DAG(dag_id, schedule=SCHEDULE, ...)
    submit_glue_job = AwsGlueJobOperator(..., pool=POOL_NAME, ...)
```

### ② SSM 호출은 실제로 쓰는 시점까지 미루기

비밀번호는 감사 로그를 보낼 때만 필요합니다. 런타임 함수 안에서 처음 쓸 때 한 번만 가져옵니다.

```python
from functools import lru_cache

@lru_cache(maxsize=1)
def audit_password():
    return get_secret("/<env>/audit/password")

# 사용처: AUDIT_PASSWORD → audit_password()
send_audit_log(AUDIT_URL, AUDIT_USER, audit_password(), context, "do_final")
```

### ③ 워커에서는 필요한 DAG 하나만 만들기

Airflow 2.4부터 `get_parsing_context()`로 "지금 어떤 DAG의 태스크를 실행하려고 파싱하는 중인가"를 알 수 있습니다. 스케줄러가 파싱할 때는 `None`입니다.

```python
def _current_dag_id():
    try:
        from airflow.utils.dag_parsing_context import get_parsing_context  # 2.4+
        return get_parsing_context().dag_id
    except ImportError:
        # 구버전: 워커는 "airflow tasks run <dag_id> <task_id> ..." 형태로 실행됨
        if len(sys.argv) > 3 and sys.argv[1:3] == ["tasks", "run"]:
            return sys.argv[3]
    return None

CURRENT_DAG_ID = _current_dag_id()

for item in items:
    dag_id = "ingest_" + item["source_id"]
    if CURRENT_DAG_ID and CURRENT_DAG_ID != dag_id:
        continue                     # 워커에서는 DAG 300개 대신 1개만 생성
    globals()[dag_id] = create_dag(dag_id, default_args_for(item), item)
```

### ④ 설정은 캐시에서 읽고, 장애 시 마지막 캐시로 버티기

```python
CACHE = Path(__file__).parent / "configs" / "source_config.json"

def load_source_config():
    try:
        rows = db.query(SQL)
        CACHE.parent.mkdir(exist_ok=True)
        CACHE.write_text(json.dumps(rows, default=str))   # 성공하면 캐시 갱신
        return rows
    except Exception:
        logger.exception("설정 조회 실패, 캐시 사용")
        return json.loads(CACHE.read_text()) if CACHE.exists() else []
```

- 4장의 "DB 장애 시 전체 소실" 문제도 이 수정으로 함께 해결할 계획입니다.
- 캐시에서 읽으면 `created_at`이 문자열이므로, `start_date` 계산 전에 `pendulum.parse()`로 변환합니다.
- 더 나아가면 파싱 경로에서 MySQL을 아예 빼고, 별도의 동기화 DAG가 10분마다 설정을 JSON으로 내보내게 합니다. 이때 스케줄러와 워커가 분리된 환경(Celery, Kubernetes)이라면 JSON을 **모든 노드가 같이 보는 DAG 폴더**(공유 볼륨, git-sync)에 써야 합니다. 파일은 임시 파일에 쓰고 `os.replace`로 바꿔, 반쯤 쓰인 파일을 읽는 일이 없게 합니다.

### ⑤ 설정 조회 쿼리 개선

`(a, b, c, d) IN (서브쿼리)` 형태를 JOIN으로 바꾸고, 최신 row를 찾는 인덱스를 둡니다.

```sql
-- 예시 스키마: 키 (domain, source, dataset), 같은 키에 버전별 row가 쌓임
SELECT c.owner, r.domain, r.source, r.dataset, r.created_at, r.version
FROM source_registry r
JOIN (
    SELECT domain, source, dataset, MAX(created_at) AS max_created_at
    FROM source_registry
    GROUP BY domain, source, dataset
) m ON r.domain = m.domain AND r.source = m.source
   AND r.dataset = m.dataset AND r.created_at = m.max_created_at
JOIN source_category c
  ON r.domain = c.domain AND r.source = c.source AND r.dataset = c.dataset;

CREATE INDEX idx_source_registry_latest ON source_registry (domain, source, dataset, created_at);
```

### 당장 장애를 막는 임시 조치

구조를 고치기 전에 DAG이 사라지는 것부터 막아야 한다면 설정을 조정합니다. 설정 변경이 늦게 반영되는 것을 감수하는 조치로, 이것도 아직 적용하지 않았습니다.

```ini
[core]
dagbag_import_timeout = 120        # import 실패로 DAG이 사라지는 것 방지
[scheduler]                        # Airflow 3에서는 [dag_processor]
min_file_process_interval = 300
dag_file_processor_timeout = 180
parsing_processes = 4
```

## 6. 파일 하나 vs 파일 300개 {#files}

**당시 가정** 파일 하나에서 DAG 300개를 만드는 대신 DAG 코드나 설정을 실제 파일 300개로 나눠 두면 파싱 부하가 줄어든다는 가정이 있었습니다.
{: .q}

**확인한 사실** 부하에는 차이가 있습니다. 하지만 "파일 300개가 무조건 가볍다"는 아닙니다. 어떤 비용은 줄고 어떤 비용은 늘어납니다.

### 비용 모델

- `T_fixed`: 파일 하나를 파싱할 때 한 번 드는 비용 (import, SSM 호출, MySQL 조회, 프로세스 생성)
- `t_dag`: DAG 하나를 만드는 비용 (Operator 생성, pool 조회·생성)
- `P`: `parsing_processes`

<div class="table-wrap" markdown="1">

|   | 파일 1개 → DAG 300개 (현재) | 파일 300개 → 각각 DAG 1개 |
| --- | --- | --- |
| 파일 1개 파싱 시간 | `T_fixed + 300·t_dag` | `T_fixed + t_dag` |
| 전체 CPU 사용량 | `T_fixed + 300·t_dag` | `300·(T_fixed + t_dag)` |
| 한 바퀴 도는 데 걸리는 시간 | `T_fixed + 300·t_dag` (병렬화 불가) | 약 `300·(T_fixed + t_dag) / P` |
| 워커가 태스크 1개 실행 시 | DAG 300개 생성 | DAG 1개 생성 |
| `dagbag_import_timeout` 위험 | N이 늘수록 초과 | 거의 없음 |
| 장애 범위 | 300개 전체 | DAG 하나 |

</div>

결론은 `T_fixed`에 달려 있습니다.

- `T_fixed`가 작으면(외부 호출이 없는 가벼운 파일) 파일 300개가 압도적으로 유리합니다.
- `T_fixed`가 크면(파일마다 SSM이나 MySQL을 호출) 파일 300개가 **오히려 300배 나빠집니다.**

### 함정 세 가지

1. **설정 파일 300개와 DAG 파일 300개는 다릅니다.** 설정 파일 300개를 파이썬 파일 하나가 읽어 DAG을 만들면 파싱 단위는 여전히 파일 하나입니다. 분산 효과를 보려면 `.py` 파일이 300개여야 합니다.
2. **공통 모듈의 최상단 코드가 300번 실행됩니다.** 얇은 DAG 파일 300개가 공통 factory 모듈을 import하면, factory의 최상단 코드가 파일마다 실행됩니다. 외부 호출 제거가 먼저 끝나 있어야 합니다.
3. **DAG 탐색 조건이 있습니다.** `dag_discovery_safe_mode`(기본 켜짐)에서는 파일 안에 `airflow`와 `dag`라는 문자열이 없으면 파싱하지 않습니다. 생성기가 만드는 파일에는 두 단어가 들어가야 합니다.

### 계획한 형태: 얇은 DAG 파일 + 공통 factory

```python
# dags/intake/dom_a_src_b_ds_c.py  ← 생성기가 만드는 파일 (airflow DAG)
from intakes.factory import create_intake_dag

dag = create_intake_dag({
    "domain": "dom_a", "source": "src_b", "dataset": "ds_c",
    "owner": "<owner>",
    "created_at": "2026-03-01T00:00:00", "version": "v2",
})
```

```python
# plugins/intakes/factory.py  ← 지금의 create_dag. 최상단에 외부 호출 금지
POOL_NAME = "intake_group_pool"          # 이름만 사용, 생성은 따로

def create_intake_dag(cfg: dict) -> DAG:
    ...  # SSM, Trino, MySQL은 do_pre/do_success 같은 런타임 함수 안에서만 호출
```

생성기는 동기화 DAG이나 CI에서 MySQL을 조회해, **바뀐 row의 파일만** 다시 쓰고 지워진 row의 파일은 지우도록 만들 계획입니다.

<div class="table-wrap" markdown="1">

|   | 파일 1개 + MySQL 조회 | 파일 300개 생성 |
| --- | --- | --- |
| 설정 반영 속도 | 다음 파싱 때 바로 | 생성기 실행 주기만큼 늦음 |
| 배포와 관리 | 단순 | 생성기, 파일 동기화, 삭제 처리 필요 |
| N이 늘어날 때 | 한계가 분명함 | 선형으로 분산 |
| 장애 격리 | 없음 | DAG 단위 |
| 추적성 | DB 상태에 의존 | 파일이 곧 설정. git으로 이력 관리 |

</div>

> **결론**
>
> 지금 300개이고 앞으로 늘어날 예정이므로 파일 생성 방식이 맞다고 판단했습니다. 다만 순서는 **factory에서 최상단 외부 호출 제거 → 파일 분리**여야 합니다. 순서를 바꾸면 고정 비용이 300배가 됩니다.
{: .callout .info}

## 7. 다른 선택지 {#alternatives}

**정말 DAG이 300개 필요한가**부터 다시 볼 수 있습니다.

<div class="table-wrap" markdown="1">

| 상황 | 맞는 구조 |
| --- | --- |
| row마다 스케줄, 소유자, 실패 알림, 재실행 단위가 다름 | DAG N개 (이 문서의 개선안) |
| 스케줄이 같고 파라미터만 다름 | DAG 하나 + Dynamic Task Mapping (`expand`) |
| 당장 장애 대응 | 5장 임시 조치 후 ①~④ 적용 |

</div>

```python
@dag(schedule="@daily", catchup=False, max_active_tasks=16)
def all_intakes():
    @task
    def get_configs():
        return MySqlHook("mysql_conn").get_records("SELECT ... FROM source_registry ...")  # 런타임에만 조회

    @task(pool="intake_group_pool")
    def run_intake(cfg): ...

    run_intake.expand(cfg=get_configs())
```

Dynamic Task Mapping은 파싱 비용이 설정 수와 무관해집니다. 대신 수집 대상별로 실행 이력을 보거나 하나만 다시 돌리기가 불편해지고, 매핑 개수 상한(`max_map_length`, 기본 1024)이 있습니다. 이 intake는 대상마다 선행·후행 DAG을 따로 둘 수 있고, 수동 실행으로 특정 날짜(`--DT`)와 Glue 리소스를 덮어써 대상 하나만 재처리하는 기능이 있습니다. 그래서 DAG을 대상별로 나눈 설계는 유지하고, DAG을 만드는 방식을 고치는 것이 맞다고 판단했습니다.

## 8. 적용 순서 {#plan}

아직 적용하지 않은 개선안을 어떤 순서로 적용할지 정한 계획입니다.

<div class="steps" markdown="1">

<div class="step" markdown="1">
<span class="when">즉시</span>
### 소실 방지

`load_source_config()` 실패 시 캐시로 대체(④), 필요하면 `dagbag_import_timeout` 상향. 실패 콜백의 `UnboundLocalError` 수정.
</div>

<div class="step" markdown="1">
<span class="when">1단계</span>
### 파싱 경로 정리

`create_dag()`에서 pool 조회·생성 제거, `get_schedule()`을 모듈 레벨 1회 계산으로 이동(①), SSM 지연 로딩(②), 적용 전후 측정.
</div>

<div class="step" markdown="1">
<span class="when">2단계</span>
### 워커 파싱 축소

`get_parsing_context()`로 워커에서는 DAG 하나만 생성(③). 쿼리와 인덱스 정리(⑤).
</div>

<div class="step" markdown="1">
<span class="when">3단계</span>
### 파일 분리

factory 모듈 + 생성기로 DAG 파일 N개 생성(6장).
</div>

</div>

> **배운 점**
>
> 제가 만든 Iceberg 커넥터의 커밋 경합과 같은 교훈입니다. 느려지는 원인은 한 번의 무거운 작업이 아니라 **가벼워 보이는 작업이 몇 겹으로 곱해지는 것**이었습니다. 커넥터에서는 "writer 수 × 몇 초마다 커밋"이, 여기서는 "설정 수 × 파싱 주기 × 태스크 수"가 그랬습니다. 코드를 볼 때는 무엇을 하느냐보다 **언제, 몇 번 실행되느냐**를 먼저 봅니다. ([Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html))
{: .callout .info}

## 9. 성과와 기대 효과 {#outcome}

### 기존 구현의 성과

<div class="table-wrap" markdown="1">

| 구분 | 성과 |
| --- | --- |
| 수집 자동화 | 수집 대상 약 300개를 MySQL에 row로 등록하는 것만으로 DAG이 만들어짐. 대상을 추가할 때 코드 변경과 배포가 필요 없음 |
| 리소스 자동 산정 | 입수된 로그 파일 크기로 Glue worker 수와 Spark 파티션 수를 실행마다 계산 |
| 비용 절감 | 입수할 데이터가 없으면 Glue 실행을 건너뜀 |
| 동시성 제어 | pool로 Glue 동시 실행을 20개로 제한해, 대상이 늘어도 Glue 한도를 넘지 않음 |
| 테이블 관리 | 다음 날짜의 실시간 조회용 파티션 생성, 보관 기간(7일)이 지난 실시간 로그 삭제, 데이터가 없는 날의 빈 파티션 삭제를 자동으로 처리 |
| 품질 관리 | 적재 후 Trino `ANALYZE`로 통계 갱신, 파일·데이터 크기를 `ingest_stats`에 기록, 적재 건수가 0이면 이상 이벤트 기록 |
| 감사·추적 | 성공, 실패, 건너뜀, 종료를 OpenSearch에 태스크 단위로 기록 |
| 재처리 | 수동 실행 시 날짜(`--DT`)와 Glue 리소스를 덮어써 대상 하나만 다시 처리. 이때는 실시간 파티션 정책을 건드리지 않음 |

</div>

### 개선을 적용하면 얻는 것

개선안은 아직 적용하지 않았으므로, 아래 수치는 N=300과 코드 구조로 계산한 예상치입니다. 적용하면 측정값으로 바꿀 계획입니다.

<div class="table-wrap" markdown="1">

| 개선 작업 | 무엇이 바뀌나 | 지금 | 적용 후 (예상) | 근거 |
| --- | --- | --- | --- | --- |
| **워커에서는 실행할 DAG 하나만 만들기** (③) | 태스크 1개 시작 시 만드는 DAG | 300개 | 1개 | 5장 |
| | DAG 1회 실행에서 만드는 DAG 객체 | 약 2,100개 | 약 7개 | 3장 |
| **`create_dag()`에서 pool 조회·생성 호출 없애기** (①) | 파싱 1회당 `ensure_pool` 호출(메타 DB 조회, 없으면 생성) | 300회 | 0회 | 5장 |
| **SSM 호출을 실제로 쓰는 시점까지 미루기** (②) | 파싱마다 SSM API 호출 | 1회 (스케줄러 30초마다 + 태스크마다) | 0회. 감사 로그를 보낼 때만 호출 | 5장 |
| **설정을 캐시에서 읽고, 장애 시 마지막 캐시로 버티기** (④) | MySQL 장애 시 사라지는 DAG | 300개 전체 | 0개 | 4장, 5장 |
| **대상마다 얇은 DAG 파일로 나누고 공통 factory 사용** | 파싱 병렬도 | 1 프로세스 | `parsing_processes`만큼 | 6장 |
| | 대상 증가에 따른 `dagbag_import_timeout` 위험 | 대상 수 N에 비례해 커짐 | N과 무관 | 6장 |
| | 장애 범위 | 300개 전체 | DAG 하나 | 6장 |
| | 설정 변경 이력 | DB 상태에만 있음 | git 이력으로 남음 | 6장 |

</div>

> **대가로 내주는 것**
>
> 설정 반영 속도: 다음 파싱 때 → **생성기 실행 주기만큼 늦음**. 파일을 나누면 DB의 설정이 파일로 만들어져야 반영됩니다. (6장)
{: .callout .warn}

> **측정 방법**
>
> 적용 전후로 `airflow dags report`의 파일별 파싱 시간, 스케줄러의 `dag_processing.last_duration` 지표, 태스크가 queued에서 running으로 바뀌기까지의 시간을 비교할 계획입니다. 워커 파싱이 줄면 마지막 값이 가장 크게 달라집니다.
{: .callout .info}

* * *

## 10. 참고 {#refs}

- [Airflow Best Practices: Top level Python Code](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html#top-level-python-code)
- [Airflow: Dynamic DAG Generation](https://airflow.apache.org/docs/apache-airflow/stable/howto/dynamic-dag-generation.html) (`get_parsing_context` 포함)
- [Airflow Configuration Reference](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html) (`min_file_process_interval`, `dagbag_import_timeout`, `dag_discovery_safe_mode`)
- [Airflow: Dynamic Task Mapping](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/dynamic-task-mapping.html)
- 연관 문서: [Iceberg 커밋 경합 리뷰](/2026/10/08/iceberg-commit-conflict-review.html) · [Logstash로 MySQL 데이터를 Kafka로 보내기](/2020/08/06/post-logstash.html)
