---
title: "[스프링 DB 1편 - 1] JDBC 이해"
date: 2026-04-15 18:06:00 +0900
categories: [spring-db1]
tags: [spring, jdbc, database, inflearn, 김영한]
toc: true
toc_sticky: true
---

## 개요

김영한님의 "스프링 DB 1편" 섹션 1에서 다루는 JDBC의 핵심 내용을 정리한다. 왜 JDBC가 필요한지, 어떻게 동작하는지, 그리고 실제 CRUD를 구현할 때 주의해야 할 포인트를 중심으로 요약했다.

## 1. JDBC가 등장한 이유

애플리케이션에서 DB에 접근하려면 다음 3가지를 해야 한다.

- 커넥션 연결
- SQL 전달
- 결과 응답

문제는 DB마다 이 방법이 전부 다르다는 것이다. MySQL에서 Oracle로 바꾸면 데이터 접근 코드를 전부 수정해야 한다. DB를 바꿀 때마다 새로운 사용법을 학습해야 하는 것도 부담이다.

이 문제를 해결하기 위해 자바는 **JDBC(Java Database Connectivity)**라는 표준 인터페이스를 제공한다.

### JDBC 표준 인터페이스 3가지

| 인터페이스 | 역할 |
|-----------|------|
| `java.sql.Connection` | DB 연결 |
| `java.sql.Statement` | SQL 전달 |
| `java.sql.ResultSet` | 결과 조회 |

각 DB 벤더(MySQL, Oracle, PostgreSQL 등)가 이 인터페이스의 구현체를 제공하는데, 이것이 **JDBC 드라이버**다.

결과적으로 개발자는 JDBC 표준 인터페이스만 사용하면 되고, DB를 변경할 때는 드라이버만 교체하면 된다. 데이터 접근 코드는 변경할 필요가 없다.

> 물론 ANSI SQL이 아닌 DB 고유 문법을 사용했다면 SQL 자체는 수정이 필요하다. 하지만 접근 코드의 변경은 막을 수 있다.

## 2. JDBC 드라이버와 DriverManager

### JDBC 드라이버

JDBC 드라이버는 각 DB 벤더가 JDBC 인터페이스를 구현한 라이브러리다. 예를 들어 MySQL을 사용한다면 `mysql-connector-java`를 의존성에 추가하면 된다.

### DriverManager 동작 방식

`DriverManager`는 등록된 JDBC 드라이버 목록을 관리하고, 커넥션 요청이 오면 적절한 드라이버를 찾아 커넥션을 반환한다.

```
DriverManager.getConnection(URL, USER, PASSWORD)
    ↓
등록된 드라이버 목록을 순회
    ↓
각 드라이버에게 "이 URL 처리할 수 있어?" 확인
    ↓
처리 가능한 드라이버가 커넥션 반환
```

동작 순서를 좀 더 풀어보면:

1. 애플리케이션이 `DriverManager.getConnection()`을 호출한다.
2. `DriverManager`는 등록된 드라이버들에게 순서대로 커넥션 요청을 위임한다.
3. URL 형식을 보고 자기가 처리할 수 있는 드라이버가 실제 DB 커넥션을 생성해서 반환한다.
4. 처리할 수 없는 드라이버는 다음으로 넘어간다.

예를 들어 URL이 `jdbc:h2:...`이면 H2 드라이버가 응답하고, `jdbc:mysql:...`이면 MySQL 드라이버가 응답한다.

## 3. SQL Mapper vs ORM

JDBC를 직접 사용하는 것은 번거롭다. 그래서 편리하게 사용할 수 있는 기술들이 등장했는데, 크게 두 가지로 나뉜다.

### SQL Mapper

- **JdbcTemplate**, **MyBatis**
- SQL 응답 결과를 객체로 매핑해준다
- JDBC의 반복 코드를 줄여준다
- SQL은 개발자가 직접 작성해야 한다

### ORM

- **JPA**, **Hibernate**
- 객체를 관계형 DB 테이블과 매핑해준다
- SQL을 직접 작성하지 않아도 ORM 기술이 대신 생성해준다
- 각 DB마다 다른 SQL을 알아서 처리해준다

핵심 포인트는 **SQL Mapper든 ORM이든 결국 내부적으로는 JDBC를 사용한다**는 것이다. JDBC가 DB 접근 기술의 가장 아래에 있는 기반 기술이기 때문에, JDBC의 동작 원리를 이해해두면 상위 기술에서 문제가 생겼을 때 원인을 파악하는 데 도움이 된다.

```
JdbcTemplate / MyBatis / JPA
        ↓
      JDBC
        ↓
    DB 드라이버
        ↓
       DB
```

## 4. 커넥션 획득 흐름

실제 코드에서 커넥션을 획득하는 과정을 정리한다.

```java
Connection connection = DriverManager.getConnection(URL, USERNAME, PASSWORD);
```

이 한 줄이 내부적으로 수행하는 과정:

1. `DriverManager`가 라이브러리에 등록된 드라이버 목록을 확인한다.
2. 각 드라이버에게 URL을 전달하며 커넥션 획득을 시도한다.
3. URL에 맞는 드라이버가 TCP/IP로 DB와 커넥션을 연결한다.
4. DB와의 연결이 완료되면 드라이버는 `Connection` 구현체를 생성하여 반환한다.

커넥션을 매번 새로 생성하는 것은 비용이 크기 때문에, 실무에서는 **커넥션 풀**(HikariCP 등)을 사용한다. 이 내용은 이후 섹션에서 다룬다.

## 5. CRUD 구현 핵심 포인트

### PreparedStatement를 사용해야 하는 이유

SQL을 문자열 연결(`+`)로 작성하면 SQL 인젝션 공격에 취약하다.

```java
// 위험 - SQL 인젝션 가능
String sql = "select * from member where member_id = '" + memberId + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);
```

`PreparedStatement`는 파라미터를 바인딩 방식으로 처리하기 때문에, 파라미터 값이 SQL 구문으로 해석되지 않는다.

```java
// 안전 - 파라미터 바인딩
String sql = "select * from member where member_id = ?";
PreparedStatement pstmt = conn.prepareStatement(sql);
pstmt.setString(1, memberId);
ResultSet rs = pstmt.executeQuery();
```

> `Statement`가 아닌 `PreparedStatement`를 사용하는 것은 선택이 아니라 필수다.

### executeUpdate vs executeQuery

| 메서드 | 용도 | 반환값 |
|--------|------|--------|
| `executeUpdate()` | INSERT, UPDATE, DELETE | 영향받은 row 수 (`int`) |
| `executeQuery()` | SELECT | `ResultSet` |

### 리소스 정리 순서

JDBC에서 사용한 리소스는 반드시 역순으로 정리해야 한다.

```
획득 순서: Connection → PreparedStatement → ResultSet
정리 순서: ResultSet → PreparedStatement → Connection
```

리소스 정리를 하지 않으면 커넥션이 반환되지 않아 **커넥션 누수**가 발생한다. 커넥션 풀을 사용하는 경우, 커넥션이 풀로 돌아가지 못해 결국 풀이 고갈되고 애플리케이션이 멈출 수 있다.

리소스 정리는 반드시 `finally` 블록 또는 `try-with-resources`에서 수행해야 한다. 각 리소스를 닫을 때 예외가 발생할 수 있으므로 개별적으로 `try-catch`를 감싸야 한다.

```java
finally {
    if (rs != null) {
        try { rs.close(); } catch (SQLException e) { log.info("error", e); }
    }
    if (pstmt != null) {
        try { pstmt.close(); } catch (SQLException e) { log.info("error", e); }
    }
    if (conn != null) {
        try { conn.close(); } catch (SQLException e) { log.info("error", e); }
    }
}
```

`rs.close()`에서 예외가 터져도 `pstmt`와 `conn`은 반드시 닫아야 하기 때문에 이렇게 분리하는 것이다.

> 실무에서 JdbcTemplate 등을 사용하면 이런 반복적인 리소스 정리를 자동으로 처리해준다. 하지만 원리를 알아야 문제가 생겼을 때 대응할 수 있다.

## 정리

- JDBC는 DB 접근의 **표준 인터페이스**다. 상위 기술(JdbcTemplate, MyBatis, JPA)이 모두 JDBC 위에서 동작한다.
- `DriverManager`는 등록된 드라이버를 순회하며 URL에 맞는 드라이버를 찾아 커넥션을 반환한다.
- SQL 인젝션 방지를 위해 반드시 `PreparedStatement`의 파라미터 바인딩을 사용해야 한다.
- 리소스 정리는 **획득의 역순**(ResultSet -> Statement -> Connection)으로, `finally` 블록에서 개별 `try-catch`로 수행한다.
- 리소스를 정리하지 않으면 커넥션 누수가 발생하여 애플리케이션이 멈출 수 있다.
