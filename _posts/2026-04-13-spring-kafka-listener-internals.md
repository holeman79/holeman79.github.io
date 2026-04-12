---
title: "@KafkaListener는 AOP가 아니다 — Spring Kafka 내부 동작 원리"
date: 2026-04-13T10:00:00+09:00
categories: [Spring]
tags: [spring-kafka, kafka-listener, bean-post-processor, aop]
---

## @Transactional과 @KafkaListener의 차이

Spring을 쓰다 보면 애노테이션 기반으로 동작하는 기능들이 많다. `@Transactional`과 `@KafkaListener` 둘 다 메서드에 붙이면 마법처럼 동작한다. 하지만 내부 메커니즘은 완전히 다르다.

## @Transactional — AOP 프록시 방식

`@Transactional`은 **AOP 프록시**로 동작한다.

```
원본 빈 → BeanPostProcessor → 프록시 빈으로 교체 (빈 자체가 바뀜)
```

`InfrastructureAdvisorAutoProxyCreator`가 `@Transactional`이 붙은 빈을 감지하면, 원본 빈을 CGLIB 프록시로 감싸서 **빈 자체를 교체**한다.

```
호출자 → 프록시.method()
            ↓
         트랜잭션 시작
            ↓
         원본.method() 실행
            ↓
         트랜잭션 커밋/롤백
```

메서드 호출을 **가로채서** 트랜잭션을 래핑하는 구조다.

## @KafkaListener — 등록(Registration) 패턴

`@KafkaListener`는 AOP가 아니다. **메서드를 Kafka 메시지 리스너로 등록**하는 방식이다.

```
원본 빈 → BeanPostProcessor → 빈은 그대로 + 메서드를 컨테이너에 등록 (사이드 이펙트)
```

### 동작 순서

#### 1단계: 빈 등록 시점

`KafkaListenerAnnotationBeanPostProcessor`가 `postProcessAfterInitialization()`에서 `@KafkaListener`가 붙은 메서드를 스캔한다.

```java
// KafkaListenerAnnotationBeanPostProcessor 내부 (간략화)
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    // @KafkaListener 붙은 메서드 스캔
    // MethodKafkaListenerEndpoint로 래핑
    // KafkaListenerEndpointRegistry에 등록
    return bean;  // 빈 자체는 변경하지 않음!
}
```

핵심: **빈을 프록시로 교체하지 않고, 원본 빈을 그대로 반환**한다. 대신 메서드 정보를 리스너 레지스트리에 등록하는 **사이드 이펙트**를 발생시킨다.

#### 2단계: 컨테이너 생성

등록된 엔드포인트마다 `ConcurrentMessageListenerContainer`가 생성된다. 이 컨테이너가 Kafka를 polling하는 루프를 돌린다.

#### 3단계: 메시지 수신 시

```
KafkaMessageListenerContainer (polling 루프)
  → ConsumerRecords를 poll()
  → MessagingMessageListenerAdapter.onMessage() 호출
    → 리플렉션으로 @KafkaListener 메서드 직접 invoke
  → 메서드 정상 리턴 시 offset commit
```

## 비교 요약

|  | @Transactional | @KafkaListener |
|---|---|---|
| **BeanPostProcessor** | InfrastructureAdvisorAutoProxyCreator | KafkaListenerAnnotationBeanPostProcessor |
| **후처리 결과** | 프록시 빈으로 교체 | 빈은 그대로, 메서드를 리스너로 등록 |
| **런타임 동작** | 프록시가 메서드 호출 전후에 개입 | 컨테이너가 직접 메서드를 invoke |
| **메커니즘** | AOP (가로채기) | Registration (등록) |

## 공통점

둘 다 `BeanPostProcessor`의 `postProcessAfterInitialization()`에서 애노테이션을 스캔하는 것까지는 동일하다. 그 이후 **프록시를 만드느냐 / 등록만 하느냐**가 갈린다.

## Offset 커밋 타이밍

`@KafkaListener` 메서드의 offset 커밋은 `ContainerProperties.AckMode`에 따라 결정된다.

| AckMode | 커밋 시점 |
|---------|----------|
| **BATCH** (기본값) | poll()로 가져온 레코드를 모두 처리한 후 |
| RECORD | 각 레코드 처리 후 즉시 |
| MANUAL | `Acknowledgment.acknowledge()` 직접 호출 |

기본값 BATCH 모드에서는 리스너 메서드가 **정상 리턴해야** offset이 커밋된다. 예외가 발생하면 `DefaultErrorHandler`가 처리하고 offset을 커밋하지 않아 **재시작 시 같은 메시지를 다시 수신**한다.

## 정리

- `@KafkaListener`는 AOP 프록시가 아니라 **리스너 등록 패턴**이다
- 빈 자체는 변경되지 않고, 메서드가 Kafka 컨테이너의 콜백으로 등록된다
- `@Transactional`과의 공통점은 `BeanPostProcessor`를 사용한다는 것뿐이다
- offset 커밋은 메서드 정상 리턴 후 자동으로 이루어진다 (BATCH 모드 기준)
