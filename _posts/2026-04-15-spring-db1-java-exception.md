---
title: "[스프링 DB 1편 - 5] 자바 예외 이해"
date: 2026-04-15 18:05:00 +0900
categories: [spring-db1]
tags: [spring, java, exception, checked-exception, unchecked-exception, inflearn, 김영한]
toc: true
toc_sticky: true
---

## 개요

김영한님의 "스프링 DB 1편 - 데이터 접근 핵심 원리" 섹션 5에서 다루는 자바 예외의 핵심 내용을 정리한다. 자바 예외 계층 구조부터 체크/언체크 예외의 차이, 그리고 실무에서 왜 런타임 예외를 기본으로 사용해야 하는지까지 한 흐름으로 요약한다.

---

## 1. 예외 계층 구조

자바 예외는 다음과 같은 계층을 가진다.

```
Object
  └── Throwable
        ├── Error           (시스템 오류, 복구 불가)
        └── Exception       (체크 예외)
              └── RuntimeException  (언체크 예외)
```

- **Error**: `OutOfMemoryError`, `StackOverflowError` 등 시스템 레벨 오류. 애플리케이션에서 잡으면 안 된다.
- **Exception**: 컴파일러가 예외 처리를 강제하는 체크 예외의 최상위 클래스.
- **RuntimeException**: `Exception`의 하위이지만 컴파일러가 예외 처리를 강제하지 않는 언체크 예외.

핵심 구분 기준은 **`RuntimeException`을 상속하는가 여부**다. 상속하면 언체크, 아니면 체크 예외다.

---

## 2. 체크 예외 (Checked Exception)

`Exception`을 직접 상속한 예외 클래스가 체크 예외다.

```java
public class MyCheckedException extends Exception {
    public MyCheckedException(String message) {
        super(message);
    }
}
```

### 특징

- 컴파일러가 예외 처리를 **강제**한다.
- 반드시 `catch`로 잡거나, `throws`로 호출자에게 던져야 한다.
- 둘 다 하지 않으면 **컴파일 오류**가 발생한다.

### 처리 방식

- **잡아서 처리**: `try-catch`로 직접 처리
- **던지기**: 메서드 시그니처에 `throws`를 선언하여 호출자에게 위임

```java
// 잡아서 처리
public void callCatch() {
    try {
        repository.call();
    } catch (MyCheckedException e) {
        log.info("예외 처리, message={}", e.getMessage());
    }
}

// 밖으로 던지기
public void callThrows() throws MyCheckedException {
    repository.call();
}
```

---

## 3. 언체크 예외 (Unchecked Exception)

`RuntimeException`을 상속한 예외 클래스가 언체크 예외다.

```java
public class MyUncheckedException extends RuntimeException {
    public MyUncheckedException(String message) {
        super(message);
    }
}
```

### 특징

- 컴파일러가 예외 처리를 강제하지 **않는다**.
- `throws` 선언을 생략해도 **자동으로 상위 호출자에게 전파**된다.
- 잡고 싶으면 `catch`로 잡을 수 있지만, 필수는 아니다.

### 체크 예외와의 핵심 차이

| 구분 | 체크 예외 | 언체크 예외 |
|------|-----------|-------------|
| 컴파일러 체크 | O | X |
| `throws` 선언 | 필수 | 생략 가능 |
| 예외 전파 | 명시적 선언 필요 | 자동 전파 |
| 대표 예 | `IOException`, `SQLException` | `NullPointerException`, `IllegalArgumentException` |

---

## 4. 체크 예외의 문제점

체크 예외는 안전해 보이지만, 실무에서 심각한 문제를 만든다.

### 4-1. 복구 불가능한 예외

- `SQLException`, `ConnectException` 같은 예외는 서비스 계층에서 복구할 수 없다.
- 결국 서비스와 컨트롤러가 아무 의미 없이 `throws`만 선언하게 된다.

```java
// 서비스 - 할 수 있는 게 없다
public void logic() throws SQLException, ConnectException {
    repository.call();
    networkClient.call();
}

// 컨트롤러 - 역시 할 수 있는 게 없다
public void request() throws SQLException, ConnectException {
    service.logic();
}
```

### 4-2. 의존 관계 문제

- 서비스/컨트롤러가 `throws SQLException`을 선언하면, JDBC라는 **특정 기술에 의존**하게 된다.
- JPA로 변경하면 `throws JPAException`으로 모두 바꿔야 한다.
- OCP(개방-폐쇄 원칙) 위반이다.

### 4-3. throws Exception 안티패턴

- 귀찮아서 `throws Exception`으로 퉁치는 경우가 많다.
- 이렇게 하면 모든 예외를 밖으로 던지겠다는 의미가 되어, **컴파일러의 체크 예외 검증 기능이 무력화**된다.
- 중요한 체크 예외를 놓치게 되므로 절대 사용하면 안 된다.

---

## 5. 해결책: 런타임 예외로 변환

체크 예외를 런타임 예외로 전환하면 위 문제가 모두 해결된다.

### 변환 예시

```java
public class RuntimeSQLException extends RuntimeException {
    public RuntimeSQLException(Throwable cause) {
        super(cause);
    }
}
```

리포지토리에서 체크 예외를 잡아 런타임 예외로 변환한다.

```java
public void call() {
    try {
        sql();
    } catch (SQLException e) {
        throw new RuntimeSQLException(e); // 원인 예외 포함
    }
}
```

### 변환 후 달라지는 점

- 서비스와 컨트롤러에서 `throws` 선언이 **사라진다**.
- JDBC에 대한 의존이 제거되어 기술 변경에 영향받지 않는다.
- 예외 공통 처리는 `ControllerAdvice` 같은 곳에서 한 번에 처리한다.

### 원인 예외(cause) 포함은 필수

런타임 예외로 변환할 때 **반드시 원래 예외를 cause로 포함**해야 한다.

```java
// 잘못된 예 - 원인 예외 누락
catch (SQLException e) {
    throw new RuntimeSQLException(); // 원인을 알 수 없게 된다
}

// 올바른 예 - 원인 예외 포함
catch (SQLException e) {
    throw new RuntimeSQLException(e); // 스택 트레이스에 원인이 남는다
}
```

cause를 넣지 않으면 장애 발생 시 근본 원인을 추적할 수 없어 치명적이다.

---

## 6. 실무 원칙 정리

### 기본 원칙: 런타임 예외를 사용하라

- 대부분의 예외는 복구 불가능하다. 런타임 예외로 만들어 서비스 계층이 신경 쓰지 않게 한다.
- 예외 공통 처리 로직(`ControllerAdvice` 등)에서 일괄 처리한다.
- 로그를 남기고 개발자에게 알림을 보내는 것이 현실적인 처리 방법이다.

### 예외 상황: 체크 예외를 쓰는 경우

- **비즈니스적으로 의도적으로 던지는 예외**에만 체크 예외를 고려한다.
- 예: 잔고 부족, 포인트 부족 등 호출자가 반드시 처리해야 하는 경우
- 하지만 이 경우에도 런타임 예외로 만들고 문서화하는 방식을 선호하는 팀이 많다.

### 요약

| 상황 | 선택 |
|------|------|
| 복구 불가능한 예외 (DB 오류, 네트워크 오류 등) | 런타임 예외 |
| 비즈니스 로직상 반드시 처리가 필요한 예외 | 체크 예외 (또는 런타임 예외 + 문서화) |
| 외부 라이브러리의 체크 예외 | 런타임 예외로 변환 (cause 포함) |

---

## 마무리

자바의 체크 예외는 좋은 의도로 만들어졌지만, 실무에서는 오히려 코드의 의존성을 높이고 유지보수를 어렵게 만든다. 최근 라이브러리들이 대부분 런타임 예외를 사용하는 것도 같은 이유다. 핵심은 단순하다.

- **기본은 런타임 예외**
- **변환 시 원인 예외(cause) 반드시 포함**
- **정말 처리가 필요한 비즈니스 예외만 체크 예외 고려**
