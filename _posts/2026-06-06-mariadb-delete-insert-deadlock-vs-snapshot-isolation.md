---
title: "snapshot isolation을 껐더니 이번엔 데드락(1213) — DELETE+INSERT를 upsert로"
date: 2026-06-06 12:50:00 +0900
categories: [Database, RDB]
tags: [mariadb, innodb, deadlock, gap-lock, insert-intention-lock, repeatable-read, upsert, react-strictmode, kotlin, exposed]
toc: true
toc_sticky: true
---

## 개요

[지난 글](/categories/rdb/)에서 같은 주식 포트폴리오 사이드 프로젝트(Kotlin + Spring Boot + Exposed + MariaDB)의 `Record has changed since last read`(ERROR 1020)를 다뤘다. 결론은 *"MariaDB 11.6.2+의 `innodb_snapshot_isolation = ON` 때문이고, DELETE든 upsert든 무관하며, snapshot isolation을 끄거나 격리 레벨을 낮추는 게 답"* 이었다.

그 권고대로 **`innodb_snapshot_isolation`을 `OFF`로 내렸다.** 1020은 사라졌다. 그런데 한동안 잠잠하던 종목 상세 페이지가 이번엔 다른 에러로 터졌다.

```
java.sql.SQLTransactionRollbackException: (conn=1764)
  Deadlock found when trying to get lock; try restarting transaction
```

`1020`이 아니라 **`1213` (ER_LOCK_DEADLOCK)**. 같은 트리거(React StrictMode 이중 호출), 같은 "동시에 같은 행 쓰기"인데, **DB 설정 한 줄을 바꾼 탓에 실패 모드와 올바른 처방이 통째로 뒤집혔다.** 지난 글이 "upsert는 무관하다"였다면, 이번 글은 "upsert가 바로 답이다"이다. 그 역설을 정리한다.

---

## 1. 증상 — 종목 상세 진입 시 500, 이번엔 `Deadlock found`

종목 상세 페이지에 들어가면 간헐적으로 `500 INTERNAL_ERROR`. 백엔드 로그의 원본 예외는 이랬다.

```
ERROR ... 예상치 못한 오류: java.sql.SQLTransactionRollbackException:
  (conn=1764) Deadlock found when trying to get lock; try restarting transaction
org.jetbrains.exposed.exceptions.ExposedSQLException:
  java.sql.SQLTransactionRollbackException: Deadlock found when trying to get lock ...
    at com.holeman.sp.domain.stock.dao.StockDao.upsert(StockDao.kt:47)
    at com.holeman.sp.domain.stock.domain.StockInfoRefresher.refreshOne(...)
    at com.holeman.sp.domain.stock.application.StockMarketDataRefreshService.refreshOne(...)
  SQL: [INSERT INTO stock (ticker, market, name, market_cap_amount, market_cap_currency, synced_at)
        VALUES (?, ?, ?, ?, ?, ?)]
```

핵심 키워드는 **`Deadlock found when trying to get lock`**, 에러 코드 **`1213 (ER_LOCK_DEADLOCK)`**. 문제의 SQL은 `stock` 테이블의 `INSERT`였다.

먼저 DB 상태부터 확인했다.

```sql
SELECT VERSION();
-- 11.8.3-MariaDB

SHOW VARIABLES LIKE 'innodb_snapshot_isolation';
-- innodb_snapshot_isolation = OFF      ← 지난 글 권고대로 꺼둔 상태

SHOW VARIABLES LIKE 'transaction_isolation';
-- transaction_isolation = REPEATABLE-READ
```

snapshot isolation이 `OFF`라 1020(`Record has changed`)은 더 이상 나지 않는다. 대신 **고전적인 InnoDB 락 동작으로 회귀**했고, 그 위에서 옛 코드가 새로운 방식으로 터진 것이다.

---

## 2. 지난 글과 무엇이 다른가 — 1020 vs 1213

같은 프로젝트, 같은 페이지, 같은 StrictMode 트리거인데 에러가 다르다.

| | 지난 글 (1020) | 이번 글 (1213) |
|---|---|---|
| 에러 | `Record has changed since last read` | `Deadlock found when trying to get lock` |
| MySQL 코드 | `1020` (ER_CHECKREAD) | `1213` (ER_LOCK_DEADLOCK) |
| `innodb_snapshot_isolation` | **ON** | **OFF** |
| 메커니즘 | snapshot isolation의 write-write 충돌 감지 (first-committer-wins) | 갭락 + insert-intention 락의 교착 사이클 |
| DELETE vs upsert | **무관** (SQL 종류와 무관하게 충돌 감지) | **결정적** (DELETE의 갭락이 원인) |
| 처방 | 격리 낮추기 / snapshot isolation OFF / 재시도 | DELETE 제거 → 원자적 upsert |

지난 글의 권고(snapshot isolation OFF)를 적용하면서 1020은 해결됐지만, 그 결과 **클래식 락 모드로 돌아오자 옛 `DELETE + INSERT` 패턴이 이번엔 데드락(1213)을 일으켰다.** 한 문제를 닫으면서 옆문을 연 셈이다.

---

## 3. 왜 `DELETE + INSERT`가 데드락을 만드나

문제의 `upsert`는 이름만 upsert였고, 실제로는 "지우고 다시 넣기"였다. (Exposed)

```kotlin
fun upsert(entity: StockEntity) {
    // 1) 같은 (ticker, market) 행을 먼저 지우고
    StockTable.deleteWhere {
        (StockTable.ticker eq entity.ticker) and (StockTable.market eq entity.market)
    }
    // 2) 새로 넣는다
    StockTable.insert { row ->
        row[ticker] = entity.ticker
        row[market] = entity.market
        // ...
    }
}
```

REPEATABLE READ(snapshot isolation OFF)에서 이 두 문장이 동시에 두 트랜잭션에서 돌면 다음이 벌어진다.

1. **`DELETE`** 는 인덱스 조건에 대해 **갭락(gap lock) / next-key 락**을 잡는다. 대상 행이 아직 없으면 "그 키가 들어갈 자리(gap)"에 갭락을 건다.
2. **갭락끼리는 서로 호환된다.** 그래서 T1과 T2가 같은 갭에 갭락을 동시에 보유할 수 있다.
3. 이어서 각자 **`INSERT`** 가 그 갭에 **insert-intention 락**을 잡으려 한다. 그런데 insert-intention 락은 **다른 트랜잭션이 보유한 갭락과 충돌**한다.
4. 결과: **T1의 INSERT는 T2의 갭락을, T2의 INSERT는 T1의 갭락을 기다린다 → 사이클 → 데드락.**

```
T1: DELETE (gap lock 획득) ─┐         ┌─ INSERT → T2의 gap lock 대기
                            ├ (호환) ─┤
T2: DELETE (gap lock 획득) ─┘         └─ INSERT → T1의 gap lock 대기
                                          ↑ 서로를 기다림 = 1213
```

이게 단 **2개만 동시여도** 거의 매번 터진 이유다. 그리고 동시 요청은 단일 사용자 로컬 앱에서도 흔하게 생긴다 — 바로 다음 절의 StrictMode 때문에.

> 참고: 지난 글의 1020은 "두 트랜잭션이 같은 행을 동시에 변경했다"는 **사실 자체**가 원인이라 DELETE를 없애도 소용없었다. 반면 1213은 **DELETE가 만드는 갭락**이 직접 원인이라, DELETE를 없애면 사라진다. 같은 트리거라도 층위가 다르다.

---

## 4. 수정 — `DELETE + INSERT` → 원자적 upsert

DELETE를 없애고 **`INSERT ... ON DUPLICATE KEY UPDATE`** 한 문장으로 바꾼다. Exposed 0.57.0의 `upsert`를 쓰면 PK를 충돌 키로 자동 사용한다. (`stock`의 PK는 `(ticker, market)`)

```kotlin
fun upsert(entity: StockEntity) {
    StockTable.upsert { row ->          // DELETE 없이 원자적 upsert
        row[ticker] = entity.ticker
        row[market] = entity.market
        row[name] = entity.name
        row[marketCapAmount] = entity.marketCapAmount
        row[marketCapCurrency] = entity.marketCapCurrency
        row[syncedAt] = entity.syncedAt
    }
}
```

생성 SQL:

```sql
INSERT INTO stock (ticker, market, name, market_cap_amount, market_cap_currency, synced_at)
VALUES (?, ?, ?, ?, ?, ?)
ON DUPLICATE KEY UPDATE
    name = VALUES(name),
    market_cap_amount = VALUES(market_cap_amount), ... ;
```

**DELETE가 사라졌으니 갭락도 사라진다.** 행이 이미 있으면 이건 그 한 행에 대한 X(배타) 락을 잡는 UPDATE일 뿐이라, 동시 트랜잭션은 그 행 락에서 **줄을 서서 직렬화**될 뿐 사이클이 생기지 않는다.

마침 같은 코드베이스의 시세 스냅샷 DAO(`QuoteSnapshotDao`)는 이미 `batchUpsert`를 쓰고 있었다. 즉 새 패턴을 도입한 게 아니라, **뒤처져 있던 두 DAO(`StockDao`, `FundamentalMetricsSnapshotDao`)를 기존 컨벤션에 맞춘** 것이다.

검증 — 같은 종목에 refresh를 동시에 두 번 쐈다.

```bash
curl -XPOST '.../api/stocks/A005930/refresh?market=DOMESTIC' &
curl -XPOST '.../api/stocks/A005930/refresh?market=DOMESTIC' &
wait
# req1: HTTP 200
# req2: HTTP 200
# 로그 deadlock/exception 건수: 0
```

수정 전엔 둘 중 하나가 반드시 500이었는데, 이제 둘 다 200이고 데드락 로그가 0건이다.

---

## 5. upsert면 N개 동시에도 안전한가 — 정직한 경계

"2개가 아니라 10개, 20개가 동시에 와도 안전하냐"는 질문에 대한 정확한 답.

**(A) 행이 이미 존재하는 동시 재갱신 — 안전 (N 무관).**
ON DUPLICATE KEY UPDATE가 기존 행 1개에 X락을 잡는 UPDATE가 되므로, N개 트랜잭션이 **같은 자원 1개를 같은 순서로** 기다린다. 사이클이 성립할 수 없어 데드락이 안 난다. 그냥 직렬화되어 느려질 뿐이다. 또 한 refresh가 `stock → quote → fundamentals → candle` 순으로 여러 테이블을 건드리지만, 모든 동시 요청이 **같은 코드 경로 = 같은 락 순서**라 테이블 간 교차 데드락도 없다.

**(B) 행이 아직 없는 "콜드스타트 떼거리" — 이론상 여지 남음.**
같은 새 키를 N개가 동시에 첫 INSERT 하면, InnoDB의 중복키 처리 과정에서 **S락 획득 → X락 승급** 경합으로 `ON DUPLICATE KEY UPDATE`도 데드락이 날 수 있다(잘 알려진 엣지). 다만 이 앱에서 종목 행은 보통 첫 sync 때 한 번 생기고, 그 뒤로는 (A)에 해당한다. delete+insert보다 압도적으로 좁은 창이다.

**(C) 진짜 보장을 원하면 — 데드락 재시도.**
InnoDB의 교과서적 지침은 *"데드락은 정상이다. 안 나게 만들려 하지 말고, 나면 재시도하라"* 이다. 어떤 단일 락 전략도 RR에서 모든 시나리오의 데드락을 0으로 증명할 수 없다. `1213 / SQLTransactionRollbackException`을 소수 횟수 재시도하는 래퍼가 정석이다.

```kotlin
// 개념 예시
fun <T> retryOnDeadlock(max: Int = 3, block: () -> T): T {
    repeat(max - 1) {
        try { return block() } catch (e: SQLTransactionRollbackException) { /* 재시도 */ }
    }
    return block()
}
```

정리하면 — **이번에 보고된 버그는 upsert로 완전히 해결**되고, "극단적 동시성에서도 절대 0"을 원하면 **재시도**를 얹는다.

---

## 6. 같은 트리거, 그리고 남은 것들

- **트리거는 지난 글과 동일하다 — React StrictMode 이중 마운트.** 개발 모드에서 `useEffect`가 두 번 실행되며 종목 상세의 refresh가 동시에 2벌 나간다. 지난 글의 1차 처방이었던 `useRef` 가드가 **새로 만든 `StockDetailPage`엔 적용돼 있지 않아** 그대로 재현됐다. 백엔드를 동시성-안전하게 만든 게 근본 처방이지만, 프론트 가드도 같이 넣으면 불필요한 중복 호출 자체가 준다.
- **외부 호출이 트랜잭션 안에 있다.** refresh 한 번이 11초 걸리는데, 키움/네이버/야후 HTTP 호출이 `@Transactional` 경계 안에서 일어나기 때문이다. 동시 요청이 많아지면 데드락보다 **커넥션 풀 고갈(HikariCP 기본 10)** 이 먼저 터진다. 외부 호출을 트랜잭션 밖으로 빼는 게 동시성·지연 둘 다에 더 큰 개선이다(별개 과제).

---

## 7. 정리 / 교훈

- **`Deadlock found when trying to get lock` (1213)** 은, snapshot isolation이 `OFF`인 REPEATABLE READ에서 `DELETE + INSERT` 패턴이 만드는 **갭락 + insert-intention 락 사이클**이 원인이다. 단 2개 동시여도 난다.
- **고치는 법은 DELETE를 없애는 것** — `INSERT ... ON DUPLICATE KEY UPDATE`(Exposed `upsert`). 행이 존재하면 단일 행 X락으로 직렬화되어 N개 동시에도 데드락이 없다.
- **지난 글과 정반대다.** 1020(snapshot isolation)에선 "upsert 무관"이었지만, 1213(갭락 데드락)에선 "upsert가 답"이다. **차이를 가른 건 코드가 아니라 `innodb_snapshot_isolation` 설정 한 줄.** 같은 증상 트리거라도 DB 설정이 실패 모드와 올바른 처방을 통째로 바꾼다.
- **한 문제를 닫으면 옆문이 열릴 수 있다.** snapshot isolation을 끄며 1020을 없앴더니 1213이 드러났다. 설정을 바꾼 뒤엔 "이전엔 가려져 있던 다른 경합"이 없는지 같이 봐야 한다.
- 절대적 보장이 필요하면 **데드락 재시도**가 정석이고, 더 근본적으론 **외부 호출을 트랜잭션 밖으로** 빼는 구조 개선이 남는다.

### 참고

- MariaDB Knowledge Base: `innodb_snapshot_isolation` (11.6.2+ 기본 활성화)
- MySQL/MariaDB 에러 `1213 ER_LOCK_DEADLOCK`, InnoDB의 gap lock / insert-intention lock
- 이전 글: *MariaDB 11.6+ snapshot isolation과 'Record has changed since last read' 디버깅*
