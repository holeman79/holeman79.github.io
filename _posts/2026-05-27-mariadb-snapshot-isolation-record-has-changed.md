---
title: "MariaDB 11.6+ snapshot isolation과 'Record has changed since last read' 디버깅"
date: 2026-05-27 23:14:00 +0900
categories: [Database]
tags: [mariadb, innodb, snapshot-isolation, transaction-isolation, repeatable-read, upsert, react-strictmode, kotlin, exposed]
toc: true
toc_sticky: true
---

## 개요

개인 주식 포트폴리오 사이드 프로젝트(Kotlin + Spring Boot + Exposed + MariaDB)에서 페이지를 열 때마다 간헐적으로 `500 INTERNAL_ERROR`가 떴다. 로그를 까보니 원인은 이 한 줄이었다.

```
java.sql.SQLException: Record has changed since last read in table 'stock_valuation'
```

처음엔 "동시에 같은 행을 지웠다 넣어서 그런가" 싶어 `DELETE + INSERT`를 원자적 `upsert`로 바꿨는데도 똑같이 터졌다. 범인은 애플리케이션 코드가 아니라 **MariaDB 11.6.2부터 기본 활성화된 `innodb_snapshot_isolation`** 라는, 비교적 최근에 바뀐 동작이었다. 이 글은 그 디버깅 과정과, 같은 함정에 빠지지 않기 위한 정리다.

---

## 1. 증상

프론트엔드(React)가 대시보드 진입 시 보유 종목을 새로고침한다. 여러 새로고침 API를 병렬로 호출하는데, 그중 일부가 `500`을 반환했다. 백엔드 전역 예외 핸들러가 잡은 원본 예외는 다음과 같았다.

```
ERROR ... 예상치 못한 오류: java.sql.BatchUpdateException:
  Record has changed since last read in table 'stock_valuation'
org.jetbrains.exposed.exceptions.ExposedSQLException:
  java.sql.SQLException: Record has changed since last read in table 'stock_valuation'
  SQL: [Failed on expanding args for ... statement]
```

핵심 키워드는 **`Record has changed since last read`**. MySQL/MariaDB 에러 코드로는 **`1020 (ER_CHECKREAD)`** 다.

로그의 요청 타임스탬프를 보면 더 분명했다. **같은 새로고침 엔드포인트가 동일한 밀리초에 두 번씩** 들어오고 있었다.

```
22:37:13.053  POST /api/.../refresh   (thread exec-8)
22:37:13.053  POST /api/.../refresh   (thread exec-9)   ← 같은 ms, 중복
```

즉 동일한 요청이 동시에 2벌 실행되면서, 두 트랜잭션이 **같은 행**을 동시에 쓰려다 충돌하고 있었다.

---

## 2. 첫 번째 오진 — "DELETE + INSERT 경합이겠지"

문제의 테이블에 쓰는 코드는 전형적인 "지우고 다시 넣기" 패턴이었다. (Exposed 기준)

```kotlin
fun upsertAll(entities: List<StockValuationEntity>) {
    if (entities.isEmpty()) return
    // 1) 이번 배치에 해당하는 행을 먼저 지우고
    StockValuationTable.deleteWhere {
        StockValuationTable.stockCode inList entities.map { it.stockCode }
    }
    // 2) 새로 넣는다
    StockValuationTable.batchInsert(entities) { entity ->
        this[StockValuationTable.stockCode] = entity.stockCode
        this[StockValuationTable.per] = entity.per
        // ...
    }
}
```

`DELETE` 후 `INSERT`라는 **두 문장 사이의 윈도우**에서 다른 트랜잭션이 끼어들어 충돌한다고 생각했다. 그래서 MariaDB의 네이티브 upsert(`INSERT ... ON DUPLICATE KEY UPDATE`)로 바꿨다. Exposed 0.57.0의 `batchUpsert`를 쓰면 한 문장으로 끝난다.

```kotlin
fun upsertAll(entities: List<StockValuationEntity>) {
    if (entities.isEmpty()) return
    StockValuationTable.batchUpsert(entities) { entity ->   // DELETE 없이 원자적 upsert
        this[StockValuationTable.stockCode] = entity.stockCode
        this[StockValuationTable.per] = entity.per
        // ...
    }
}
```

생성되는 SQL은 다음과 같다.

```sql
INSERT INTO stock_valuation (stock_code, per, ...)
VALUES (?, ?, ...)
ON DUPLICATE KEY UPDATE
    per = VALUES(per), ... ;
```

`DELETE`가 사라졌으니 경합 윈도우도 사라질 거라 기대했다. **그런데 똑같이 터졌다.** 이번엔 `BatchUpdateException`으로 감싸였을 뿐, 메시지는 동일한 `Record has changed since last read`.

여기서 깨달았다. **DELETE가 문제가 아니다.** 두 트랜잭션이 같은 행을 동시에 건드리는 것 자체가 에러로 떨어지고 있다.

---

## 3. 진짜 원인 — `innodb_snapshot_isolation`

DB 버전과 격리 설정을 직접 확인했다.

```sql
SELECT VERSION();
-- 11.8.3-MariaDB

SHOW VARIABLES LIKE 'innodb_snapshot_isolation';
-- innodb_snapshot_isolation = ON

SHOW VARIABLES LIKE 'transaction_isolation';
-- transaction_isolation = REPEATABLE-READ
```

`innodb_snapshot_isolation`. 이게 핵심이었다.

### 무엇이 바뀌었나

이 변수는 **MariaDB 11.6.2에서 도입되어 기본값 `ON`** 으로 들어왔다. REPEATABLE READ(및 SERIALIZABLE)에서 InnoDB에 **진짜 스냅샷 격리(true snapshot isolation)** 를 부여한다.

핵심 동작은 **first-committer-wins(또는 first-updater-wins)** 다.

- 트랜잭션 T1, T2가 같은 시점에 시작해 각자 같은 행의 스냅샷을 읽는다.
- T1이 그 행을 수정하고 **먼저 커밋**한다.
- T2가 같은 행을 수정하려는 순간, "내가 읽은 스냅샷 이후 이 행이 바뀌었다"는 것을 InnoDB가 감지한다.
- **예전(snapshot isolation OFF)** 이라면: T2는 잠깐 락을 기다렸다가, 최신 커밋된 버전 위에서 그냥 진행했다(이른바 semi-consistent read, last-committer-wins).
- **지금(snapshot isolation ON)** 이라면: T2는 진행하지 않고 즉시 **`ERROR 1020: Record has changed since last read`** 를 던진다.

즉, **조용히 넘어가던 write-write 충돌이 이제 명시적인 에러로 바뀌었다.** 이건 "잃어버린 갱신(lost update)"과 write skew를 막아주는, 더 엄격하고 올바른 동작이다. 문제는 기존 코드가 이 동작을 가정하고 작성되지 않았다는 점이다.

### 그래서 DELETE든 upsert든 무관했다

충돌은 특정 SQL 문장의 문제가 아니라 **"두 트랜잭션이 같은 행을 동시에 변경한다"는 사실 그 자체**에서 발생한다. `DELETE+INSERT`를 `INSERT ... ON DUPLICATE KEY UPDATE`로 바꿔도, 같은 행을 동시에 건드리는 한 똑같이 1020이 난다.

> 참고로 이 동작은 **REPEATABLE READ / SERIALIZABLE에서만** 적용된다. READ COMMITTED에서는 매 문장이 새 스냅샷을 읽으므로 이 에러가 발생하지 않는다. 이 점이 뒤의 해결책에서 중요하다.

---

## 4. 왜 같은 행을 동시에 썼나 — React StrictMode

그럼 단일 사용자 로컬 앱에서 왜 같은 행이 동시에 써졌을까? 로그의 "같은 ms에 중복 요청"이 단서였다.

프론트는 페이지 진입 시 `useEffect`로 새로고침을 호출하고 있었다.

```tsx
useEffect(() => {
  refresh()   // 내부에서 여러 refresh API를 Promise.all로 병렬 호출
}, [])
```

그리고 `main.tsx`에는 `<StrictMode>`가 켜져 있었다. **React 18의 StrictMode는 개발 모드에서 컴포넌트를 mount → unmount → mount로 한 번 더 마운트**한다(effect의 정리 누락을 잡아내기 위한 의도된 동작). 그 결과 `useEffect`가 두 번 실행되고, **동일한 새로고침이 동시에 2벌** 날아간다. 두 벌은 같은 종목 행을 동시에 upsert → 1020.

확인을 위해 같은 엔드포인트를 동시에 2번 호출해봤더니 한쪽이 500으로 재현됐고, 서로 다른 행을 쓰는 엔드포인트끼리 병렬 호출했을 때는 200으로 멀쩡했다. 가설이 맞았다.

### 1차 처방: 중복 호출 제거

가장 직접적인 처방은 프론트에서 페이지 진입 새로고침이 중복으로 나가지 않게 막는 것이다. StrictMode 이중 마운트를 견디는 표준 패턴은 `useRef` 가드다.

```tsx
const didInitialRefresh = useRef(false)
useEffect(() => {
  if (didInitialRefresh.current) return
  didInitialRefresh.current = true
  refresh()
}, [])
```

`ref`는 StrictMode의 unmount/remount 사이에도 보존되므로, 두 번째 마운트에서는 새로고침을 건너뛴다. 이걸 적용하니 실제 앱 경로(새로고침 1회 = 서로 다른 행을 쓰는 API들의 병렬 호출)는 전부 정상 동작했다.

> 덤으로, 중복 호출이 사라지면서 외부 API(시세 제공처)에서 받던 `429 Too Many Requests`도 같이 사라졌다. 같은 뿌리의 증상이었다.

---

## 5. 백엔드를 정말로 동시성-안전하게 만들려면

프론트 가드로 "보고된 증상"은 사라진다. 하지만 백엔드 자체는 여전히 **서로 다른 흐름이 같은 행을 동시에 쓰면**(예: 종목 상세 페이지 진입과 대시보드 새로고침이 같은 종목을 동시에 갱신) 1020을 던질 수 있다. 근본적으로 안전하게 만들려면 선택지는 다음과 같다.

| 방법 | 내용 | 트레이드오프 |
|---|---|---|
| **READ COMMITTED로 격리 낮추기** | 앱 트랜잭션 격리를 RC로. snapshot isolation은 RR/SERIALIZABLE에만 적용되므로 1020 자체가 안 남(대신 일반 row-lock 블로킹) | 외부 데이터 스냅샷 저장 용도엔 RR이 굳이 필요 없어 합리적. RR이 주던 일관성은 포기 |
| **충돌 재시도** | 1020 발생 시 트랜잭션 재시도(`@Retryable` 등) | 스냅샷 격리 직렬화 실패의 정석 대응이나 코드 복잡도 증가 |
| **`innodb_snapshot_isolation = OFF`** | 서버 설정을 꺼서 11.6 이전 블로킹 동작으로 회귀 | 가장 단순하지만 DB 전역 설정. 컨테이너/인프라 밖이면 영속·재현이 어려움 |
| **애플리케이션에서 동시 같은-행 write를 구조적으로 회피** | 프론트 중복 제거 + 흐름 직렬화 | 단일 사용자 로컬 앱이면 현실적으로 충분 |

이 프로젝트는 "외부 데이터(시세/지표)의 로컬 사본을 멱등하게 덮어쓰는" 성격이라, 충돌 시 **last-committer-wins(나중 값으로 덮기)** 가 정확히 원하는 동작이다. 잃어버린 갱신을 걱정할 필요가 없다. 그래서 RR의 엄격한 스냅샷 격리는 이 워크로드에 과하고, READ COMMITTED나 snapshot isolation OFF가 의미상 더 맞는다.

---

## 6. 정리 / 교훈

- **`Record has changed since last read` (ERROR 1020, ER_CHECKREAD)** 는 MariaDB 11.6.2+ 의 **`innodb_snapshot_isolation = ON`**(기본값) 때문에, REPEATABLE READ에서 **같은 행을 동시에 변경하는 두 트랜잭션 중 나중 것**에 던져지는 에러다. write-write 충돌을 조용히 넘기지 않고 명시적으로 실패시키는, 더 올바른 동작이다.
- **DELETE+INSERT를 upsert로 바꾼다고 해결되지 않는다.** 충돌은 특정 SQL이 아니라 "동시 같은-행 변경"이라는 사실에서 나온다.
- 우리 케이스의 트리거는 **React StrictMode 이중 마운트**로 인한 새로고침 중복 호출이었다. `useRef` 가드로 중복을 제거하는 게 1차 처방.
- 백엔드를 근본적으로 안전하게 하려면 **격리 레벨 조정(READ COMMITTED)**, **충돌 재시도**, **snapshot isolation 끄기** 중 워크로드 성격에 맞는 걸 고른다. 멱등한 스냅샷 덮어쓰기라면 RR의 엄격함은 과하다.
- 무엇보다, **DB 메이저 버전 업그레이드는 기본 격리 동작까지 바꿀 수 있다.** "코드는 그대로인데 갑자기 터진다" 싶으면 `SHOW VARIABLES`로 DB가 내 가정대로 동작하는지부터 확인하자.

### 참고

- MariaDB Knowledge Base: `innodb_snapshot_isolation` 시스템 변수, MariaDB 11.6.2부터 도입·기본 활성화
- MySQL/MariaDB 에러 `1020 ER_CHECKREAD`
