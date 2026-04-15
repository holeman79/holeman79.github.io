---
title: "트랜잭션의 한계와 분산 시스템에서의 데이터 정합성"
date: 2026-03-15
categories: [Infra]
tags: [transaction, distributed-system, eventual-consistency, outbox-pattern, idempotency]
toc: true
toc_sticky: true
---

## RDB 트랜잭션 — 단일 DB에서의 보장

RDB의 트랜잭션은 ACID를 보장한다.

```sql
BEGIN;
  UPDATE account SET balance = balance - 1000 WHERE id = 1;
  UPDATE account SET balance = balance + 1000 WHERE id = 2;
COMMIT;
```

이 트랜잭션은 **성공하면 딱 한 번 반영**, **실패하면 전체 롤백**된다. 중간 상태가 존재하지 않는다.

단일 DB 트랜잭션 내에서는 **exactly-once**를 보장한다고 볼 수 있다.

## 트랜잭션의 한계 — 범위를 넘어서면

문제는 트랜잭션의 범위를 넘어서는 순간 발생한다.

| 범위 | exactly-once 보장 |
|------|:-:|
| 단일 DB 트랜잭션 | O |
| DB + 외부 시스템 (ES, Kafka 등) | X |
| 분산 트랜잭션 (2PC) | O (성능 대가) |

### DB + Kafka — 왜 안 되는가

```java
@Transactional
public void placeOrder(OrderRequest request) {
    orderRepository.save(order);               // 1. DB 저장
    kafkaTemplate.send("order-events", order);  // 2. Kafka 발행
}
```

`@Transactional`은 DB 트랜잭션만 관리한다. Kafka 발행은 트랜잭션 밖이다.

```
시나리오 A: DB 커밋 성공 → Kafka 발행 실패 → 주문은 있는데 이벤트 없음
시나리오 B: Kafka 발행 성공 → DB 커밋 실패 → 이벤트는 갔는데 주문 없음
```

**서로 다른 시스템 간에는 단일 트랜잭션이 불가능**하다. MySQL 트랜잭션이 Kafka를 포함할 수 없고, MongoDB 트랜잭션이 Elasticsearch를 포함할 수 없다.

### DB + Elasticsearch — 같은 문제

```kotlin
@Transactional
fun hiddenCertificates(certificateIds: List<String>) {
    outboxRepository.save(event)                         // MongoDB 저장
    certificateRepository.updateCertificateHidden(...)    // ES 업데이트
}
```

MongoDB와 Elasticsearch는 서로 다른 시스템이므로 하나의 트랜잭션으로 묶을 수 없다.

## 해결 전략 1: Outbox 패턴

트랜잭션으로 묶을 수 없다면, **트랜잭션이 보장되는 범위 안에서만 데이터를 저장**하고 나머지는 비동기로 처리한다.

```
API Server                    CDC                    Consumer
    │                          │                        │
    │ @Transactional           │                        │
    │  └─ Outbox 이벤트 저장    │                        │
    │──→ DB (트랜잭션 보장)     │                        │
    │                          │ 변경 감지               │
    │                          │──→ Kafka 발행           │
    │                          │                        │ 메시지 수신
    │                          │                        │──→ ES 업데이트
```

핵심: **API 서버는 자신의 DB에 이벤트만 저장**한다. 트랜잭션이 보장되는 범위에서만 작업한다. ES 업데이트 같은 외부 시스템 처리는 CDC + Kafka를 통해 비동기로 처리한다.

이 구조에서 트랜잭션이 실패하면 이벤트 자체가 저장되지 않으므로, 후속 처리도 일어나지 않는다. 트랜잭션이 성공하면 이벤트가 반드시 저장되고, CDC가 감지하여 처리한다.

## 해결 전략 2: at-least-once + 멱등성

Outbox 패턴에서 메시지 전달은 **at-least-once**(최소 한 번)이다.

### 메시지 전달 보장 수준

| 방식 | 설명 | 중복 | 유실 |
|------|------|:----:|:----:|
| at-most-once | 최대 1번 (유실 허용) | X | O |
| **at-least-once** | 최소 1번 (중복 허용) | O | X |
| exactly-once | 정확히 1번 | X | X |

at-least-once에서 중복이 발생하는 시나리오:

```
1. Consumer가 메시지 수신
2. ES 업데이트 성공
3. offset 커밋 전에 앱 다운
4. 재시작 → 같은 메시지 다시 수신
5. ES 업데이트 다시 실행 (중복)
```

### 멱등성으로 중복 해결

중복 실행이 문제가 되지 않으려면, 연산이 **멱등(idempotent)**해야 한다.

```sql
-- 멱등한 연산: 여러 번 실행해도 결과 동일
UPDATE certificates SET hidden = true WHERE id = 'cert-001'

-- 멱등하지 않은 연산: 중복 실행 시 데이터 오류
UPDATE accounts SET balance = balance - 1000 WHERE id = 'acc-001'
```

`hidden = true`로 덮어쓰는 연산은 1번 실행하든 10번 실행하든 결과가 같다. 하지만 잔액 차감은 실행할 때마다 결과가 달라진다.

**at-least-once + 멱등한 연산 = 결과적 정합성(eventual consistency)**

exactly-once는 구현 비용이 너무 높고, at-most-once는 유실 위험이 있다. 실무에서는 **at-least-once + 멱등성** 조합이 가장 많이 쓰인다.

## @Transactional과 MongoDB

Spring의 `@Transactional`은 기본적으로 RDB(JPA)의 `PlatformTransactionManager`를 사용한다. MongoDB에서 `@Transactional`을 사용하려면:

1. **Replica Set이 필수** — standalone MongoDB에서는 트랜잭션이 지원되지 않는다
2. **MongoTransactionManager** 빈이 등록되어 있어야 한다

MongoDB 트랜잭션은 RDB와 동일하게 all-or-nothing을 보장한다. Outbox 이벤트를 MongoDB에 저장할 때 `@Transactional`을 사용하면, 비즈니스 데이터와 이벤트가 함께 커밋되거나 함께 롤백된다.

## Spring Kafka와 offset 커밋

Spring Kafka의 `@KafkaListener`는 기본적으로 **at-least-once** 방식이다.

### offset 커밋 타이밍

```
메시지 수신
  → @KafkaListener 메서드 실행
  → 메서드 정상 리턴
  → offset 커밋
```

메서드가 **정상 리턴해야** offset이 커밋된다. 예외가 발생하면 offset을 커밋하지 않고, 재시작 시 같은 메시지를 다시 수신한다.

### AckMode별 커밋 시점

| AckMode | 커밋 시점 |
|---------|----------|
| **BATCH** (기본값) | poll()로 가져온 레코드를 모두 처리한 후 |
| RECORD | 각 레코드 처리 후 즉시 |
| MANUAL | `Acknowledgment.acknowledge()` 직접 호출 |

## 멱등성 설계 패턴

### 조건부 조회로 중복 방지

```kotlin
// MongoDB에서 status=OPEN인 것만 조회
val riskFactors = repository.findOpenByRelServiceIds(assetIds)
```

이미 AUTO_CLOSED 처리된 RiskFactor는 조회되지 않으므로, 같은 메시지가 중복 수신되어도 재처리 대상이 없다.

### 덮어쓰기 연산

```kotlin
// ES에서 riskfactorMessages를 빈 배열로 설정
val updateFields = mapOf("riskfactorMessages" to emptyList<Any>())
esClient.update(index, documentId, updateFields)
```

이미 빈 배열이어도 다시 빈 배열로 설정하는 것은 결과가 동일하다.

### 조회 → 판단 → 저장 패턴

```kotlin
// 1. 조회
val assets = repository.findDocumentsWithRiskFactorIds(index, riskFactorIds)

// 2. 도메인 로직으로 판단 (순수 함수, 테스트 가능)
val filtered = asset.removeByRiskFactorIds(riskFactorIds)

// 3. 저장
repository.updateRiskFactorMessages(index, asset.id, filtered)
```

비즈니스 로직을 DB 쿼리(Painless script 등)에 넣지 않고 도메인 객체에서 처리하면, 멱등성 검증이 쉬워지고 단위 테스트가 가능해진다.

## 정리

| 개념 | 핵심 |
|------|------|
| **RDB 트랜잭션** | 단일 DB 내에서 exactly-once 보장 |
| **분산 시스템** | 서로 다른 시스템 간 단일 트랜잭션 불가 |
| **Outbox 패턴** | 트랜잭션 범위 안에서만 저장, 나머지는 비동기 |
| **at-least-once** | 유실 없이 최소 1번 전달, 중복 가능 |
| **멱등성** | 중복 실행해도 결과 동일 → at-least-once의 중복 문제 해결 |
| **결과적 정합성** | at-least-once + 멱등성 = 실시간은 아니지만 최종적으로 정합 |

분산 시스템에서 **exactly-once는 비현실적**이다. 대신 **"유실보다 중복이 낫다"**는 전략으로 at-least-once를 채택하고, 멱등한 연산 설계로 중복을 무해하게 만드는 것이 실무의 표준 패턴이다.
