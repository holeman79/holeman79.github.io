---
title: "Outbox + CDC 패턴 — 분산 시스템에서 데이터 정합성 보장하기"
date: 2026-04-12T11:00:00+09:00
categories: [Infra]
tags: [outbox-pattern, cdc, kafka, mongodb, eventual-consistency]
---

## 문제: 두 시스템을 동시에 업데이트해야 한다

하나의 API 요청에서 MongoDB와 Elasticsearch를 모두 업데이트해야 하는 상황을 생각해보자.

```
API: Certificate 제외 요청
  1. MongoDB에 이벤트 저장
  2. Elasticsearch에서 hidden = true 업데이트
```

두 작업이 서로 다른 데이터소스이므로 **단일 트랜잭션으로 묶을 수 없다.**

```
시나리오 A: MongoDB 저장 성공 → ES 업데이트 실패 → 데이터 불일치
시나리오 B: ES 업데이트 성공 → MongoDB 저장 실패 → 데이터 불일치
```

RDB 트랜잭션은 단일 DB 내에서 ACID를 보장하지만, **DB + 외부 시스템** 사이에서는 exactly-once를 보장할 수 없다.

## 해결: Outbox + CDC 패턴

### 핵심 아이디어

1. **API 서버는 MongoDB에 Outbox 이벤트만 저장한다** (트랜잭션 보장)
2. **CDC(Change Data Capture)**가 MongoDB 변경을 감지하여 Kafka로 발행한다
3. **Subscriber**가 Kafka 메시지를 수신하여 ES를 업데이트한다

```
API Server                CDC App                 Subscriber
    │                        │                        │
    │ OutboxEvent 저장       │                        │
    │──→ MongoDB             │                        │
    │    (트랜잭션 보장)       │                        │
    │                        │ Change Stream 감지      │
    │                        │──→ Kafka 발행           │
    │                        │                        │ 메시지 수신
    │                        │                        │──→ ES 업데이트
```

### 왜 이 구조가 안전한가

**API 서버는 MongoDB 저장만 책임진다.** MongoDB 트랜잭션이 실패하면 이벤트 자체가 저장되지 않으므로 ES 업데이트도 일어나지 않는다. 트랜잭션이 성공하면 이벤트가 반드시 저장되고, CDC가 이를 감지하여 Kafka로 발행한다.

## 실제 구현 — Spring Boot + MongoDB + Kafka

### OutboxEvent 엔티티

```kotlin
data class OutboxEvent(
    var id: String? = null,
    val aggregateType: AggregateType,
    val aggregateId: String,
    val eventType: OutboxEventType,
    val routingTopic: String,
    val payload: OutboxPayload,
    val status: OutboxStatus = OutboxStatus.PENDING,
    val retryCount: Int = 0,
    val createdAt: Date = Date(),
    var processedAt: Date? = null
)
```

`routingTopic`은 `AggregateType`에서 결정된다. CDC는 이 필드를 읽어 Kafka 토픽을 동적으로 결정한다.

```kotlin
enum class AggregateType(val topic: String) {
    ASSET_STATE_CHANGE("asset-state-change"),
    CERTIFICATE_STATE_CHANGE("certificate-state-change")
}
```

### API → Outbox 저장

```kotlin
@Transactional
fun hiddenCertificates(certificateIds: List<String>, customerId: String, username: String) {
    outboxRepository.save(
        OutboxEvent.create(
            aggregateType = AggregateType.CERTIFICATE_STATE_CHANGE,
            aggregateId = customerId,
            eventType = OutboxEventType.CERTIFICATE_HIDDEN,
            payload = CertificateStateChangePayload.create(
                customerId = customerId,
                certificateIds = certificateIds,
                createdBy = username
            )
        )
    )
}
```

`@Transactional` 안에서 OutboxEvent를 MongoDB에 저장하는 것이 전부다. ES 업데이트는 하지 않는다.

### CDC — MongoDB Change Stream → Kafka

```kotlin
private fun publishToKafka(document: Document) {
    val routingTopic = document.getString("routingTopic") ?: return
    val aggregateId = document.getString("aggregateId") ?: return

    val message = OutboxMessage(
        id = eventId,
        aggregateId = aggregateId,
        eventType = document.getString("eventType") ?: "",
        payload = OutboxPayload(payloadMap)
    )

    kafkaTemplate.send(routingTopic, aggregateId, message).get()
    outboxRepository.markProcessed(eventId)
}
```

MongoDB Change Stream으로 `outbox_events` 컬렉션의 INSERT를 감지하고, `routingTopic` 필드를 읽어 해당 Kafka 토픽으로 발행한다. **토픽별 분기 로직이 없다** — 필드 기반 동적 라우팅이다.

### Subscriber — Kafka → ES 업데이트

```kotlin
@KafkaListener(
    topics = ["certificate-state-change"],
    groupId = "asm-certificate-state-change-group"
)
fun handleCertificateStateChange(message: OutboxMessage) {
    val payload = CertificateStateChangePayload.from(message.payload)

    certificateRepository.updateCertificateHidden(
        payload.certificateIds,
        payload.customerId,
        payload.createdBy,
        true
    )
    outboxRepository.markProcessed(message.id)
}
```

## 메시지 전달 보장: at-least-once + 멱등성

이 구조에서 정합성이 깨질 수 있는 시나리오를 살펴보자.

### ES 업데이트 성공 → markProcessed 실패

```
1. ES hidden = true 업데이트 ✅
2. outboxRepository.markProcessed() ❌ (앱 다운)
3. 재시작 → 같은 메시지 재수신
4. ES hidden = true 다시 업데이트 (이미 true이므로 결과 동일)
```

`hidden = true`로 덮어쓰는 연산은 **멱등(idempotent)**하다. 여러 번 실행해도 결과가 같으므로, 중복 실행이 문제가 되지 않는다.

### ES 업데이트 실패

```
1. ES 업데이트 ❌ (예외 발생)
2. outboxRepository.markFailed() 호출
3. Kafka offset 커밋 안 됨 (예외 throw)
4. 재시작 → 같은 메시지 재수신 → 재시도
```

**at-least-once** (최소 한 번 전달) + **멱등한 연산** 조합으로 **결과적 정합성(eventual consistency)**을 달성한다.

## Outbox 이벤트 상태 관리

```kotlin
enum class OutboxStatus {
    PENDING,    // 생성됨, CDC 미처리
    PROCESSED,  // Kafka 발행 + Subscriber 처리 완료
    FAILED      // 처리 실패 (재시도 대상)
}
```

실패한 이벤트는 `retryCount`를 증가시키며, 최대 재시도 횟수 초과 시 수동 개입이 필요하다.

## 정리

| 항목 | 설명 |
|------|------|
| **핵심 원칙** | API 서버는 Outbox 저장만, 실제 처리는 Subscriber가 비동기로 |
| **트랜잭션 범위** | MongoDB 단일 트랜잭션 (Outbox 저장만 보장) |
| **메시지 전달** | at-least-once (최소 한 번) |
| **정합성** | 멱등한 연산 + at-least-once = 결과적 정합성 |
| **라우팅** | routingTopic 필드 기반 동적 라우팅 (CDC에 분기 로직 불필요) |

Outbox + CDC 패턴은 "분산 트랜잭션 없이 데이터 정합성을 보장하는 방법"이다. 완벽한 exactly-once는 아니지만, **at-least-once + 멱등성**으로 실무에서 충분한 수준의 정합성을 달성할 수 있다.
