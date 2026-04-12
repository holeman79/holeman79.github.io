---
title: "Outbox + CDC 패턴 (RDB편) — JPA + MySQL에서 데이터 정합성 보장하기"
date: 2026-04-12T12:00:00+09:00
categories: [Infra]
tags: [outbox-pattern, cdc, kafka, mysql, jpa, debezium, eventual-consistency]
toc: true
toc_sticky: true
---

## 문제: 주문 완료 후 알림도 보내야 한다

쇼핑몰에서 주문이 완료되면 두 가지 일이 일어나야 한다.

1. **MySQL에 주문 저장** (핵심 비즈니스)
2. **Kafka로 주문 완료 이벤트 발행** (알림, 재고 차감 등 후속 처리)

```java
@Transactional
public void placeOrder(OrderRequest request) {
    orderRepository.save(order);           // 1. MySQL 저장
    kafkaTemplate.send("order-events", order); // 2. Kafka 발행
}
```

이 코드의 문제점:

```
시나리오 A: MySQL 커밋 성공 → Kafka 발행 실패 → 주문은 있는데 알림이 안 감
시나리오 B: Kafka 발행 성공 → MySQL 커밋 실패 → 알림은 갔는데 주문이 없음
```

MySQL 트랜잭션은 Kafka 발행을 포함하지 못한다. **서로 다른 시스템 간에는 단일 트랜잭션이 불가능**하기 때문이다.

## 해결: Outbox 테이블 + CDC

### 핵심 아이디어

Kafka에 직접 발행하지 말고, **같은 DB 트랜잭션 안에서 Outbox 테이블에 이벤트를 저장**한다. 이후 CDC(Change Data Capture)가 Outbox 테이블의 변경을 감지하여 Kafka로 발행한다.

```
API Server                    CDC (Debezium)           Consumer
    │                              │                       │
    │ @Transactional               │                       │
    │  ├─ orders INSERT            │                       │
    │  └─ outbox_events INSERT     │                       │
    │──→ MySQL (단일 트랜잭션)       │                       │
    │                              │ binlog 감지            │
    │                              │──→ Kafka 발행          │
    │                              │                       │ 메시지 수신
    │                              │                       │──→ 알림 발송
```

**주문 저장과 이벤트 저장이 같은 트랜잭션**이므로, 둘 다 성공하거나 둘 다 실패한다.

## 구현

### 1. Outbox 테이블 설계

```sql
CREATE TABLE outbox_events (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    aggregate_type  VARCHAR(50)  NOT NULL,   -- 'ORDER', 'PAYMENT' 등
    aggregate_id    VARCHAR(100) NOT NULL,   -- 주문 ID
    event_type      VARCHAR(50)  NOT NULL,   -- 'ORDER_PLACED', 'ORDER_CANCELLED'
    routing_topic   VARCHAR(100) NOT NULL,   -- Kafka 토픽명
    payload         JSON         NOT NULL,   -- 이벤트 데이터
    status          VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
    created_at      DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    processed_at    DATETIME     NULL,
    
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);
```

핵심 필드:
- `routing_topic`: CDC가 이 값을 읽어 Kafka 토픽을 동적으로 결정
- `status`: PENDING → PROCESSED / FAILED
- `payload`: JSON으로 이벤트 데이터를 유연하게 저장

### 2. JPA 엔티티

```java
@Entity
@Table(name = "outbox_events")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OutboxEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 50)
    private String aggregateType;

    @Column(nullable = false, length = 100)
    private String aggregateId;

    @Column(nullable = false, length = 50)
    private String eventType;

    @Column(nullable = false, length = 100)
    private String routingTopic;

    @Column(nullable = false, columnDefinition = "JSON")
    private String payload;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private OutboxStatus status = OutboxStatus.PENDING;

    private LocalDateTime createdAt = LocalDateTime.now();
    private LocalDateTime processedAt;

    public static OutboxEvent create(
            String aggregateType,
            String aggregateId,
            String eventType,
            String routingTopic,
            String payload
    ) {
        OutboxEvent event = new OutboxEvent();
        event.aggregateType = aggregateType;
        event.aggregateId = aggregateId;
        event.eventType = eventType;
        event.routingTopic = routingTopic;
        event.payload = payload;
        return event;
    }
}
```

```java
public enum OutboxStatus {
    PENDING,
    PROCESSED,
    FAILED
}
```

### 3. 주문 서비스 — 같은 트랜잭션에서 Outbox 저장

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final OutboxEventRepository outboxEventRepository;
    private final ObjectMapper objectMapper;

    @Transactional
    public void placeOrder(OrderRequest request) {
        // 1. 주문 저장
        Order order = Order.create(request);
        orderRepository.save(order);

        // 2. 같은 트랜잭션에서 Outbox 이벤트 저장
        OutboxEvent event = OutboxEvent.create(
            "ORDER",
            order.getId().toString(),
            "ORDER_PLACED",
            "order-events",
            toJson(order)
        );
        outboxEventRepository.save(event);
        
        // Kafka 직접 발행 없음!
    }

    private String toJson(Order order) {
        try {
            return objectMapper.writeValueAsString(
                Map.of(
                    "orderId", order.getId(),
                    "userId", order.getUserId(),
                    "totalAmount", order.getTotalAmount(),
                    "orderedAt", order.getOrderedAt().toString()
                )
            );
        } catch (JsonProcessingException e) {
            throw new RuntimeException(e);
        }
    }
}
```

`@Transactional` 안에서 `orders` INSERT와 `outbox_events` INSERT가 **같은 MySQL 트랜잭션**으로 묶인다. 둘 중 하나라도 실패하면 전부 롤백된다.

### 4. CDC — MySQL binlog → Kafka (Debezium)

RDB에서는 **Debezium**이 가장 많이 쓰이는 CDC 도구다. MySQL의 binlog를 읽어서 테이블 변경을 Kafka로 발행한다.

```json
// Debezium Connector 설정
{
  "name": "outbox-connector",
  "config": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "mysql-host",
    "database.port": "3306",
    "database.user": "debezium",
    "database.password": "password",
    "database.server.id": "1",
    "database.include.list": "shop",
    "table.include.list": "shop.outbox_events",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.fields.additional.placement": "event_type:header",
    "transforms.outbox.route.topic.replacement": "${routedByValue}",
    "transforms.outbox.route.by.field": "routing_topic"
  }
}
```

Debezium의 `EventRouter` SMT(Single Message Transform)가 핵심이다:
- `outbox_events` 테이블의 INSERT를 감지
- `routing_topic` 필드 값을 읽어 해당 Kafka 토픽으로 라우팅
- `payload` 필드를 메시지 본문으로 발행

### CDC 없이 Polling 방식도 가능

Debezium 설정이 부담스러우면 **스케줄러로 Outbox 테이블을 폴링**하는 방식도 있다.

```java
@Component
@RequiredArgsConstructor
public class OutboxPollingPublisher {

    private final OutboxEventRepository outboxEventRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;

    @Scheduled(fixedDelay = 1000)  // 1초마다
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> events = outboxEventRepository
            .findByStatusOrderByCreatedAt(OutboxStatus.PENDING);

        for (OutboxEvent event : events) {
            try {
                kafkaTemplate.send(
                    event.getRoutingTopic(),
                    event.getAggregateId(),
                    event.getPayload()
                ).get();
                event.markProcessed();
            } catch (Exception e) {
                event.markFailed();
            }
        }
    }
}
```

| 방식 | 장점 | 단점 |
|------|------|------|
| **Debezium (CDC)** | 실시간, binlog 기반이라 DB 부하 적음 | 인프라 구성 복잡 |
| **Polling** | 구현 간단, 추가 인프라 불필요 | 지연 발생, DB 폴링 부하 |

### 5. Consumer — Kafka 메시지 수신

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OrderEventListener {

    private final NotificationService notificationService;
    private final InventoryService inventoryService;

    @KafkaListener(topics = "order-events", groupId = "notification-group")
    public void handleOrderEvent(String message) {
        OrderEvent event = parseMessage(message);

        log.info("Received order event: orderId={}", event.getOrderId());

        notificationService.sendOrderConfirmation(event);
        inventoryService.decreaseStock(event);
    }
}
```

## MongoDB vs RDB 비교

앞선 글에서 MongoDB + Change Stream으로 구현한 Outbox 패턴과 비교하면:

| 항목 | MongoDB | RDB (MySQL) |
|------|---------|-------------|
| **트랜잭션** | replica set 필요 | 기본 지원 |
| **CDC 도구** | Change Stream (내장) | Debezium (외부) |
| **변경 감지** | oplog 기반 실시간 | binlog 기반 실시간 |
| **폴링 대안** | 가능 | 가능 |
| **Outbox 저장** | Document (유연한 스키마) | 테이블 (고정 스키마) |
| **인프라 복잡도** | 낮음 (Change Stream 내장) | 높음 (Debezium 별도 운영) |

MongoDB는 Change Stream이 내장되어 별도 CDC 도구가 필요 없다는 장점이 있고, RDB는 트랜잭션이 기본 지원되어 `@Transactional`만으로 정합성을 보장할 수 있다는 장점이 있다.

## 전체 흐름 정리

```
┌─────────────────────────────────────────────────┐
│ API Server (@Transactional)                      │
│                                                   │
│   orderRepository.save(order)          ─┐         │
│   outboxEventRepository.save(event)     ├→ MySQL  │
│                                        ─┘ (1 TX)  │
└─────────────────────────────────────────────────┘
                     │
                     │ binlog
                     ▼
┌─────────────────────────────────────────────────┐
│ Debezium CDC                                      │
│   outbox_events INSERT 감지                       │
│   → routing_topic 필드로 Kafka 토픽 결정           │
│   → payload를 메시지 본문으로 발행                  │
└─────────────────────────────────────────────────┘
                     │
                     │ Kafka
                     ▼
┌─────────────────────────────────────────────────┐
│ Consumer                                          │
│   @KafkaListener(topics = "order-events")         │
│   → 알림 발송, 재고 차감 등 후속 처리               │
└─────────────────────────────────────────────────┘
```

## 핵심 정리

1. **Kafka에 직접 발행하지 않는다** — 같은 DB 트랜잭션에서 Outbox 테이블에 저장
2. **CDC가 Outbox → Kafka 브릿지 역할** — Debezium(binlog) 또는 폴링 방식
3. **at-least-once + 멱등한 처리** — 중복 수신 가능성이 있으므로 Consumer는 멱등하게 구현
4. **routing_topic 필드로 동적 라우팅** — CDC에 토픽별 분기 로직이 필요 없음
