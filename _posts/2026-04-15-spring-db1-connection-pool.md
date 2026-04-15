---
title: "[스프링 DB 1편 - 2] 커넥션풀과 데이터소스 이해"
date: 2026-04-15 18:05:00 +0900
categories: [spring-db1]
tags: [spring, connection-pool, datasource, hikaricp, inflearn, 김영한]
toc: true
toc_sticky: true
---

## 개요

데이터베이스에 접근할 때마다 커넥션을 새로 만들면 성능 문제가 생긴다. 이를 해결하기 위해 커넥션풀이라는 개념이 등장했고, Spring은 `DataSource`라는 인터페이스로 커넥션 획득 방법 자체를 추상화한다. 이 글은 김영한님의 "스프링 DB 1편" 섹션 2에서 다룬 커넥션풀과 DataSource의 핵심 내용을 정리한 것이다.

---

## 1. 커넥션을 매번 생성하면 왜 느린가

애플리케이션에서 DB 커넥션을 획득하는 과정은 생각보다 무겁다.

1. 애플리케이션 로직이 DB 드라이버를 통해 커넥션을 요청한다.
2. DB 드라이버가 DB와 **TCP/IP 3-way 핸드셰이크**를 수행해 네트워크 연결을 맺는다.
3. 연결이 완료되면 **ID/PW 등 인증 정보**를 전달한다.
4. DB 내부에서 인증을 완료하고 **DB 세션**을 생성한다.
5. DB가 커넥션 생성 완료 응답을 보낸다.
6. DB 드라이버가 커넥션 객체를 생성해서 반환한다.

이 과정을 SQL 실행마다 반복하면 다음과 같은 비용이 누적된다.

- **네트워크 비용**: TCP 핸드셰이크 왕복
- **인증 비용**: 매 요청마다 사용자 검증
- **세션 생성 비용**: DB 내부 리소스 할당

결과적으로 응답 시간이 느려지고, 사용자 경험에 직접적인 영향을 준다.

---

## 2. 커넥션풀 개념

### 기본 원리

커넥션풀은 애플리케이션이 시작될 때 **미리 일정 수의 커넥션을 만들어 풀에 보관**하는 방식이다.

- **초기화**: 애플리케이션 구동 시점에 설정된 수만큼 커넥션을 미리 생성해서 풀에 저장한다.
- **사용**: 커넥션이 필요하면 풀에서 이미 생성된 커넥션을 꺼내 사용한다.
- **반환**: 사용이 끝나면 커넥션을 종료하지 않고 풀에 반환한다.

이 방식의 장점은 다음과 같다.

- 커넥션을 미리 만들어두므로 SQL 실행 시 **커넥션 생성 시간이 없다**.
- 커넥션 수를 제한하므로 DB에 **과도한 연결이 몰리는 것을 방지**한다.
- 커넥션을 재사용하므로 **리소스 효율**이 높다.

### HikariCP

- Spring Boot 2.0부터 **HikariCP**가 기본 커넥션풀이다.
- 속도, 안정성 면에서 다른 커넥션풀(DBCP2, Tomcat Pool 등)보다 우수하다.
- 별도 설정 없이 `spring-boot-starter-jdbc` 또는 `spring-boot-starter-data-jpa`를 추가하면 자동으로 사용된다.

기본 풀 사이즈는 10이며, `spring.datasource.hikari.maximum-pool-size`로 조정할 수 있다.

---

## 3. DataSource 인터페이스

### 커넥션 획득 방법의 추상화

커넥션을 얻는 방법은 여러 가지가 있다.

- `DriverManager`를 직접 사용
- HikariCP 커넥션풀 사용
- DBCP2 커넥션풀 사용

문제는 커넥션 획득 방법을 바꿀 때마다 애플리케이션 코드가 변경되어야 한다는 점이다. 예를 들어 `DriverManager`에서 HikariCP로 전환하면, 커넥션을 얻는 코드 전체를 수정해야 한다.

Java는 이 문제를 해결하기 위해 `javax.sql.DataSource`라는 인터페이스를 제공한다.

```java
public interface DataSource {
    Connection getConnection() throws SQLException;
}
```

핵심 메서드는 `getConnection()` 하나다. 어떤 방식으로 커넥션을 얻든, 애플리케이션은 `DataSource`에만 의존하면 된다.

### 설정과 사용의 분리

`DataSource`가 주는 또 하나의 이점은 **설정과 사용의 분리**다.

- **설정**: `DataSource`를 생성하는 시점에 URL, USERNAME, PASSWORD 등을 넘긴다.
- **사용**: `dataSource.getConnection()`만 호출한다. 설정값을 몰라도 된다.

`DriverManager`를 직접 쓰면 커넥션을 얻을 때마다 URL, USERNAME, PASSWORD를 파라미터로 넘겨야 한다. 반면 `DataSource`를 쓰면 객체 생성 시점에 한 번만 설정하고, 이후에는 `getConnection()`만 호출하면 된다.

이 분리 덕분에 **커넥션을 사용하는 코드는 설정에 대해 전혀 알 필요가 없다**. Repository 같은 사용 측 코드가 더 깔끔해진다.

---

## 4. DriverManagerDataSource vs HikariDataSource

### DriverManagerDataSource

Spring이 제공하는 `DataSource` 구현체로, 내부적으로 `DriverManager`를 사용한다.

```java
DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
Connection con = dataSource.getConnection();
```

- `DataSource` 인터페이스를 구현했으므로 코드에서는 `DataSource` 타입으로 사용 가능하다.
- 하지만 **호출할 때마다 새로운 커넥션을 생성**한다. 커넥션풀을 사용하지 않는다.

### HikariDataSource

HikariCP가 제공하는 `DataSource` 구현체다.

```java
HikariDataSource dataSource = new HikariDataSource();
dataSource.setJdbcUrl(URL);
dataSource.setUsername(USERNAME);
dataSource.setPassword(PASSWORD);
dataSource.setMaximumPoolSize(10);
Connection con = dataSource.getConnection();
```

- 내부에 커넥션풀을 가지고 있으며, `getConnection()` 호출 시 풀에서 커넥션을 꺼내 반환한다.
- 커넥션풀에 커넥션을 채우는 작업은 **별도 스레드**에서 진행된다. 애플리케이션 구동 로그에서 `HikariPool` 스레드가 커넥션을 추가하는 것을 확인할 수 있다.

### 전환이 쉬운 이유

`DataSource` 인터페이스 덕분에 구현체만 바꾸면 된다.

```java
// 변경 전
DataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);

// 변경 후
HikariDataSource dataSource = new HikariDataSource();
dataSource.setJdbcUrl(URL);
dataSource.setUsername(USERNAME);
dataSource.setPassword(PASSWORD);
```

Repository를 포함한 나머지 코드는 `DataSource`에만 의존하므로 **변경할 필요가 없다**. OCP(개방-폐쇄 원칙)를 잘 지킨 설계다.

---

## 5. MemberRepositoryV1 변화

### 기존 방식의 문제점

`DriverManager`를 직접 사용하는 기존 Repository는 다음과 같은 문제가 있었다.

- 커넥션 획득 시 URL, USERNAME, PASSWORD를 매번 전달해야 한다.
- `close()` 처리가 번거롭다. Connection, Statement, ResultSet을 각각 `try-catch`로 닫아야 한다.
- 커넥션 획득 방법을 바꾸려면 Repository 코드를 수정해야 한다.

### DataSource 주입으로 개선

```java
public class MemberRepositoryV1 {

    private final DataSource dataSource;

    public MemberRepositoryV1(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    private Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
}
```

- `DataSource`를 외부에서 주입받으므로 커넥션 획득 방법에 의존하지 않는다.
- `DriverManagerDataSource`든 `HikariDataSource`든 주입하는 쪽에서 결정한다.

### JdbcUtils로 리소스 정리 간소화

기존에는 `close()` 처리를 직접 작성해야 했다.

```java
// 기존: 직접 close 처리
if (rs != null) { try { rs.close(); } catch (SQLException e) { ... } }
if (stmt != null) { try { stmt.close(); } catch (SQLException e) { ... } }
if (con != null) { try { con.close(); } catch (SQLException e) { ... } }
```

Spring이 제공하는 `JdbcUtils`를 사용하면 한 줄로 줄일 수 있다.

```java
private void close(Connection con, Statement stmt, ResultSet rs) {
    JdbcUtils.closeResultSet(rs);
    JdbcUtils.closeStatement(stmt);
    JdbcUtils.closeConnection(con);
}
```

- `null` 체크와 예외 처리를 내부에서 알아서 해준다.
- 코드가 훨씬 깔끔해지고 실수할 여지가 줄어든다.

---

## 정리

| 항목 | 핵심 내용 |
|------|----------|
| 커넥션 생성 비용 | TCP 핸드셰이크 + 인증 + 세션 생성으로 매번 생성하면 느리다 |
| 커넥션풀 | 미리 만들어두고 빌려쓰고 반환한다. HikariCP가 Spring Boot 기본이다 |
| DataSource | 커넥션 획득 방법을 추상화한 인터페이스. 설정과 사용을 분리한다 |
| 구현체 전환 | DataSource 인터페이스 덕분에 구현체만 바꾸면 코드 변경 없이 전환 가능하다 |
| Repository 개선 | DataSource 주입 + JdbcUtils로 코드가 간결해진다 |

커넥션풀과 DataSource는 이후 트랜잭션 처리의 기반이 된다. 트랜잭션 매니저가 DataSource를 통해 커넥션을 획득하고, 같은 커넥션을 트랜잭션 동기화 매니저에 보관하는 흐름으로 이어지므로, 이 개념을 확실히 잡아두는 것이 중요하다.
