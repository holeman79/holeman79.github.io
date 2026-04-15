---
title: "[스프링 DB 1편] 스프링과 문제 해결 — 예외 처리, 반복"
date: 2026-04-15
categories: [spring-db1]
tags: [spring, exception, jdbctemplate, datasource, inflearn, 김영한]
toc: true
toc_sticky: true
---

## 개요

김영한님의 "스프링 DB 1편" 섹션 6은 지금까지 만들어 온 데이터 접근 계층의 **남은 문제 두 가지**를 해결한다.

1. **예외 누수** — 리포지토리에서 던지는 `SQLException`이 서비스 계층까지 올라와 JDBC 기술에 의존하게 되는 문제
2. **반복 코드** — 커넥션 획득, `PreparedStatement` 생성, 결과 매핑, 자원 정리 등의 보일러플레이트

이 글에서는 체크 예외를 런타임 예외로 전환하고, Spring 예외 추상화를 적용하고, 마지막으로 `JdbcTemplate`으로 반복을 제거하는 과정을 정리한다.

---

## 1. 체크 예외와 인터페이스의 문제

### 인터페이스에 체크 예외가 포함되면

리포지토리를 인터페이스로 추상화하려 해도, 구현 메서드가 `throws SQLException`을 선언하면 인터페이스에도 같은 선언이 필요하다.

```java
public interface MemberRepositoryEx {
    Member save(Member member) throws SQLException;
    Member findById(String memberId) throws SQLException;
    void update(String memberId, int money) throws SQLException;
    void delete(String memberId) throws SQLException;
}
```

- 인터페이스 자체가 JDBC에 종속된다.
- 나중에 JPA로 구현을 바꾸면 `throws JPAException`으로 변경해야 하고, 인터페이스와 서비스 코드까지 전부 수정이 필요하다.
- 결국 **인터페이스를 도입한 의미가 사라진다.**

### 순수 인터페이스 — MemberRepository

체크 예외를 제거하면 깨끗한 인터페이스가 만들어진다.

```java
public interface MemberRepository {
    Member save(Member member);
    Member findById(String memberId);
    void update(String memberId, int money);
    void delete(String memberId);
}
```

이 인터페이스는 특정 기술에 의존하지 않으므로 구현체를 자유롭게 교체할 수 있다. 문제는 구현체에서 `SQLException`을 어떻게 처리하느냐다.

---

## 2. 런타임 예외 전환 — MemberRepositoryV4_1

### 커스텀 런타임 예외 정의

```java
public class MyDbException extends RuntimeException {
    public MyDbException(Throwable cause) {
        super(cause);
    }
}
```

### 구현체에서 전환

`MemberRepositoryV4_1`은 `MemberRepository` 인터페이스를 구현하면서, 내부에서 `SQLException`을 잡아 `MyDbException`으로 감싼다.

```java
@Override
public Member save(Member member) {
    try {
        // JDBC 로직
    } catch (SQLException e) {
        throw new MyDbException(e);  // 런타임 예외로 전환
    }
}
```

핵심 포인트:

- 메서드 시그니처에서 `throws SQLException`이 제거된다.
- 인터페이스의 순수성이 유지된다.
- 원본 예외(`SQLException`)를 `cause`로 반드시 포함해야 한다. 빠뜨리면 디버깅이 불가능해진다.

---

## 3. MemberServiceV4 — 순수 비즈니스 로직

서비스 계층이 `MemberRepository` 인터페이스에만 의존하게 되면서, 기술 관련 코드가 모두 사라진다.

```java
@Service
public class MemberServiceV4 {

    private final MemberRepository memberRepository;

    @Transactional
    public void accountTransfer(String fromId, String toId, int money) {
        Member fromMember = memberRepository.findById(fromId);
        Member toMember = memberRepository.findById(toId);

        memberRepository.update(fromId, fromMember.getMoney() - money);
        memberRepository.update(toId, toMember.getMoney() + money);
    }
}
```

- `SQLException`, `DataSource`, `Connection` 등 기술 의존이 전혀 없다.
- `@Transactional`이 트랜잭션 처리를 담당한다.
- 순수 비즈니스 로직만 남아 **유지보수성과 테스트 용이성이 극대화**된다.

---

## 4. 데이터 접근 예외 직접 만들기

### 문제 — 예외를 구분할 수 없다

`MyDbException` 하나로는 "키 중복"인지 "커넥션 실패"인지 구분이 안 된다. 특정 예외(예: 키 중복)에 대해 복구 로직을 작성하려면 예외를 세분화해야 한다.

### MyDuplicateKeyException

```java
public class MyDuplicateKeyException extends MyDbException {
    public MyDuplicateKeyException(Throwable cause) {
        super(cause);
    }
}
```

리포지토리에서 에러코드를 확인해 분기한다.

```java
catch (SQLException e) {
    if (e.getErrorCode() == 23505) {  // H2 DB 키 중복 에러코드
        throw new MyDuplicateKeyException(e);
    }
    throw new MyDbException(e);
}
```

서비스에서 복구 로직을 작성할 수 있다.

```java
try {
    repository.save(member);
} catch (MyDuplicateKeyException e) {
    // 키 변경 후 재시도 등 복구 로직
}
```

### 한계

- DB마다 에러코드가 다르다. H2는 `23505`, MySQL은 `1062`, Oracle은 `1`이다.
- 수십 종류의 에러코드를 직접 매핑하는 것은 비현실적이다.
- 이 문제를 Spring이 해결해준다.

---

## 5. Spring 예외 추상화

### DataAccessException 계층 구조

Spring은 데이터 접근 관련 예외를 **런타임 예외** 계층으로 추상화한다.

```
DataAccessException (최상위, RuntimeException)
├── Transient (일시적 — 재시도하면 성공 가능)
│   ├── QueryTimeoutException
│   └── ...
└── NonTransient (비일시적 — 같은 요청은 항상 실패)
    ├── DuplicateKeyException
    ├── BadSqlGrammarException
    └── ...
```

- **Transient**: 쿼리 타임아웃, 락 획득 실패 등. 재시도 로직을 적용하면 복구 가능성이 있다.
- **NonTransient**: 문법 오류, 제약 조건 위반 등. 같은 요청은 반복해도 실패한다.

### SQLExceptionTranslator

Spring은 `SQLException`의 에러코드를 읽어서 적절한 `DataAccessException` 하위 클래스로 자동 변환해준다.

```java
SQLExceptionTranslator translator = new SQLErrorCodeSQLExceptionTranslator(dataSource);
DataAccessException resultEx = translator.translate("save", sql, e);
```

- `translate()` 메서드가 DB 에러코드를 분석해 알맞은 예외로 변환한다.
- DB가 바뀌어도 애플리케이션 코드를 수정할 필요가 없다.

### sql-error-codes.xml

에러코드 매핑 정보는 Spring 내부의 `sql-error-codes.xml`에 정의되어 있다.

- H2, MySQL, Oracle, PostgreSQL 등 주요 DB의 에러코드가 미리 등록되어 있다.
- 각 에러코드가 어떤 `DataAccessException` 하위 클래스에 매핑되는지 선언한다.
- 개발자가 직접 에러코드를 관리할 필요가 없다.

---

## 6. JdbcTemplate — 반복 제거

### 남은 문제: 반복 코드

예외 처리는 해결했지만, 리포지토리 메서드마다 반복되는 코드가 여전히 있다.

- 커넥션 획득
- `PreparedStatement` 생성 및 파라미터 바인딩
- 결과 매핑 (`ResultSet` 순회)
- 자원 정리 (`close()`)
- 예외 변환

### MemberRepositoryV5 — 최종 완성형

`JdbcTemplate`을 사용하면 이 모든 반복이 사라진다.

```java
@Repository
public class MemberRepositoryV5 implements MemberRepository {

    private final JdbcTemplate template;

    public MemberRepositoryV5(DataSource dataSource) {
        this.template = new JdbcTemplate(dataSource);
    }

    @Override
    public Member save(Member member) {
        String sql = "insert into member(member_id, money) values(?, ?)";
        template.update(sql, member.getMemberId(), member.getMoney());
        return member;
    }

    @Override
    public Member findById(String memberId) {
        String sql = "select * from member where member_id = ?";
        return template.queryForObject(sql, memberRowMapper(), memberId);
    }

    @Override
    public void update(String memberId, int money) {
        String sql = "update member set money = ? where member_id = ?";
        template.update(sql, money, memberId);
    }

    @Override
    public void delete(String memberId) {
        String sql = "delete from member where member_id = ?";
        template.update(sql, memberId);
    }

    private RowMapper<Member> memberRowMapper() {
        return (rs, rowNum) ->
            new Member(rs.getString("member_id"), rs.getInt("money"));
    }
}
```

`JdbcTemplate`이 처리해주는 것:

- 커넥션 획득 및 동기화 (`TransactionSynchronizationManager` 연동)
- `PreparedStatement` 생성
- 결과 매핑
- 자원 정리 (예외 발생 시에도 안전하게)
- `SQLException` → Spring 예외 자동 변환

---

## 7. 전체 진화 요약

| 버전 | 핵심 변경 | 서비스 계층 의존성 |
|------|----------|------------------|
| **V0** | 순수 JDBC, 예외/자원 직접 처리 | `SQLException`, `Connection`, `DataSource` |
| **V1** | 트랜잭션 적용 (커넥션 파라미터 전달) | `SQLException`, `Connection`, `DataSource` |
| **V2** | 트랜잭션 매니저 도입 | `SQLException`, `PlatformTransactionManager` |
| **V3** | `@Transactional` AOP 적용 | `SQLException` |
| **V4** | 런타임 예외 전환 + 인터페이스 의존 | 없음 (순수 비즈니스 로직) |
| **V5** | `JdbcTemplate`으로 반복 제거 | 없음 (순수 비즈니스 로직) |

V0에서 V5까지의 과정은 결국 **서비스 계층에서 기술 의존성을 하나씩 제거하는 여정**이다.

- V1~V2: 트랜잭션 처리 방식 개선
- V3: 트랜잭션 코드 자체를 AOP로 분리
- V4: 예외 의존성 제거
- V5: 반복 코드 제거

---

## 정리

이 섹션에서 배운 핵심 원칙을 요약하면 다음과 같다.

- **체크 예외는 인터페이스를 오염시킨다.** 런타임 예외로 전환해야 기술에 독립적인 인터페이스를 유지할 수 있다.
- **예외 전환 시 원인 예외를 반드시 포함한다.** `new MyDbException(e)`처럼 `cause`를 넘기지 않으면 스택 트레이스가 유실된다.
- **Spring의 예외 추상화를 활용한다.** DB별 에러코드 매핑을 직접 관리하는 것은 비현실적이다. `DataAccessException` 계층과 `SQLExceptionTranslator`가 이 문제를 해결한다.
- **JdbcTemplate은 필수다.** 커넥션 관리, 자원 정리, 예외 변환까지 모든 보일러플레이트를 제거하면서도 트랜잭션 동기화와 완벽하게 연동된다.

최종적으로 서비스 계층에는 순수 비즈니스 로직만 남게 되고, 리포지토리 계층은 SQL과 매핑 로직에만 집중할 수 있다.
