---
title: "[스프링 DB 1편 - 4] 스프링과 문제 해결 — 트랜잭션"
date: 2026-04-15 18:03:00 +0900
categories: [spring-db1]
tags: [spring, transaction, transactional, aop, proxy, inflearn, 김영한]
toc: true
toc_sticky: true
---

## 개요

김영한님의 "스프링 DB 1편" 섹션 4에서는 애플리케이션에서 트랜잭션을 다룰 때 발생하는 구조적 문제 3가지를 정의하고, Spring이 이를 어떻게 해결하는지 단계별로 보여준다. 트랜잭션 매니저 직접 사용 -> TransactionTemplate -> `@Transactional` AOP 순으로 서비스 코드가 개선되는 흐름을 따라가며, 최종적으로 서비스 계층에 순수 비즈니스 로직만 남기는 것이 목표다. 이 글은 해당 섹션의 핵심 내용을 정리한 것이다.

---

## 1. 기존 코드의 문제 3가지

### 1-1. 트랜잭션 코드 침투

서비스 계층에 `connection.setAutoCommit(false)`, `connection.commit()`, `connection.rollback()` 같은 트랜잭션 처리 코드가 비즈니스 로직과 섞여 있다. 서비스 로직의 핵심은 계좌 이체 같은 도메인 로직인데, 트랜잭션 관리 코드가 절반 이상을 차지하게 된다.

- 비즈니스 로직 변경 시 트랜잭션 코드를 건드릴 위험
- 서비스 계층이 `java.sql.Connection`에 직접 의존

### 1-2. SQLException 누수

서비스 계층에서 `SQLException`(JDBC 기술에 종속된 예외)을 `throws`하거나 `catch`하고 있다. 만약 JDBC에서 JPA로 기술을 변경하면 서비스 코드의 예외 처리도 전부 수정해야 한다.

- 서비스가 특정 데이터 접근 기술에 종속
- 기술 변경 시 서비스 코드 수정 불가피

### 1-3. JDBC 반복 코드

리포지토리에서 `getConnection()`, `PreparedStatement` 생성, 파라미터 바인딩, 결과 매핑, `close()` 등 거의 동일한 코드를 매 메서드마다 반복한다.

- 이 문제는 섹션 4보다는 이후 JdbcTemplate 등으로 해결

---

## 2. 트랜잭션 추상화 — PlatformTransactionManager

### 2-1. 왜 추상화가 필요한가

JDBC는 `Connection.commit()`, JPA는 `EntityTransaction.commit()`처럼 기술마다 트랜잭션 처리 API가 다르다. 서비스 코드가 특정 기술의 API에 직접 의존하면 기술 변경 시 서비스 코드를 전부 수정해야 한다.

### 2-2. PlatformTransactionManager 인터페이스

Spring은 트랜잭션을 추상화한 `PlatformTransactionManager` 인터페이스를 제공한다.

```java
public interface PlatformTransactionManager extends TransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition);
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

- `getTransaction()` : 트랜잭션 시작 (기존 트랜잭션이 있으면 참여)
- `commit()` : 커밋
- `rollback()` : 롤백

### 2-3. 구현체

| 구현체 | 대상 기술 |
|-------|----------|
| `DataSourceTransactionManager` | JDBC, MyBatis |
| `JpaTransactionManager` | JPA |
| `HibernateTransactionManager` | Hibernate |

서비스 코드는 `PlatformTransactionManager` 인터페이스에만 의존하고, 실제 구현체는 DI로 주입받는다. JDBC에서 JPA로 바꿀 때 서비스 코드 변경 없이 구현체만 교체하면 된다.

---

## 3. 트랜잭션 동기화

### 3-1. 문제 상황

하나의 트랜잭션 안에서 여러 리포지토리 메서드를 호출할 때, 모두 **같은 커넥션**을 사용해야 한다. 그렇지 않으면 각 메서드가 별도 커넥션을 잡아 트랜잭션이 분리된다.

### 3-2. TransactionSynchronizationManager

Spring은 `TransactionSynchronizationManager`를 통해 트랜잭션 동기화를 처리한다. 내부적으로 **ThreadLocal**을 사용하여 쓰레드별로 커넥션을 보관한다.

동작 흐름:

1. 트랜잭션 매니저가 `DataSource`에서 커넥션을 획득
2. `TransactionSynchronizationManager`에 커넥션 보관 (ThreadLocal)
3. 리포지토리에서 `DataSourceUtils.getConnection(dataSource)`를 호출하면 보관된 커넥션을 반환
4. 트랜잭션이 끝나면 동기화 매니저에서 커넥션을 꺼내 커밋/롤백 후 정리

### 3-3. DataSourceUtils

```java
// 동기화된 커넥션 획득 (없으면 새로 생성)
Connection con = DataSourceUtils.getConnection(dataSource);

// 커넥션 반환 (트랜잭션 동기화 중이면 닫지 않음)
DataSourceUtils.releaseConnection(con, dataSource);
```

- 리포지토리에서 `dataSource.getConnection()` 대신 `DataSourceUtils.getConnection()`을 사용해야 트랜잭션 동기화가 동작한다.
- 파라미터로 커넥션을 넘기는 방식(V2)의 문제를 깔끔하게 해결한다.

---

## 4. 서비스 코드의 진화

### 4-1. V3_1 — 트랜잭션 매니저 직접 사용

```java
public void accountTransfer(String fromId, String toId, int money) {
    TransactionStatus status = transactionManager.getTransaction(
        new DefaultTransactionDefinition());
    try {
        bizLogic(fromId, toId, money);
        transactionManager.commit(status);
    } catch (Exception e) {
        transactionManager.rollback(status);
        throw new IllegalStateException(e);
    }
}
```

**개선된 점:**
- 서비스가 `Connection`에 직접 의존하지 않음
- 트랜잭션 추상화 인터페이스 사용 (기술 변경에 유연)
- 커넥션을 파라미터로 넘기지 않아도 됨 (동기화 매니저)

**남은 문제:**
- try/catch/commit/rollback 패턴이 매 메서드마다 반복

### 4-2. V3_2 — TransactionTemplate

```java
private final TransactionTemplate txTemplate;

public void accountTransfer(String fromId, String toId, int money) {
    txTemplate.executeWithoutResult((status) -> {
        bizLogic(fromId, toId, money);
    });
}
```

`TransactionTemplate`은 템플릿 콜백 패턴을 적용하여 try/catch 반복을 제거한다.

- 성공하면 자동 커밋
- 언체크 예외 발생 시 자동 롤백
- `execute()` (반환값 있음) / `executeWithoutResult()` (반환값 없음)

**남은 문제:**
- 여전히 트랜잭션 처리 코드(`txTemplate.executeWithoutResult(...)`)가 서비스에 존재
- 서비스에 비즈니스 로직만 남기고 싶다

### 4-3. V3_3 — @Transactional AOP

```java
@Transactional
public void accountTransfer(String fromId, String toId, int money) {
    bizLogic(fromId, toId, money);
}
```

**최종 형태.** 서비스 메서드에 `@Transactional`만 선언하면 트랜잭션 관련 코드가 완전히 사라진다.

#### 동작 원리

1. Spring이 `@Transactional`이 붙은 클래스에 대해 **프록시 객체**를 생성
2. 컨트롤러가 서비스를 호출하면, 실제로는 프록시가 호출됨
3. 프록시가 트랜잭션을 시작하고 실제 서비스 메서드를 호출
4. 정상 완료 시 커밋, 런타임 예외 발생 시 롤백

```
Client -> Proxy (트랜잭션 시작)
              -> 실제 Service (비즈니스 로직)
         Proxy (커밋 or 롤백)
```

#### 핵심 포인트

- 서비스 계층에 순수 비즈니스 로직만 남는다
- 트랜잭션 처리는 프록시(AOP)가 전담
- 클래스 레벨에 선언하면 모든 public 메서드에 적용
- 메서드 레벨에 선언하면 해당 메서드만 적용
- 테스트에서 `@Transactional`을 사용하면 테스트 완료 후 자동 롤백

---

## 5. Spring Boot의 자동 리소스 등록

### 5-1. DataSource 자동 등록

Spring Boot는 `application.properties`(또는 `yml`)에 설정된 정보를 기반으로 `DataSource`를 자동으로 빈 등록한다.

```properties
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
spring.datasource.password=
```

- 기본 구현체: `HikariDataSource` (커넥션 풀)
- 개발자가 직접 `DataSource` 빈을 등록하면 자동 등록은 동작하지 않음

### 5-2. TransactionManager 자동 등록

Spring Boot는 사용하는 데이터 접근 기술을 감지하여 적절한 트랜잭션 매니저를 자동 등록한다.

- JDBC 사용 시 -> `DataSourceTransactionManager`
- JPA 사용 시 -> `JpaTransactionManager`
- 빈 이름: `transactionManager`

개발자가 직접 트랜잭션 매니저 빈을 등록하면 자동 등록은 비활성화된다.

---

## 정리

| 버전 | 방식 | 특징 |
|-----|------|------|
| V1 | `Connection` 직접 사용 | 서비스가 JDBC에 완전 종속 |
| V2 | 커넥션 파라미터 전달 | 같은 커넥션 공유는 되지만 코드가 지저분 |
| V3_1 | `PlatformTransactionManager` | 트랜잭션 추상화 + 동기화 적용 |
| V3_2 | `TransactionTemplate` | try/catch 반복 제거 (템플릿 콜백) |
| V3_3 | `@Transactional` | AOP 프록시로 서비스에서 트랜잭션 코드 완전 제거 |

결국 Spring이 해결하는 핵심은 **관심사의 분리**다. 비즈니스 로직과 트랜잭션 처리를 분리하여, 서비스 계층이 도메인 로직에만 집중할 수 있게 만든다. `@Transactional` 하나로 이 모든 것이 가능한 이유는 트랜잭션 추상화, 트랜잭션 동기화, AOP 프록시라는 세 가지 메커니즘이 뒷받침하기 때문이다.
