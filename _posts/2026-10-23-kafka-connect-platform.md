---
layout: post
title: 쿠버네티스 위 Kafka Connect 클러스터 구성과 오픈소스 패치
tags: [Kafka Connect, Kubernetes, Helm, Debezium, Docker]
author: caimask
---

이벤트 기반 데이터 파이프라인의 커넥터들이 도는 Kafka Connect 클러스터를 공용 EKS에 Helm chart로 구성하고, 공식 이미지(`confluentinc/cp-kafka-connect`)를 다시 빌드해 운영한 기록입니다. 클러스터와 배포 구조를 먼저 정리하고, 운영 중 설정으로 풀리지 않아 오픈소스 코드를 직접 고친 세 가지를 문제 → 원인 → 수정 → 대가 순서로 적었습니다.

작성 2026-10-23 · 역할: Helm chart 구성, 이미지 재빌드 구조 설계, 커넥터 배포 CI, 오픈소스 패치(Avro Converter, Debezium MongoDB 커넥터)(2023-08 ~ 2026-09) · Confluent Platform 7.2.6 · Debezium 2.0.1
{: .meta}

이 글을 포함하는 사례: [이벤트 기반 데이터 파이프라인 구축](/portfolio/event-driven-pipeline)
{: .meta}

<div class="summary" markdown="1">

## 요약 {#tldr}

1. **이미지와 플러그인을 나눴습니다.** 이미지에는 공통 런타임 수정만 넣고, 커넥터 플러그인은 pod가 뜰 때 init container가 설치합니다. 직접 만든 커넥터를 새로 배포할 때 이미지를 다시 빌드하지 않습니다.
2. **커넥터 배포는 설정 파일 + CI로 했습니다.** 커넥터마다 환경별 설정 JSON을 두고, CI가 Connect REST API로 배포·일시정지·재시작·삭제를 수행합니다.
3. **설정으로 풀리지 않는 문제 세 가지를 오픈소스 코드 수정으로 풀었습니다.** Schema Registry 일시 실패로 task가 내려가는 문제(Avro Converter), `@`가 들어간 필드를 필터에 쓸 수 없는 문제와 제외한 필드가 변경 목록에서도 사라지는 문제(Debezium MongoDB)입니다.
4. **패치는 같은 버전 tag에서 최소로 고치고, 고친 위치와 빌드 절차를 이미지 저장소에 남겼습니다.** 대가는 공식 jar와 버전 번호로 구분되지 않는다는 점과, 버전을 올릴 때 패치를 다시 옮겨야 한다는 점입니다.

</div>

<div class="toc" markdown="1">

### 목차

1. [클러스터 구성](#cluster)
2. [커넥터 배포](#deploy)
3. [패치 1: JSON 로그](#json-log)
4. [패치 2: Schema Registry 일시 실패를 재시도로](#avro-retry)
5. [패치 3: Debezium MongoDB 필드 필터](#debezium-filter)
6. [패치를 관리한 방식과 대가](#patch-cost)
7. [남은 과제](#next)

</div>

## 1. 클러스터 구성 {#cluster}

Kafka Connect를 distributed 모드로 띄우는 Deployment와 Service, PodDisruptionBudget, 설정용 ConfigMap을 Helm chart 하나로 묶었습니다. Confluent의 공개 chart를 바탕으로 init container, log4j ConfigMap, 외부 Secret, PDB를 더했고, 환경마다 values 파일만 달리 둡니다. 용도에 따라 클러스터를 둘(실험적인 자체 커넥터용, 안정화된 공개 커넥터용)로 나눠 운영했습니다.

```
[이미지]  cp-kafka-connect 7.2.6
          + JSON 로그 layout jar (log4j)
          + 재빌드한 Avro Converter jar        ← 공식 jar 자리에 덮어씀
          + 재빌드한 Debezium MongoDB zip      ← Confluent Hub zip 형식 그대로
          + jq                                ← 플러그인 설치 스크립트용

[pod 시작]
  init container (같은 이미지)
    ├─ Confluent Hub 플러그인 설치 (S3 sink, MongoDB, Debezium, Protobuf converter)
    └─ 사내 Maven 저장소에서 직접 만든 커넥터 jar 내려받기
         └─ 공유 emptyDir 볼륨에 설치
  worker container
    └─ plugin.path = 기본 경로 + 위 두 볼륨
```

<div class="table-wrap" markdown="1">

| 항목 | 구성 |
| --- | --- |
| worker 설정 | values의 설정 dict를 `CONNECT_*` 환경 변수로 바꿔 주입. converter(key는 JSON, value는 Avro), 내부 토픽 복제 수(3), client override 정책(`All`) 등 |
| 플러그인 배포 | init container의 설치 스크립트가 Maven 저장소 검색 API로 jar를 내려받음. SNAPSHOT이면 검색 결과를 페이지 단위로 넘기며 가장 최근 빌드를 고름 |
| 비밀 값 | Secret을 볼륨으로 마운트하고 `DirectoryConfigProvider`로 커넥터 설정에서 참조. 설정 JSON에 비밀번호를 쓰지 않음 |
| 로그 | log4j 설정을 ConfigMap으로 마운트하고 `KAFKA_LOG4J_OPTS`로 지정. 이미지를 다시 빌드하지 않고 로그 레벨과 형식을 바꿈 |
| 메모리 | 힙을 고정값 대신 컨테이너 메모리 비율(`InitialRAMPercentage=50`, `MaxRAMPercentage=80`)로 지정 |
| 가용성 | 환경별 replica 2~3, PodDisruptionBudget으로 노드 교체 중 worker가 한꺼번에 내려가지 않게 함 |
| 빌드 | Jenkins로 이미지 빌드, 이후 멀티 아키텍처(amd64/arm64) 이미지로 전환 |

</div>

이미지와 플러그인을 나눈 이유는 바뀌는 속도가 달라서입니다. 직접 만든 커넥터는 자주 고쳐 배포하지만, 런타임과 그 라이브러리는 거의 바뀌지 않습니다. 커넥터 jar를 이미지에 굽게 하면 커넥터 하나를 고칠 때마다 공용 이미지를 다시 빌드하고 모든 환경의 이미지 태그를 바꿔야 합니다. 대신 커넥터 jar를 바꾼 뒤에는 worker를 재시작해야 새 jar가 설치됩니다.

## 2. 커넥터 배포 {#deploy}

커넥터마다 디렉터리 하나에 환경별 설정 JSON(dev/qa/prod)과 배포 파이프라인을 둡니다. CI는 실행할 동작을 파라미터로 받아 Connect REST API를 호출합니다.

<div class="table-wrap" markdown="1">

| 동작 | REST API | 쓰는 때 |
| --- | --- | --- |
| deploy | `PUT /connectors/{name}/config` | 설정 반영. 없으면 생성, 있으면 갱신 |
| pause / resume | `PUT /connectors/{name}/pause`, `/resume` | 하류 점검 중 소비를 멈출 때 |
| restartAll / restartFailed | `POST /connectors/{name}/restart` | 전체 task 또는 실패한 task만 재시작 |
| remove | `DELETE /connectors/{name}` | 커넥터 제거 |

</div>

`PUT .../config`는 멱등이라 같은 설정을 여러 번 배포해도 결과가 같습니다. CDC source 커넥터는 deploy 단계에서 출력 토픽의 스키마를 Schema Registry에 먼저 등록하도록 했습니다. 등록 대상 토픽은 설정 파일의 토픽 라우팅 정규식에서 뽑아내므로, 수집 대상을 추가할 때 설정 한 곳만 고치면 됩니다.

## 3. 패치 1: 로그를 JSON으로 출력 {#json-log}

공용 EKS의 로그는 로그 플랫폼으로 수집되는데, 비용과 필드 검색을 위해 JSON 형식으로 출력하라는 가이드가 있었습니다. Kafka Connect는 log4j 1.x를 쓰므로 `JSONEventLayoutV1`(logstash jsonevent-layout)과 그 의존성 jar 두 개(commons-lang, json-smart)를 worker classpath에 넣고, layout 지정은 ConfigMap으로 했습니다.

```
log4j.appender.stdout.layout=net.logstash.log4j.JSONEventLayoutV1
```

코드 수정은 없고 jar만 더했습니다. jar는 이미지에, 설정은 ConfigMap에 두어서 layout이나 로그 레벨을 바꿀 때 이미지를 다시 빌드하지 않습니다.

## 4. 패치 2: Schema Registry 일시 실패를 재시도로 {#avro-retry}

**발견** Schema Registry에 접근하지 못해 커넥터 task가 `FAILED`로 내려가는 일이 반복됐습니다. 서버가 죽은 것이 아니라, 공용 Schema Registry의 동시 접근 한도 때문에 생기는 일시적인 실패였습니다. 다시 시도하면 성공하는 실패인데 task가 내려가서 사람이 재시작해야 했습니다.

**원인** Kafka Connect 런타임은 converter·transform 단계의 예외를 `RetryWithToleranceOperator`로 감싸 실행합니다. 이때 예외가 `RetriableException`이면 `errors.retry.timeout` 동안 백오프하며 다시 시도하고, 그 밖의 예외는 바로 tolerance 처리(기본값이면 task 실패)로 넘어갑니다. 그런데 Avro Converter는 직렬화 실패(`SerializationException`)를 모두 `DataException`으로 감싸 던지고 있어서, 일시적인 네트워크 실패도 재시도 없이 바로 task 실패로 이어졌습니다.

**수정** 사용 중인 버전과 같은 tag(`v7.2.6`)의 Schema Registry 저장소를 받아, `AvroConverter`의 `fromConnectData()`·`toConnectData()`에서 던지는 예외를 `RetriableException`으로 바꿨습니다. 같은 버전 번호로 빌드한 jar를 공식 jar 자리에 덮어쓰고, 커넥터에는 `errors.retry.timeout`을 10초로 지정했습니다.

```java
// 변경 전: 모든 직렬화 실패가 즉시 tolerance 처리로
} catch (SerializationException e) {
    throw new DataException("Failed to serialize Avro data ...", e);
}
// 변경 후: 런타임의 재시도 경로로
} catch (SerializationException e) {
    throw new RetriableException("Failed to serialize Avro data ...", e);
}
```

**재시도가 끝난 뒤** 10초 안에 회복하지 못하면 그다음은 커넥터의 `errors.tolerance`가 정합니다. 커넥터마다 다르게 뒀습니다.

<div class="table-wrap" markdown="1">

| 커넥터 | `errors.tolerance` | 재시도 후에도 실패하면 |
| --- | --- | --- |
| CDC source, 분배, 적재 커넥터 | 기본값(`none`) | task 실패. 레코드를 건너뛰지 않음 |
| 외부 API 호출 커넥터(트리거) | `all` + DLQ 토픽 | 레코드를 DLQ로 보내고 다음 레코드 처리 |

</div>

분배·적재 커넥터는 레코드 하나를 건너뛰면 하류 데이터가 틀어지므로 멈추는 쪽을, 트리거는 레코드 하나 때문에 전체 호출이 멈추지 않도록 DLQ로 격리하는 쪽을 골랐습니다.

> **판단과 대가**
>
> 런타임의 재시도 정책을 바꾸거나 커넥터마다 재시작 스크립트를 두는 대신, 예외를 분류하는 한 곳만 고쳤습니다. 런타임이 이미 가진 재시도·타임아웃·DLQ 설정을 그대로 쓸 수 있기 때문입니다. 대가로, 스키마가 실제로 맞지 않는 영구 실패도 일시 실패와 구분하지 않고 10초 동안 재시도한 뒤 실패합니다. 네트워크 계열 예외만 골라 `RetriableException`으로 바꾸는 편이 더 정확하지만, 당시에는 영구 실패가 드물고 10초 지연은 감수할 만하다고 판단했습니다.
{: .callout .info}

## 5. 패치 3: Debezium MongoDB 필드 필터 {#debezium-filter}

**배경** MongoDB CDC 이벤트에는 문서 전체가 실려서, 큰 본문 필드가 있는 컬렉션은 메시지가 매우 커졌습니다. 하류 처리에는 그 필드의 값이 아니라 **어떤 필드가 바뀌었는지**만 필요했습니다. Debezium의 `field.exclude.list`로 이 필드를 빼려고 했는데 두 가지가 맞지 않았습니다.

<div class="table-wrap" markdown="1">

| 문제 | 원인 | 수정 |
| --- | --- | --- |
| `@`로 시작하는 필드를 설정하면 커넥터가 설정 검증에서 실패 | MongoDB는 필드명에 `@`를 허용하지만, 설정값을 검사하는 정규식이 `\w`와 `-`만 허용 | 필드명 자리의 문자 집합에 `@`를 추가 |
| 제외한 필드는 바뀌어도 `updatedFields`에서 함께 빠짐 | 필터가 문서 본문과 변경 필드 목록에 같은 제거 규칙을 적용 | 새 옵션을 켜면 제외 필드도 변경 목록에 남기고, 값은 빈 문자열로 바꿈 |

</div>

`v2.0.1.Final` tag를 받아 설정 검증 정규식을 고치고, 새 설정 `custom.empty.updated.fields`를 추가했습니다. 옵션이 꺼져 있으면 기존 동작 그대로입니다.

```
변경 전: ^[*|\w|\-|\s*]+(?:\.[\w|\-]+\.[\w|\-]+)+(\.[\w|\-]+)*\s*$
변경 후: ^[*|\w|\-|\s*]+(?:\.[\w|\-]+\.[\@|\w|\-]+)+(\.[\@|\w|\-]+)*\s*$
```

```
updatedFields 처리 (custom.empty.updated.fields = true 일 때)
  1. 변경된 필드의 값을 모두 빈 문자열로 바꾼다       ← "무엇이 바뀌었나"만 남김
  2. 필드 rename 규칙은 적용하고, remove 규칙은 적용하지 않는다
     → 제외한 필드도 변경 목록에는 이름이 남는다
```

빌드한 jar는 Confluent Hub에서 받은 같은 버전의 zip에 넣어 다시 압축했습니다. 공개 패키지와 형식이 같으므로 설치 방법을 바꿀 필요가 없습니다.

결과적으로 문서 본문에서는 큰 필드가 빠지고, 그 필드가 바뀌었다는 사실은 `updatedFields`에 이름과 빈 값으로 남습니다. 하류는 기존 방식대로 변경 여부를 판단하면서 메시지 크기는 줄었습니다. 옵션으로 만든 것은 같은 플러그인을 쓰는 다른 커넥터의 동작을 바꾸지 않기 위해서입니다.

## 6. 패치를 관리한 방식과 대가 {#patch-cost}

<div class="table-wrap" markdown="1">

| 선택지 | 고른 것 | 이유 |
| --- | --- | --- |
| 오픈소스 수정 vs 우회(재시작 스크립트, 하류에서 필드 제거) | 오픈소스 수정 | 우회는 실패 뒤에 수습하거나 이미 커진 메시지를 다루는 방식이라, 원인 지점에서 고치는 편이 단순 |
| 포크해 새 버전으로 관리 vs 같은 버전으로 덮어쓰기 | 같은 tag에서 최소 수정, 같은 버전 번호로 덮어쓰기 | 공식 이미지의 다른 jar와 호환을 유지하고, 차이를 수정 몇 줄로 한정 |
| 동작 변경을 기본값으로 vs 옵션으로 | 옵션 | 같은 플러그인을 쓰는 다른 커넥터의 동작을 바꾸지 않음 |

</div>

같은 버전 번호로 덮어쓰면 이미지 안의 jar가 공식 jar인지 패치한 jar인지 버전만으로는 알 수 없습니다. 사실상 포크인데 포크라는 표시가 없는 상태입니다. 그래서 이미지 저장소에 다음을 함께 남겼습니다.

- 각 jar의 출처(Maven Central URL)와 넣은 이유를 Dockerfile 주석으로
- 패치마다 원본 저장소, checkout할 tag, 수정한 파일과 메서드, 수정 전후 코드, 빌드 명령과 산출물 경로를 README로
- Debezium zip을 다시 묶는 절차

지금 다시 한다면 버전 번호에 접미사(예: `7.2.6-patch1`)를 붙여 이미지 안에서도 구분되게 하겠습니다. 덮어쓸 경로는 그대로 두되 파일 이름만 바꾸면 되므로 호환성을 잃지 않습니다.

## 7. 남은 과제 {#next}

- **버전 업그레이드 시 패치 이식**: 런타임(7.2)과 Debezium(2.0)을 올리려면 두 패치를 새 버전에 다시 적용해야 합니다. 상위 버전에서 같은 문제가 고쳐졌는지 먼저 보고, 남는 패치만 옮기는 것이 순서입니다.
- **init container 설치의 외부 의존**: pod가 뜰 때마다 Confluent Hub와 Maven 저장소에 접근하므로, 저장소 장애가 worker 재시작 실패로 이어질 수 있습니다. 릴리스 버전 플러그인은 이미지에 넣고 개발 중인 커넥터만 시작 시 설치하는 방식으로 나누는 편이 낫다고 봅니다.
- **영구 실패의 재시도**: Avro Converter 패치를 네트워크 계열 예외에만 적용하도록 좁히면, 스키마 불일치는 바로 실패해 원인을 더 빨리 드러냅니다.
