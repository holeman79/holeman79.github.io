---
title: "Elasticsearch `_update`의 409 version_conflict — read-then-write의 함정과 retry_on_conflict / painless script"
date: 2026-06-01 21:00:00 +0900
categories: [Database, Elasticsearch]
tags: [elasticsearch, optimistic-concurrency-control, version-conflict, kafka, consumer, painless-script, kotlin, retry, race-condition]
toc: true
toc_sticky: true
---

## 들어가며

오늘 운영 Kafka consumer에서 ERROR 알람이 두 건 떴다. 둘 다 같은 종류의 예외였다.

```
org.elasticsearch.client.ResponseException:
  method [POST], host [...], URI [/asset/_update/{docId}],
  status line [HTTP/1.1 409 Conflict]
{"error":{"root_cause":[{"type":"version_conflict_engine_exception",
  "reason":"[{docId}]: version conflict,
    required seqNo [16951707], primary term [3].
    current document has seqNo [16951710] and primary term [3]"}]}}
```

신기한 점은 Kafka container가 retry로 두 번째 시도에서 매끄럽게 처리해 **데이터 손실은 없었다**는 것이다. 그럼에도 첫 시도가 ERROR로 찍히면서 알람만 울렸다. "retry로 복구됐으니 알람 강등만 하면 되겠지" 싶었는데, 그 자리에 멈추면 같은 사고가 반복될 가능성이 컸다.

이 글은 그 디버깅 과정과, **ES `_update`의 OCC(Optimistic Concurrency Control) 모델을 모르고 짠 read-then-write 코드가 어떻게 동시성 사고를 부르는지**, 그리고 두 단계로 갈라지는 fix(`retry_on_conflict` → painless script)를 정리한다.

> **잠깐, OCC (Optimistic Concurrency Control) 가 뭔지부터**
>
> **OCC(낙관적 동시성 제어)** 는 "동시에 같은 데이터를 수정하는 경우는 드물 거다"라고 **낙관적으로** 가정하고, 락을 걸지 않은 채 일단 작업을 진행한 뒤 **쓰기 직전에만 버전을 비교해 충돌을 검증**하는 동시성 모델이다. RDBMS의 `SELECT FOR UPDATE` 같은 PCC(Pessimistic Concurrency Control, 비관적 동시성 제어)가 "충돌이 자주 날 거다"라고 비관해 미리 락을 잡는 것과 정반대다.
>
> | | OCC | PCC |
> |---|----|----|
> | 가정 | 충돌이 드물다 | 충돌이 자주 난다 |
> | 동시 접근 | 막지 않음 | 락으로 차단 |
> | 충돌 검증 시점 | **쓰기 직전** (버전 비교) | 사전 (락 획득 시) |
> | 충돌 시 | 호출자에게 "충돌!" 알려 재시도 결정 | 대기 또는 deadlock |
> | 장점 | 락 오버헤드 0, 분산 시스템에 적합 | 일관성 보장이 직관적 |
> | 단점 | 충돌 잦으면 재시도 비용 폭증 | 락 대기/데드락 |
>
> ES는 분산 시스템이라 노드 간 락이 비싸서 OCC가 자연스러운 선택이다. 모든 문서는 `_seq_no`(쓰기 순번)와 `_primary_term`(현재 primary shard 세대) 두 값으로 버전을 표현하고, 쓰기 시점에 클라이언트가 알고 있던 버전과 ES 측의 현재 버전이 다르면 `version_conflict_engine_exception(409)`을 던진다. 사용자가 본문에서 보게 될 모든 409는 이 모델의 정상 동작이다 — **버그가 아니라 ES가 "당신이 본 버전은 이미 낡았다"라고 알려주는 신호**.

---

## 1. 사건 — Kafka consumer가 ES `_update`에서 409

코드를 단순화하면 이런 모양이다.

```kotlin
@Component
class AssetHideListener(
    private val assetRepository: AssetRepository,
    private val relatedTagCloser: RelatedTagCloser,
) {
    @KafkaListener(topics = ["asset-state-change"])
    fun handle(event: AssetStateChangeEvent) {
        assetRepository.markHidden(event.assetIds)
        if (event.hidden) {
            relatedTagCloser.closeForHiddenAssets(event.assetIds)
        }
    }
}
```

`assetRepository.markHidden`은 `asset` 인덱스의 `hidden` 필드를 true로 update.
`relatedTagCloser.closeForHiddenAssets`는 그 asset이 참조하는 다른 인덱스(예: `certificate`, `cve`)에서도 관련 태그를 정리한다.

문제는 두 번째 단계에 있었다.

```kotlin
@Component
class RelatedTagCloser(
    private val esRepository: EsRepository,
) {
    fun closeForHiddenAssets(assetIds: List<String>) {
        RELATED_INDICES.forEach { index ->
            // ① 같은 docId 의 문서를 읽고
            esRepository.findDocumentsWithTagIds(index, assetIds).forEach { doc ->
                // ② 메모리에서 필터링 후
                val filtered = doc.removeTagsByAssetIds(assetIds)
                // ③ 같은 docId 를 통째로 update
                esRepository.updateDocumentTags(index, doc.docId, filtered)
            }
        }
    }
}
```

그리고 `updateDocumentTags`는 ES client의 partial update를 호출한다.

```kotlin
fun updateDocumentTags(index: String, docId: String, tags: List<Map<String, Any>>) {
    val updateFields = mapOf<String, Any>("tagMessages" to tags)
    esClient.update<Void, Map<String, Any>>(
        { u -> u.index(index).id(docId).doc(updateFields) },   // ← 옵션 없음
        Void::class.java
    )
}
```

평소엔 잘 돌다가, 운영에서 두 번 — 다른 인덱스, 다른 시각에 — 409가 떴다. retry 로그를 보면 Kafka가 1초 뒤 같은 메시지를 다시 폴해서 두 번째 시도에선 깔끔하게 성공했다.

```
09:01:45 ERROR ... version conflict, required seqNo [16951707], current [16951710]
09:01:46 INFO  Seeking to offset 102 for partition asset-state-change-0
09:01:46 INFO  Record in retry and not yet recovered
09:01:46 INFO  Received asset-state-change: hidden=true, assetIds=4
09:01:50 INFO  Asset state updated: hidden=true, count=4
```

ERROR가 단발성이고 retry로 복구되니까 "그냥 OCC 충돌인데 다음 시도에 풀린 케이스"라는 건 직감으로 알았지만, **왜 풀리는지/언제 다시 터지는지를 모르면 그건 디버깅이 아니라 점**이다. 원리부터 짚어보자.

---

## 2. 왜 발생하는가 — ES `_update`의 GET-MODIFY-INDEX와 read-then-write

### 2-1. ES `_update`는 내부적으로 GET → MODIFY → INDEX

ES `_update` API에 partial doc(예: `{"doc": {"hidden": true}}`)을 전달하면 ES가 내부적으로 세 단계를 거친다.

```
1) GET   document by id  → 현재 _source + seqNo + primaryTerm 확보
2) MODIFY 메모리에서 partial doc 을 merge
3) INDEX  merged 결과를 다시 색인 (if_seq_no=GET 시점의 seqNo 로 OCC 검사)
```

이 세 단계는 ES 노드 내부에서 일어나지만 **원자적이지 않다**. GET과 INDEX 사이에 다른 writer가 같은 문서를 색인하면 INDEX 단계에서 seqNo가 어긋나 `version_conflict_engine_exception(409)`이 떨어진다.

### 2-2. `retry_on_conflict`의 기본값은 0

ES `_update`는 이 충돌을 자체적으로 다시 시도해 주는 [`retry_on_conflict`](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update.html#_parameters_3) 파라미터를 받는다. 그런데 **기본값이 0**이다. 명시적으로 안 주면 한 번 실패하고 그대로 클라이언트에게 409를 던진다. 우리 코드도 안 줬다.

### 2-3. read-then-write는 race window를 더 넓힌다

여기까지가 단일 `_update` 호출의 OCC 모델이다. 우리 코드의 진짜 문제는 그 위에 한 단계 더 있다.

```kotlin
val docs = esRepository.findDocumentsWithTagIds(index, assetIds)   // ① application 단의 GET
docs.forEach { doc ->
    val filtered = doc.removeTagsByAssetIds(assetIds)              // ② application 메모리에서 modify
    esRepository.updateDocumentTags(index, doc.docId, filtered)    // ③ application 단의 PUT-style update
}
```

ES `_update`는 자기 안에서 GET-MODIFY-INDEX를 1ms 안에 끝내는데, 우리 코드는 application 단에서 같은 패턴을 **수십~수백 ms**에 걸쳐 한다. 그 사이에 다른 writer가 같은 문서를 한 번이라도 update하면, 우리가 보낸 update의 메모리 상태는 이미 stale이다.

> 거기에 ES `_update`는 한 번 더 GET을 수행하므로(스텝 1), 운이 나쁘면 두 곳에서 다 충돌이 잡힌다.

오늘 운영 로그의 두 케이스를 다시 보자.

| 케이스 | required seqNo | current seqNo | diff | primaryTerm | 해석 |
|--------|---------------|--------------|------|------------|------|
| asset 인덱스 | 16951707 | 16951710 | **+3** | 3 = 3 | 짧은 시간에 같은 문서가 3번 더 update됨 → 동시 writer 존재 |
| certificate 인덱스 | 1404278 | 2777620 | **+1373342** | **1 → 3** | 이 문서는 매우 오래된 시점에 read됨. primaryTerm이 바뀐 건 shard recovery가 있었다는 뜻 |

특히 두 번째 케이스는 application 단의 read와 write 사이가 길었거나, **이미 read된 데이터를 어딘가 캐시해 두고 한참 후에 update**한 흔적으로 보였다. 어느 쪽이든 동시 writer가 있는 데이터 위에서 read-then-write를 하면 시간 문제로 충돌이 난다.

---

## 3. 단기 fix — `retry_on_conflict`로 ES가 자체 재시도하게 둔다

ES `_update`의 OCC 충돌은 대부분 짧은 race window에서 발생하므로, ES 측에서 몇 번만 자동 재시도해 주면 99% 사라진다.

```kotlin
fun updateDocumentTags(index: String, docId: String, tags: List<Map<String, Any>>) {
    val updateFields = mapOf<String, Any>("tagMessages" to tags)
    esClient.update<Void, Map<String, Any>>(
        { u -> u
            .index(index)
            .id(docId)
            .doc(updateFields)
            .retryOnConflict(3)              // ← 추가
        },
        Void::class.java
    )
}
```

`retryOnConflict(3)`은 ES 노드 안에서 GET-MODIFY-INDEX 사이클을 최대 3번 더 돌리라는 뜻이다. 충돌이 풀리면 200, 그래도 안 풀리면 클라이언트에게 409를 던진다.

이건 진짜 race가 아니라 **race window가 짧은 정상 동작**에 대한 안전망이다. application 코드/Kafka retry 없이 ES가 알아서 처리한다.

> 주의: `retry_on_conflict`는 update API 전용 옵션이다. index/create API에는 없고, 그쪽은 `if_seq_no/if_primary_term`을 명시해 사용자 코드가 직접 재시도해야 한다.

---

## 4. 더 나아간 fix — painless script로 application read-then-write를 제거

`retry_on_conflict`만으로도 운영에서 99% 사라진다. 하지만 우리 read-then-write 패턴은 여전히 남는다.

```kotlin
val docs = esRepository.findDocumentsWithTagIds(index, assetIds)   // ① application GET
docs.forEach { doc ->
    val filtered = doc.removeTagsByAssetIds(assetIds)              // ② application MODIFY
    esRepository.updateDocumentTags(index, doc.docId, filtered)    // ③ application PUT
}
```

② 단계의 비즈니스 로직(`removeTagsByAssetIds`)이 단순한 컬렉션 필터링이라면, **ES 측에 painless script로 위임하는 것이 race window 자체를 없앤다**.

```kotlin
fun removeTagsByAssetIdsAtomic(index: String, docId: String, assetIds: List<String>) {
    esClient.update<Void, Map<String, Any>>(
        { u -> u
            .index(index)
            .id(docId)
            .script { s -> s.inline { i -> i
                .lang("painless")
                .source("""
                    if (ctx._source.tagMessages != null) {
                        ctx._source.tagMessages.removeIf(m -> params.ids.contains(m.assetId));
                    }
                """.trimIndent())
                .params("ids", JsonData.of(assetIds))
            }}
            .retryOnConflict(3)
        },
        Void::class.java
    )
}
```

이러면 다음 두 가지가 동시에 해결된다.

1. **application 단의 read가 사라진다** — `findDocumentsWithTagIds` 호출이 통째로 빠진다. 네트워크 라운드트립도 절약.
2. **MODIFY가 ES 노드 안에서 일어난다** — GET과 INDEX 사이의 짧은 ES 내부 사이클 안에서 처리되고, `retry_on_conflict`까지 더해지면 사실상 conflict가 안 보인다.

> painless script는 도메인 로직이 너무 복잡해지면 디버깅이 어렵다. 단순 컬렉션 조작(`removeIf`, `add`, `replace`)까지가 적정선이고, "현재 상태에 따라 분기/여러 인덱스를 봐야 하는" 로직이라면 application 단에 두는 게 유지보수에 낫다.

---

## 5. 부수 개선 — Kafka retry 첫 실패는 WARN으로 강등

ES 측 fix를 다 적용해도, 운영에서 진짜로 conflict 폭주가 일어나면 알람을 받아야 한다. 다만 **첫 시도 실패가 다음 시도에 성공한 케이스를 ERROR로 찍을 필요는 없다.**

Spring Kafka의 `CommonErrorHandler`/`RetryListener`를 커스텀해 다음과 같이 분기:

```kotlin
class ConflictAwareRetryListener : RetryListener {
    override fun failedDelivery(
        record: ConsumerRecord<*, *>,
        ex: Exception,
        deliveryAttempt: Int,
    ) {
        if (isVersionConflict(ex) && deliveryAttempt < MAX_ATTEMPTS) {
            logger.warn { "ES conflict on attempt $deliveryAttempt, will retry: ${ex.message}" }
        } else {
            logger.error(ex) { "Delivery failed on attempt $deliveryAttempt" }
        }
    }
    private fun isVersionConflict(ex: Throwable): Boolean =
        generateSequence<Throwable>(ex) { it.cause }
            .any { it.message?.contains("version_conflict_engine_exception") == true }
}
```

이전 글에서 다뤘던 [`[API 4XX]` vs `[API ERROR]` 분류]({% post_url 2026-05-28-es-query-string-parser-2-4xx-classification %})와 같은 사상이다. 운영 ERROR 알람은 "사람이 봐야 하는 상황"만 잡혀야 신뢰가 회복된다.

---

## 6. 정리 — "retry로 풀렸으니 OK"에 머물지 않기

오늘의 사고는 데이터 손실이 없었다. 그런데 멈춰서 "retry가 잘 동작한다"고 결론 내렸다면 다음 시점에 같은 패턴이 더 큰 규모로 터지거나, retry가 풀어내지 못하는 진짜 race를 만났을 때 디버깅 단서가 없었을 것이다.

정리하면 다음 4단계 체크리스트.

1. **ES `_update`를 호출하는 모든 코드에 `retry_on_conflict` 가 있는가?** — 없으면 한 줄 추가.
2. **`findDoc → modify → updateDoc` 패턴이 있는가?** — 있다면 painless script로 치환 가능한지 검토.
3. **여러 writer가 같은 문서를 update할 수 있는가?** — Kafka topic 분리, partition key 통일, 또는 단일 writer 보장.
4. **Kafka retry 첫 시도 실패가 ERROR로 찍히는가?** — 첫 실패는 WARN, 최종 실패만 ERROR로 분리.

ES OCC는 잘 만들어진 메커니즘이지만, 사용자가 그 모델을 모른 채 application 단에서 read-then-write를 하면 그 안전망이 무력해진다. 충돌은 race condition이 발생할 수 있다는 신호이고, **신호를 retry로 묻어두는 대신 동시성 모델을 코드에 명시적으로 드러내는 것**이 핵심이다.
