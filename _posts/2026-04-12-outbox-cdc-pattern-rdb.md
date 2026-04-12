---
title: "Outbox + CDC 패턴 (RDB편) — JPA + MySQL에서 데이터 정합성 보장하기"
date: 2026-04-12T12:00:00+09:00
categories: [Infra]
tags: [outbox-pattern, cdc, kafka, kafka-connect, mysql, jpa, debezium, eventual-consistency]
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

### Kafka Connect — Debezium이 동작하는 플랫폼

Debezium은 단독으로 실행되는 애플리케이션이 아니다. **Kafka Connect**라는 프레임워크 위에서 동작하는 **Connector 플러그인**이다.

#### Kafka Connect란?

Kafka Connect는 외부 시스템과 Kafka 사이에서 데이터를 주고받는 **표준화된 통합 프레임워크**다.

```
┌──────────────┐     ┌─────────────────────┐     ┌──────────┐
│ MySQL        │────→│ Kafka Connect       │────→│ Kafka    │
│ PostgreSQL   │     │  ├─ Source Connector │     │          │
│ MongoDB      │     │  │  (Debezium 등)   │     │  토픽들   │
│              │     │  │                   │     │          │
│ Elasticsearch│←────│  └─ Sink Connector   │←────│          │
│ S3           │     │     (ES Sink 등)     │     │          │
└──────────────┘     └─────────────────────┘     └──────────┘
```

| 종류 | 방향 | 예시 |
|------|------|------|
| **Source Connector** | 외부 시스템 → Kafka | Debezium MySQL, Debezium PostgreSQL |
| **Sink Connector** | Kafka → 외부 시스템 | Elasticsearch Sink, S3 Sink, JDBC Sink |

Debezium은 **Source Connector**다. MySQL의 binlog를 읽어서 Kafka 토픽으로 발행한다.

#### Kafka Connect의 장점

**별도 코드 작성 없이 설정만으로 데이터 파이프라인을 구성**할 수 있다.

```bash
# Connector 등록 (REST API)
curl -X POST http://kafka-connect:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "outbox-connector",
    "config": {
      "connector.class": "io.debezium.connector.mysql.MySqlConnector",
      "database.hostname": "mysql-host",
      "database.port": "3306",
      "database.user": "debezium",
      "database.password": "password",
      "database.server.id": "1",
      "table.include.list": "shop.outbox_events",
      "transforms": "outbox",
      "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
      "transforms.outbox.route.by.field": "routing_topic"
    }
  }'

# Connector 상태 확인
curl http://kafka-connect:8083/connectors/outbox-connector/status

# Connector 목록 조회
curl http://kafka-connect:8083/connectors

# Connector 삭제
curl -X DELETE http://kafka-connect:8083/connectors/outbox-connector
```

Kafka Connect 클러스터에 REST API로 Connector를 등록/삭제/관리한다. Java 코드를 작성하거나 애플리케이션을 빌드할 필요가 없다.

#### Standalone vs Distributed 모드

| 모드 | 특징 | 용도 |
|------|------|------|
| **Standalone** | 단일 프로세스, 설정 파일 기반 | 개발/테스트 |
| **Distributed** | 여러 Worker가 클러스터 구성, REST API 기반 | 운영 환경 |

운영에서는 **Distributed 모드**를 사용한다. 여러 Worker 노드가 클러스터를 이루며, Connector의 Task를 자동으로 분배한다. Worker가 죽으면 다른 Worker가 Task를 이어받는다.

### Debezium 장애 복구 — 데이터를 놓치지 않는 원리

Debezium이 장애 상황에서도 데이터 정합성을 유지할 수 있는 이유는 **offset 관리** 덕분이다.

#### binlog offset 저장

Debezium은 MySQL binlog의 어디까지 읽었는지를 **Kafka 내부 토픽(`connect-offsets`)**에 저장한다.

```
1. binlog position (mysql-bin.000003, offset 12345) 까지 읽음
2. 해당 변경을 Kafka로 발행
3. binlog position을 connect-offsets 토픽에 커밋
```

이 구조는 Kafka Consumer의 `__consumer_offsets`와 동일한 원리다.

#### 장애 시나리오별 복구

**시나리오 1: Debezium Worker 다운**

```
1. binlog position 12345까지 처리 후 offset 커밋
2. Worker 다운
3. Worker 재시작 (또는 다른 Worker가 Task 인계)
4. connect-offsets에서 마지막 offset 조회 → 12345
5. binlog position 12345 이후부터 다시 읽기 시작
```

Kafka Consumer가 `__consumer_offsets`에서 마지막 읽은 위치를 복구하는 것과 똑같다.

**시나리오 2: Kafka 발행 성공 → offset 커밋 전 다운**

```
1. binlog 변경을 Kafka로 발행 성공
2. offset 커밋 전에 Worker 다운
3. 재시작 → 마지막 커밋된 offset부터 다시 읽기
4. 같은 binlog 변경을 다시 Kafka로 발행 (중복)
```

이 경우 **같은 메시지가 중복 발행**된다. 따라서 **at-least-once** 전달이며, Consumer 측에서 멱등한 처리가 필수적이다.

**시나리오 3: MySQL 다운 → 복구**

```
1. MySQL 다운
2. Debezium 연결 실패 → 재연결 대기 (backoff)
3. MySQL 복구
4. Debezium 재연결 → 마지막 binlog position부터 재개
```

MySQL의 binlog는 `expire_logs_days` 설정에 따라 보존된다 (기본 30일). Debezium이 다운된 동안 발생한 변경도 binlog에 남아있으므로, 복구 후 누락 없이 처리할 수 있다. 단, binlog 보존 기간을 초과하면 **스냅샷부터 다시 수행**해야 한다.

#### 정리: 장애 복구 메커니즘

```
MySQL binlog                 Kafka (connect-offsets)
  │                              │
  │ binlog는 디스크에 보존         │ offset은 Kafka 토픽에 보존
  │ (expire_logs_days)           │ (무기한)
  │                              │
  └──── Debezium 재시작 시 ───────┘
         두 위치를 비교하여
         놓친 부분부터 재처리
```

| 장애 상황 | 데이터 유실 | 중복 발생 | 복구 방식 |
|-----------|:---------:|:---------:|----------|
| Debezium 다운 → 재시작 | X | 가능 | connect-offsets에서 위치 복구 |
| Kafka 발행 후 offset 커밋 전 다운 | X | O | 같은 메시지 재발행 (at-least-once) |
| MySQL 다운 → 복구 | X | X | binlog에서 이어서 읽기 |
| binlog 만료 (장기 다운) | 가능 | X | 스냅샷 재수행 필요 |

**핵심: Debezium은 "유실보다 중복이 낫다"는 at-least-once 전략**을 따른다. 중복은 Consumer의 멱등성으로 해결하고, 유실은 원천적으로 방지한다.

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
