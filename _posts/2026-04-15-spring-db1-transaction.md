---
title: "[스프링 DB 1편 - 3] 트랜잭션 이해"
date: 2026-04-15
categories: [spring-db1]
tags: [spring, transaction, acid, db-lock, inflearn, 김영한]
toc: true
toc_sticky: true
---

## 개요

김영한님의 "스프링 DB 1편 - 데이터 접근 핵심 원리" 섹션 3에서 다루는 트랜잭션의 핵심 개념을 정리한다. 트랜잭션이 왜 필요한지, DB 내부에서 커넥션과 세션이 어떻게 동작하는지, 락은 어떤 상황에서 걸리는지, 그리고 애플리케이션 레벨에서 트랜잭션을 적용할 때 어떤 문제가 발생하는지까지 순서대로 살펴본다.

---

## 1. 트랜잭션과 ACID

트랜잭션은 하나의 작업 단위다. 계좌 이체를 예로 들면, A 계좌에서 출금하고 B 계좌에 입금하는 두 작업은 반드시 함께 성공하거나 함께 실패해야 한다. 이를 보장하기 위해 DB는 ACID 속성을 제공한다.

### ACID 속성

| 속성 | 설명 |
|------|------|
| **Atomicity (원자성)** | 트랜잭션 내 작업은 모두 성공하거나 모두 실패한다. 중간 상태는 없다. |
| **Consistency (일관성)** | 트랜잭션 완료 후에도 DB의 무결성 제약조건이 유지된다. |
| **Isolation (격리성)** | 동시에 실행되는 트랜잭션들이 서로 영향을 주지 않는다. |
| **Durability (지속성)** | 커밋된 데이터는 시스템 장애가 발생해도 보존된다. |

### 트랜잭션 격리 수준

격리성은 동시성과 트레이드오프 관계에 있다. 격리 수준이 높을수록 데이터 정합성은 좋지만 성능은 떨어진다.

| 격리 수준 | Dirty Read | Non-Repeatable Read | Phantom Read |
|-----------|:----------:|:-------------------:|:------------:|
| READ UNCOMMITTED | O | O | O |
| READ COMMITTED | X | O | O |
| REPEATABLE READ | X | X | O |
| SERIALIZABLE | X | X | X |

- **READ UNCOMMITTED**: 커밋되지 않은 데이터도 읽을 수 있다. 실무에서 거의 사용하지 않는다.
- **READ COMMITTED**: 커밋된 데이터만 읽는다. 대부분의 DB 기본값이다.
- **REPEATABLE READ**: 같은 트랜잭션 내에서 같은 데이터를 반복 조회해도 결과가 동일하다.
- **SERIALIZABLE**: 가장 엄격하다. 트랜잭션을 순차 실행하는 것처럼 동작한다.

일반적으로 **READ COMMITTED**를 사용하고, 필요한 경우에만 격리 수준을 올린다.

---

## 2. DB 커넥션과 세션

### 커넥션-세션 구조

```
클라이언트 → [커넥션] → DB 서버
                         ↓
                       [세션 생성]
                         ↓
                    SQL 실행, 트랜잭션 관리
```

- 클라이언트가 DB에 커넥션을 맺으면, DB 서버는 내부에 **세션**을 생성한다.
- 모든 SQL은 이 세션을 통해 실행된다.
- 트랜잭션의 시작과 종료(커밋/롤백)도 세션 단위로 관리된다.
- 커넥션 풀에서 커넥션 10개를 생성하면, DB 서버에도 세션 10개가 만들어진다.

핵심은 **커넥션을 통해 세션이 만들어지고, 세션이 트랜잭션을 관리한다**는 것이다.

---

## 3. 자동 커밋 vs 수동 커밋

### 자동 커밋 (Auto Commit)

```sql
set autocommit true;  -- 기본값
insert into member values ('A', 10000);  -- 즉시 커밋
insert into member values ('B', 10000);  -- 즉시 커밋
```

- 각 SQL이 실행될 때마다 자동으로 커밋된다.
- 편리하지만, 여러 SQL을 하나의 트랜잭션으로 묶을 수 없다.
- 첫 번째 INSERT는 성공하고 두 번째 INSERT가 실패하면 데이터 정합성이 깨진다.

### 수동 커밋 (Manual Commit)

```sql
set autocommit false;  -- 트랜잭션 시작
insert into member values ('A', 10000);
insert into member values ('B', 10000);
commit;  -- 두 작업을 한 번에 커밋
```

- `set autocommit false`를 실행하면 트랜잭션이 시작된다.
- 이후 `commit` 또는 `rollback`으로 트랜잭션을 종료한다.
- 중간에 문제가 생기면 `rollback`으로 모든 변경을 되돌릴 수 있다.

**트랜잭션을 사용한다는 것은 자동 커밋을 끄고 수동 커밋 모드로 전환하는 것**이다.

---

## 4. DB 락 (Lock)

트랜잭션의 격리성을 보장하려면, 같은 데이터를 동시에 수정하지 못하도록 제어해야 한다. DB는 이를 **락(Lock)** 메커니즘으로 해결한다.

### 변경 시 락 동작

```
세션1: set autocommit false;
세션1: update member set money=500 where member_id='A';  -- 락 획득

세션2: set autocommit false;
세션2: update member set money=1000 where member_id='A';  -- 락 대기 (블로킹)

세션1: commit;  -- 락 반납

세션2: -- 락 획득, update 실행
세션2: commit;
```

- 데이터를 변경하려면 해당 row의 락을 먼저 획득해야 한다.
- 다른 세션이 이미 락을 보유 중이면, 락이 반납될 때까지 대기한다.
- 락 대기 시간이 초과되면 타임아웃 오류가 발생한다.

### 조회 시 락 — SELECT FOR UPDATE

일반적인 SELECT는 락을 사용하지 않는다. 하지만 조회한 데이터를 기반으로 변경 작업을 수행해야 할 때, 다른 세션이 중간에 값을 바꾸면 문제가 된다.

```sql
select * from member where member_id='A' for update;
```

- `SELECT FOR UPDATE`는 조회 시점에 해당 row의 락을 획득한다.
- 트랜잭션이 끝날 때까지 다른 세션은 해당 데이터를 변경할 수 없다.
- 금액 조회 후 업데이트하는 계좌 이체 같은 시나리오에서 유용하다.

---

## 5. 애플리케이션에서 트랜잭션 적용

### 트랜잭션 없는 코드 — MemberServiceV1

트랜잭션 없이 계좌 이체를 구현하면 다음과 같은 구조가 된다.

```
MemberServiceV1.accountTransfer()
    ├── memberRepository.findById(fromId)  -- 커넥션1
    ├── memberRepository.update(fromId, money - transferAmount)  -- 커넥션2
    └── memberRepository.update(toId, money + transferAmount)  -- 커넥션3 (예외 발생 시?)
```

**문제점:**

- 각 리포지토리 호출이 서로 다른 커넥션(세션)을 사용할 수 있다.
- 커넥션이 다르면 트랜잭션도 다르다.
- 출금은 성공했는데 입금에서 예외가 발생하면, 출금만 반영되어 데이터 정합성이 깨진다.

### 트랜잭션 적용 — MemberServiceV2

하나의 트랜잭션으로 묶으려면, **같은 커넥션을 사용**해야 한다. 이를 위해 서비스에서 커넥션을 직접 관리하고, 리포지토리에 파라미터로 전달한다.

```java
public void accountTransfer(String fromId, String toId, int money) throws SQLException {
    Connection con = dataSource.getConnection();
    try {
        con.setAutoCommit(false);  // 트랜잭션 시작

        // 비즈니스 로직 수행 — 같은 커넥션 사용
        Member fromMember = memberRepository.findById(con, fromId);
        Member toMember = memberRepository.findById(con, toId);
        memberRepository.update(con, fromId, fromMember.getMoney() - money);
        memberRepository.update(con, toId, toMember.getMoney() + money);

        con.commit();  // 성공 시 커밋
    } catch (Exception e) {
        con.rollback();  // 실패 시 롤백
        throw new IllegalStateException(e);
    } finally {
        release(con);
    }
}
```

**해결된 점:**

- 같은 커넥션을 사용하므로 같은 세션, 같은 트랜잭션에서 실행된다.
- 중간에 예외가 발생하면 전체 롤백된다.
- 데이터 정합성이 보장된다.

**남아 있는 문제점:**

- 서비스 계층에 `Connection`, `SQLException` 등 JDBC 관련 코드가 침투한다.
- 리포지토리 메서드마다 `Connection` 파라미터를 추가해야 한다. 커넥션을 사용하는 메서드와 사용하지 않는 메서드가 공존하게 되어 코드가 지저분해진다.
- 서비스 계층은 비즈니스 로직에 집중해야 하는데, 트랜잭션 관리 코드가 섞여 있다.
- JDBC에서 JPA로 기술을 변경하면 서비스 코드도 함께 수정해야 한다.

이 문제들은 이후 섹션에서 **트랜잭션 추상화**와 **트랜잭션 매니저**를 통해 해결된다.

---

## 정리

| 주제 | 핵심 내용 |
|------|----------|
| ACID | 트랜잭션이 보장해야 할 4가지 속성. 격리성은 격리 수준으로 조절한다. |
| 커넥션과 세션 | 커넥션마다 세션이 생성되고, 세션이 트랜잭션을 관리한다. |
| 자동/수동 커밋 | `autocommit false`가 트랜잭션의 시작이다. |
| DB 락 | 동시 수정을 방지하기 위해 row 단위 락을 사용한다. |
| V1 vs V2 | 같은 커넥션을 공유해야 트랜잭션이 동작한다. 하지만 직접 관리하면 JDBC 코드가 서비스에 침투한다. |

트랜잭션의 개념과 DB 내부 동작 원리를 이해하는 것이 먼저다. 그 위에서 Spring이 제공하는 추상화가 어떤 문제를 해결하는지 파악하면, `@Transactional` 하나를 붙이더라도 내부에서 무슨 일이 일어나는지 알 수 있다.
