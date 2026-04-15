---
title: "@Transactional 동작 원리 — 프록시 생성부터 커밋/롤백까지"
date: 2024-03-20
categories: [Spring]
tags: [spring, transactional, aop, proxy, cglib, kotlin]
toc: true
toc_sticky: true
---

## 개요

`@Transactional`을 붙이면 트랜잭션이 알아서 처리된다. 하지만 내부에서 실제로 무슨 일이 일어나는지 모르면, 트랜잭션이 안 걸리는 상황을 만나도 원인을 찾기 어렵다. 이 글에서는 Spring이 `@Transactional`을 처리하는 전체 프로세스를 단계별로 정리한다.

## 1. 전체 흐름

```
Client Request
    ↓
Controller → Service(@Transactional) 호출
    ↓
AOP Proxy가 가로챔
    ↓
TransactionInterceptor
    ↓
PlatformTransactionManager.getTransaction()
    ↓
실제 Service 메서드 실행
    ↓
성공 → commit() / 예외 → rollback()
```

핵심은 **프록시**다. Controller가 호출하는 Service는 원본 객체가 아니라 Spring이 만든 프록시 객체이며, 이 프록시가 트랜잭션 시작과 종료를 처리한다.

## 2. 프록시 생성 — Bean 등록 시점

### 2-1. 등장인물

| 구성요소 | 역할 | 소속 |
|---------|------|------|
| **빈 후처리기** (`AnnotationAwareAspectJAutoProxyCreator`) | 프록시를 **만드는** 주체 | spring-aop |
| **어드바이저** (`BeanFactoryTransactionAttributeSourceAdvisor`) | **누구에게** + **무엇을** 적용할지 정의 | spring-tx |
| **포인트컷** (`TransactionAttributeSourcePointcut`) | `@Transactional` 붙은 곳을 감지 | spring-tx |
| **어드바이스** (`TransactionInterceptor`) | 트랜잭션 시작/커밋/롤백 로직 | spring-tx |

어드바이저는 포인트컷과 어드바이스를 묶는 컨테이너다.

```
BeanFactoryTransactionAttributeSourceAdvisor (Advisor)
├── Pointcut: TransactionAttributeSourcePointcut  ← "어디에"
└── Advice:  TransactionInterceptor               ← "무엇을"
```

### 2-2. 프록시 생성 프로세스

Spring 컨테이너가 Bean을 등록할 때, 빈 후처리기가 다음 과정을 수행한다.

```
1. Bean 생성 완료 → postProcessAfterInitialization() 호출
    ↓
2. 등록된 Advisor들을 순회하며 이 Bean에 적용 가능한지 확인
    ↓
3. BeanFactoryTransactionAttributeSourceAdvisor의 Pointcut이
   해당 Bean의 클래스/메서드에 @Transactional이 있는지 검사
    ↓
4. 매칭되면 CGLIB 프록시 생성
    ↓
5. 프록시를 Bean으로 등록 (원본 대체)
```

이후 Controller가 `@Autowired`로 Service를 주입받으면, 실제로는 이 **프록시 객체**가 주입된다.

### 2-3. 빈 후처리기의 계층 구조

프록시를 생성하는 빈 후처리기는 여러 개가 등록될 수 있지만, Spring은 상위 호환되는 하나만 남긴다.

```
AbstractAutoProxyCreator
    └─ AbstractAdvisorAutoProxyCreator
        └─ InfrastructureAdvisorAutoProxyCreator    ← @Transactional 전용
            └─ AspectJAwareAdvisorAutoProxyCreator
                └─ AnnotationAwareAspectJAutoProxyCreator  ← 최종 동작
```

`AnnotationAwareAspectJAutoProxyCreator`가 `@Transactional`과 `@Aspect` 기반 AOP를 모두 처리한다.

## 3. 트랜잭션 실행 — 메서드 호출 시점

### 3-1. TransactionInterceptor 진입

Controller에서 `service.update()`를 호출하면:

```
Proxy.update() 호출
    ↓
TransactionInterceptor.invoke(MethodInvocation)
    ↓
TransactionAspectSupport.invokeWithinTransaction()
```

`TransactionInterceptor`는 AOP의 `MethodInterceptor` 구현체로, 어드바이저에 등록된 **어드바이스**다.

### 3-2. 트랜잭션 시작

```
invokeWithinTransaction()
    ↓
PlatformTransactionManager.getTransaction(TransactionDefinition)
    ↓
1. DataSource에서 Connection 획득
2. Connection.setAutoCommit(false)
3. ThreadLocal에 트랜잭션 바인딩
    ↓
TransactionStatus 반환
```

`PlatformTransactionManager`는 인터페이스이고, 실제 구현체는 프레임워크마다 다르다.

| 프레임워크 | 구현체 |
|-----------|--------|
| JDBC / MyBatis | `DataSourceTransactionManager` |
| JPA / Hibernate | `JpaTransactionManager` |
| Exposed ORM | `SpringTransactionManager` |

### 3-3. 실제 메서드 실행

```
invocation.proceed()  ← 원본 Service 메서드 실행
```

메서드 내 모든 DB 작업은 **같은 Connection, 같은 트랜잭션** 안에서 실행된다.

### 3-4. 트랜잭션 종료

**정상 반환 시:**
```
TransactionAspectSupport.commitTransactionAfterReturning()
    ↓
PlatformTransactionManager.commit(TransactionStatus)
    ↓
Connection.commit() → Connection 반환 (Pool로)
```

**예외 발생 시:**
```
TransactionAspectSupport.completeTransactionAfterThrowing()
    ↓
rollbackOn 판단
    ↓
PlatformTransactionManager.rollback(TransactionStatus)
    ↓
Connection.rollback() → Connection 반환 → 예외 다시 throw
```

## 4. Rollback 규칙

| 예외 타입 | 기본 동작 |
|----------|----------|
| `RuntimeException` (unchecked) | rollback |
| `Error` | rollback |
| `Exception` (checked) | **commit** (주의!) |

Java의 checked exception은 기본적으로 **커밋**된다. 이 동작이 의도하지 않은 데이터 저장으로 이어질 수 있으므로, 필요하면 명시적으로 지정한다.

```kotlin
@Transactional(rollbackFor = [Exception::class])       // checked도 rollback
@Transactional(noRollbackFor = [SomeException::class])  // 특정 예외는 commit
```

Kotlin에서는 checked exception이 없으므로 큰 문제가 되지 않지만, Java 라이브러리를 호출할 때는 주의가 필요하다.

## 5. 전파 속성 (Propagation)

`@Transactional` 메서드가 다른 `@Transactional` 메서드를 호출할 때의 동작을 결정한다.

| 전파 속성 | 동작 |
|----------|------|
| `REQUIRED` (기본값) | 기존 트랜잭션 있으면 참여, 없으면 새로 생성 |
| `REQUIRES_NEW` | 항상 새 트랜잭션 생성 (기존 트랜잭션 일시 중단) |
| `SUPPORTS` | 기존 트랜잭션 있으면 참여, 없으면 트랜잭션 없이 실행 |
| `NOT_SUPPORTED` | 트랜잭션 없이 실행 (기존 트랜잭션 일시 중단) |
| `MANDATORY` | 기존 트랜잭션 필수, 없으면 예외 발생 |
| `NEVER` | 트랜잭션이 있으면 예외 발생 |
| `NESTED` | 기존 트랜잭션 안에 savepoint를 설정하여 부분 롤백 가능 |

대부분 기본값 `REQUIRED`로 충분하다. `REQUIRES_NEW`는 로그 저장처럼 "본 트랜잭션이 롤백되더라도 반드시 커밋해야 하는" 경우에 사용한다.

## 6. readOnly 최적화

```kotlin
@Transactional(readOnly = true)
```

`readOnly = true`가 설정되면:

1. `Connection.setReadOnly(true)` — DB 드라이버가 읽기 전용 힌트를 사용
2. MySQL의 경우 읽기 전용 트랜잭션으로 처리되어 락 경합이 줄어듦
3. JPA/Hibernate 환경에서는 더티 체킹 비활성화, 플러시 모드를 MANUAL로 설정하여 성능 향상

조회 전용 Service에 붙이면 불필요한 쓰기 락을 방지할 수 있다.

## 7. 주의사항

### 7-1. 같은 클래스 내부 호출은 프록시를 거치지 않음

```kotlin
@Service
class SomeService {
    @Transactional
    fun methodA() {
        methodB()  // ❌ 프록시를 거치지 않음! @Transactional 무시
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    fun methodB() { ... }
}
```

`this.methodB()`는 원본 객체의 메서드를 직접 호출하므로, 프록시를 거치지 않아 트랜잭션 설정이 무시된다.

**해결 방법:**
- 별도 Bean으로 분리
- `TransactionTemplate`을 직접 사용

### 7-2. private 메서드에는 @Transactional 무효

```kotlin
@Transactional
private fun someMethod() { ... }  // ❌ 동작하지 않음
```

CGLIB은 클래스 상속 기반으로 프록시를 만들기 때문에 `private` 메서드는 오버라이드할 수 없다. `public` 또는 `protected` 메서드에만 적용 가능하다.

### 7-3. Kotlin의 final 클래스와 kotlin-spring 플러그인

Kotlin은 클래스가 기본적으로 `final`이다. CGLIB은 상속 기반이므로 `final` 클래스는 프록시를 만들 수 없다.

이를 해결하는 것이 `kotlin-spring` 컴파일러 플러그인이다.

```kotlin
// build.gradle.kts
kotlin("plugin.spring")
```

이 플러그인이 컴파일 시점에 `@Service`, `@Transactional`, `@Component` 등이 붙은 클래스를 자동으로 `open`으로 변경한다.

```kotlin
// 소스 코드 — final처럼 보이지만
@Service
@Transactional
class MyService(...)

// 바이트코드 — 실제로는 open
open class MyService(...)
```

만약 이 플러그인 없이 Kotlin에서 `@Transactional`을 쓰면 프록시 생성 실패로 에러가 발생한다.

## 8. Exposed ORM에서의 트랜잭션

Exposed를 사용하는 프로젝트에서는 `PlatformTransactionManager` 구현체로 `SpringTransactionManager`를 등록한다.

```kotlin
@Bean
fun transactionManager(dataSource: DataSource): PlatformTransactionManager {
    return SpringTransactionManager(dataSource)
}
```

`SpringTransactionManager`는 `DataSourceTransactionManager`를 확장하면서 추가로:
- Exposed의 내부 `TransactionManager`에 현재 트랜잭션을 등록
- `ThreadLocal`에 Exposed `Transaction` 객체를 바인딩

이로 인해 `@Transactional` 안에서 Exposed DSL(`Table.select`, `Table.insert` 등)이 자연스럽게 동작한다.

만약 일반 `DataSourceTransactionManager`를 쓰면 Exposed DSL이 트랜잭션 컨텍스트를 찾지 못해 에러가 발생한다.

Exposed는 자체적으로 `transaction { }` 블록도 제공하지만, Spring과 함께 사용할 때는 `@Transactional`을 사용하는 것이 일관성 있다.

## 정리

`@Transactional` 하나가 동작하기 위해 AOP 프록시, 어드바이저, 트랜잭션 매니저가 협력한다.

1. **Bean 등록 시점**: 빈 후처리기가 `@Transactional`을 감지하고 CGLIB 프록시를 생성
2. **메서드 호출 시점**: 프록시가 `TransactionInterceptor`를 통해 트랜잭션을 시작
3. **메서드 종료 시점**: 정상이면 커밋, RuntimeException이면 롤백
4. **프레임워크별 구현체**: JDBC는 `DataSourceTransactionManager`, Exposed는 `SpringTransactionManager`

트랜잭션이 안 걸리는 문제를 만나면: **프록시를 거치는가?** → **메서드가 public인가?** → **같은 클래스 내부 호출이 아닌가?** 순서로 확인하면 대부분 원인을 찾을 수 있다.
