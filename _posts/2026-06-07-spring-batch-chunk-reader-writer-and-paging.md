---
title: "Spring Batch chunk는 어디까지가 '한 번'인가 — Reader fetch · chunk commit · 페이징 전략"
date: 2026-06-07 23:28:00 +0900
categories: [Spring]
tags: [spring-batch, chunk, item-reader, item-writer, paging, pagination, no-offset, cursor, mongodb, skip]
toc: true
toc_sticky: true
---

## 개요

Spring Batch의 chunk 지향(chunk-oriented) 스텝을 쓰다 보면 거의 누구나 한 번은 헷갈리는 지점이 있다.

```kotlin
StepBuilder("someStep", jobRepository)
    .chunk<In, Out>(1000, transactionManager)   // 이 1000은?
    .reader(reader)                              // reader는 한 번에 몇 건 읽지?
    .writer(writer)                              // writer는 몇 건씩 받지?
    .build()
```

> "reader가 한 번에 1000건을 읽고, `.chunk(1000)` 때문에 1000건씩 1000번 반복하는 건가?"

결론부터 말하면 **둘 다 아니다.** 여기엔 의미가 다른 숫자가 최소 두 개 숨어 있고, 그게 우연히 같은 값일 때 더 헷갈린다. 이 글에서는

1. chunk 지향 스텝이 실제로 어떻게 도는지 (`read()`는 1건씩이다)
2. **"Reader의 DB fetch 크기"** 와 **"chunk = commit 단위"** 가 별개의 손잡이라는 것
3. Reader의 페이징 전략 — **offset(skip)** vs **no-offset(cursor)** 의 성능·정합성 차이

를 정리한다. 예시는 특정 도메인과 무관하게, "원본 이벤트를 읽어 가공 저장하는 배치"라는 일반적인 상황으로 든다.

---

## 1. chunk 지향 스텝은 어떻게 도는가

chunk 지향 스텝의 핵심 인터페이스는 셋이다.

| 컴포넌트 | 단위 | 역할 |
|---|---|---|
| `ItemReader<T>` | **1건** (`read(): T?`) | 다음 아이템 하나를 반환, 끝나면 `null` |
| `ItemProcessor<I,O>` | 1건 | (선택) 변환/필터 |
| `ItemWriter<T>` | **chunk 묶음** (`write(Chunk<T>)`) | chunk 단위로 한 번에 기록 |

가장 먼저 잡아야 할 사실: **`ItemReader.read()`는 한 번에 1건만 반환한다.** 시그니처가 `read(): T?`로 단수다. "한 번에 1000건"이 아니다.

스텝의 실제 루프는 (간략화) 이렇다.

```
loop {
    chunk = []
    repeat (chunkSize 번) {
        item = reader.read()        // 1건씩 호출
        if (item == null) break     // 더 없으면 종료
        chunk += processor.process(item)
    }
    if (chunk is empty) break
    writer.write(chunk)             // 모인 chunk를 한 번에 기록
    transaction.commit()            // 여기서 커밋
}
```

즉 `.chunk(N)`의 N은 **"`read()`를 N번 호출해 모은 다음, writer에 한 번 넘기고 트랜잭션을 commit하는 단위"** 다. 흔히 **commit interval**이라고 부른다.

그래서 "**1000건씩 1000번 반복**"은 틀렸다. 반복(=chunk=commit) 횟수는 1000 고정이 아니라

```
chunk 수 = ceil(전체 건수 / chunkSize)
```

다. 전체가 25만 건이면 250번 commit, 5천 건이면 5번 commit. 데이터 양에 종속된다.

---

## 2. 헷갈리는 진짜 이유 — "두 개의 N"

위 루프만 보면 `read()`가 1건씩이니, 1000건짜리 chunk를 만들려면 DB를 1000번 때리는 것 아닌가 싶다. 보통은 아니다. **Reader가 내부에서 페이지 단위로 미리 버퍼링**하기 때문이다.

여기서 두 번째 N이 등장한다. 많은 Reader는 "DB에서 한 번에 끌어올 행 수(page/fetch size)"를 따로 가진다.

```kotlin
// 커스텀 커서 Reader (개념 예시)
abstract class NoOffsetReader<T : HasCursorId<C>, C>(
    private val fetchSize: Int = 1000,                       // ← (A) DB fetch 크기
    private val readFunction: (cursorId: C?, limit: Int) -> List<T>,
) : ItemReader<T> {

    private var buffer: List<T> = emptyList()
    private var index = 0
    private var lastCursorId: C? = null
    private var finished = false

    override fun read(): T? {                                // ← 1건씩 반환
        if (finished) return null

        if (index >= buffer.size) {                          // 버퍼 소진 시에만
            buffer = readFunction(lastCursorId, fetchSize)   // DB에서 fetchSize 만큼 fetch
            if (buffer.isEmpty()) { finished = true; return null }
            index = 0
            lastCursorId = buffer.last().cursorId
        }
        return buffer[index++]                               // 버퍼에서 1건 꺼내 반환
    }
}
```

`read()`는 여전히 1건씩 반환하지만, 실제 DB 조회는 **버퍼가 비었을 때만** 일어나고 한 번에 `fetchSize`만큼 가져온다. 이렇게 두 개의 N이 생긴다.

| 구분 | 어디에 있나 | 의미 | 서로 독립? |
|---|---|---|---|
| **(A) fetch size** | Reader 내부 | DB 쿼리 1번에 가져올 행 수 (메모리 버퍼) | ✅ |
| **(B) chunk size** | `.chunk(N)` (스텝) | writer 호출 + 트랜잭션 commit 단위 | ✅ |

(A)와 (B)는 **개념적으로 완전히 별개의 손잡이**다. fetch=5000, chunk=1000으로 두면 "DB 1번 조회 → commit 5번"이 되고, fetch=1000, chunk=1000이면 "DB 한 페이지 = 한 chunk = 한 트랜잭션"으로 경계가 딱 맞아떨어진다. 많은 코드가 두 값을 같은 상수로 쓰는 바람에 둘이 같은 개념처럼 보이지만, 그렇지 않다.

> 정리: **`.chunk(N)`은 commit 단위, Reader의 fetch/page size는 DB 한 번에 읽는 양.** 둘은 다른 것이고 값이 달라도 된다.

---

## 3. Reader의 페이징 전략 — offset vs no-offset

전체 데이터를 메모리에 한 번에 올리지 않는다는 건 같지만, "다음 페이지를 어떻게 집어오느냐"에 따라 Reader는 크게 두 부류로 갈린다. 이 선택이 대용량에서 성능과 정합성을 가른다.

### 3-1. offset(skip) 기반 — Spring 내장 페이징 Reader들

`JpaPagingItemReader`, `RepositoryItemReader`, (deprecated된) `MongoItemReader` 등 Spring이 제공하는 페이징 Reader들은 `AbstractPaginatedDataItemReader`를 상속하며 **page 번호 → offset 변환** 방식으로 동작한다.

```
page 0 → offset 0     limit 100
page 1 → offset 100   limit 100
page 2 → offset 200   limit 100
...
page k → offset k*100 limit 100   ← 매번 앞쪽 k*100건을 세고 버린다
```

RDB라면 `LIMIT ... OFFSET ...`, MongoDB라면 `skip(k*pageSize).limit(pageSize)`로 번역된다.

### 3-2. no-offset(cursor) 기반 — 직접 만드는 커서 Reader

2장의 `NoOffsetReader`가 이 방식이다. offset을 쓰지 않고, **인덱스된 커서 필드의 범위 조건**으로 다음 구간을 집어온다.

```
1페이지: WHERE ... AND id > NULL        ORDER BY id LIMIT 100
2페이지: WHERE ... AND id > {직전 last}  ORDER BY id LIMIT 100   ← 인덱스로 점프
```

`skip`이 항상 0이라 페이지가 깊어져도 비용이 늘지 않는다.

---

## 4. offset 페이징의 두 가지 함정

직접 겪고 나서야 체감하는 부분이다. offset 페이징은 작고 고정된 데이터에는 충분하지만, 대용량 배치에서는 두 가지로 무너진다.

### 함정 1 — offset이 커질수록 느려진다 (RDB·Mongo 공통)

`OFFSET N` / `skip(N)`은 **결과 셋 처음부터 N건을 세면서 버린 뒤** 반환한다. RDB의 고전적 문제로 알려져 있지만, **MongoDB도 동일하다.** MongoDB 공식 문서도 *"As the offset increases, `cursor.skip()` will become slower"* 라고 명시한다. 구조적으로 `skip(N)`은 O(N)이다.

배치처럼 처음부터 끝까지 전부 페이징하면 비용이 누적된다.

```
page 0   skip(0)
page 1   skip(100)
page 2   skip(200)
...      ← skip 총합 ≈ 100 + 200 + ... ≈ N²/(2·pageSize)
```

전체 N건을 pageSize로 끝까지 읽으면 **skip 총량이 O(N²)** 가 된다. 하루 10만 건이면 skip만 누적 약 5천만 건을 스캔하는 셈. 뒤로 갈수록 급격히 느려진다.

> 인덱스가 있으면? sort 필드에 인덱스가 있으면 skip이 문서 대신 인덱스 엔트리를 걷어 더 싸지긴 하지만, **여전히 N개를 걸어야 하므로 O(N)** 이다. offset 방식인 한 근본은 안 바뀐다.

### 함정 2 — sort 없는 페이징은 누락/중복을 부른다

성능보다 더 위험한 함정이다. **페이지마다 별도의 쿼리가 나간다.** page 0, 1, 2…가 각각 독립된 `find`/`select`다. 이때 **정렬(sort)이 없으면** DB는 쿼리 간 동일한 순서를 보장하지 않는다. 그러면 페이지 경계에서 **문서가 누락되거나 중복**될 수 있다.

여기서 흔한 함정 하나. Spring Batch의 `MongoItemReader`는 "sort가 필수"라고 알려져 있는데, 실제 소스를 보면 **JSON 문자열 쿼리일 때만** 강제한다.

```java
// org.springframework.batch.item.data.MongoItemReader (Spring Batch 5.x)
@Override
public void afterPropertiesSet() throws Exception {
    Assert.state(template != null, "...");
    Assert.state(type != null, "...");
    Assert.state(queryString != null || query != null, "A query is required.");

    if (queryString != null) {
        Assert.state(sort != null, "A sort is required.");  // ← queryString 경로만 검사
    }
}
```

즉 `setQuery(Query)`(Query 객체)로 넘기면 이 검증을 **우회**해서, sort를 안 걸어도 기동된다. 그리고 그 경로의 `doPageRead()`는 이렇다.

```java
else {
    Pageable pageRequest = PageRequest.of(page, pageSize);  // ← Sort 인자 없음!
    query.with(pageRequest);
    return template.find(query, type).iterator();
}
```

`PageRequest.of(page, pageSize)`는 정렬 없는 Pageable → `skip(page*pageSize).limit(pageSize)`로만 번역된다. 넘긴 Query에도 `.with(Sort)`가 없으면 **어디에도 sort가 없는 채로** 페이지를 넘게 된다. 컴파일·기동은 멀쩡히 되지만, 데이터가 많아지면 조용히 누락/중복이 생길 수 있는 잠재 버그다.

최소한 `_id`처럼 **유니크하고 불변인 필드로 sort**를 걸어야 안전하다. (과거의 고정된 구간만 읽고, 그 구간 데이터가 잡 실행 중 변하지 않는다면 실질 위험은 작지만, "안전하다"와 "운이 좋다"는 다르다.)

---

## 5. 근본 해결 — no-offset(cursor) 페이징

함정 1·2를 한 번에 없애는 방법이 2장의 커서 Reader다. 핵심은 offset 대신 **인덱스된 유니크 필드의 범위 조건 + 그 필드 정렬**이다.

```sql
-- offset 방식 (느리고 불안정)
SELECT * FROM event
WHERE created_date = :date
ORDER BY id          -- 정렬 없으면 더 위험
LIMIT 100 OFFSET 200000   -- 깊을수록 O(N)

-- no-offset 방식 (커서)
SELECT * FROM event
WHERE created_date = :date
  AND id > :lastId   -- 직전 페이지 마지막 id
ORDER BY id
LIMIT 100            -- 인덱스로 점프, skip 0
```

| | offset(skip) Reader | no-offset(cursor) Reader |
|---|---|---|
| 다음 페이지 | `OFFSET k*size` / `skip` | `id > lastId` |
| sort | 빠지기 쉬움 (정합성 위험) | 커서 필드로 **필수 정렬** |
| 깊은 페이지 비용 | O(N), 전체 O(N²) | O(log N) 탐색, 전체 O(N log N) |
| 누락/중복 | sort 없으면 가능 | 없음 |
| 구현 | Spring 내장 사용 | 직접 구현(또는 라이브러리) |

대신 no-offset에는 전제 조건이 있다.

- 커서 필드가 **유니크 + 단조** 여야 한다 (보통 PK, `_id`, 또는 정렬 가능한 ULID 등).
- 그 필드에 **인덱스**가 있어야 점프가 싸진다.
- 복합 정렬이 필요하면 `(sortKey, id)` 복합 커서로 확장해야 한다 (tie-break).

---

## 6. chunk size는 어떻게 잡나

마지막으로 (B) chunk size 튜닝 감각만 정리한다. 정답 숫자는 없고 트레이드오프다.

- **크게** → commit 횟수 감소(오버헤드↓), 처리량↑. 단 한 트랜잭션이 커져 **메모리 사용↑**, 락 보유 시간↑, 실패 시 롤백·재처리 범위↑.
- **작게** → 트랜잭션이 가볍고 실패 영향이 작지만, commit이 잦아 **오버헤드↑**.
- **fetch size와의 관계** → (A) fetch size를 (B) chunk size의 정수배로 맞추면 "DB 조회 경계 = commit 경계"가 깔끔히 정렬돼 직관적이다. 꼭 같을 필요는 없다.
- 보통 수백~수천 사이에서 시작해, 데이터 크기·건당 메모리·writer 대상(RDB bulk insert 등)을 보고 조정한다.

---

## 정리

- `ItemReader.read()`는 **1건씩** 반환한다. "한 번에 1000건"이 아니다.
- `.chunk(N)`의 N은 **commit interval** — `read()`를 N번 모아 writer에 한 번 넘기고 트랜잭션을 커밋하는 단위다. 반복 횟수는 `ceil(전체/N)`이지 N이 아니다.
- Reader의 **fetch/page size**(DB 한 번에 읽는 양)와 **chunk size**(commit 단위)는 **별개의 손잡이**다. 값이 같아도 개념은 다르다.
- 페이징 전략에서 **offset(skip)** 은 깊어질수록 O(N)(전체 O(N²))이고, **sort를 빠뜨리면 누락/중복**까지 난다. RDB·MongoDB 공통이다.
- 대용량 배치는 **no-offset(cursor)** 페이징 — 인덱스된 유니크 필드 범위 조건 + 정렬 — 으로 가는 게 정석이다. 작고 고정된 구간이면 offset도 충분하다.
