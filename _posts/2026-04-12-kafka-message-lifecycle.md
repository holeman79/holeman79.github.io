---
title: "Kafka 메시지 생명주기 — 발행부터 삭제까지"
date: 2026-04-12
categories: [Kafka]
tags: [kafka, message-queue, consumer-group, offset, spring-kafka]
toc: true
toc_sticky: true
---

## 개요

Kafka를 처음 접하면 "메시지를 누가 읽으면 사라지는 건가?"라는 의문이 든다. RabbitMQ처럼 consumer가 ack하면 큐에서 삭제되는 방식에 익숙하다면 더욱 그렇다. Kafka는 근본적으로 다른 구조를 가지고 있다.

## 1. 메시지 발행 (Producer → Broker)

Producer가 특정 토픽에 메시지를 보내면, 브로커는 해당 토픽의 파티션에 **append-only 로그**로 기록한다.

```
토픽: order-events (파티션 1개 기준)

offset:  [0] [1] [2] [3] [4] [5]
메시지:   A   B   C   D   E   F
                                 ← 새 메시지는 항상 끝에 추가
```

각 메시지에는 **offset**이라는 순차 번호가 붙는다. 이 offset이 Kafka의 핵심이다.

## 2. 메시지 수신 (Consumer Group)

Consumer는 반드시 **Consumer Group**에 속한다. `@KafkaListener`의 `groupId`가 이것이다.

```java
@KafkaListener(topics = "order-events", groupId = "order-processing-group")
public void handle(OrderEvent event) {
    // 처리 로직
}
```

### 같은 Group 내 Consumer

같은 groupId를 가진 Consumer가 여러 개면, 파티션이 **분배**된다. 하나의 파티션은 그룹 내 하나의 Consumer만 읽을 수 있다.

```
토픽: order-events (파티션 3개)

order-processing-group:
  Consumer A → partition-0
  Consumer B → partition-1
  Consumer C → partition-2
```

이 구조 덕분에 같은 그룹 내에서는 **메시지 중복 수신 없이 병렬 처리**가 가능하다.

### 다른 Group 간 Consumer

groupId가 다르면 **각 그룹이 독립적으로 모든 메시지를 수신**한다.

```
토픽: order-events
  메시지: [0] [1] [2] [3]

  order-processing-group  → [0] [1] [2] [3]  (전부 수신)
  order-analytics-group   → [0] [1] [2] [3]  (전부 수신)
```

한 그룹이 메시지를 읽어도 다른 그룹에 영향을 주지 않는다.

## 3. Offset 관리 — 어디까지 읽었는지 기억하는 원리

Kafka의 핵심 질문: **Consumer가 죽었다 살아나면 어디서부터 읽지?**

답은 `__consumer_offsets`라는 **브로커 내부 토픽**에 있다.

### 정상 흐름

```
1. Consumer가 메시지 [0][1][2] 처리
2. offset=3을 브로커에 커밋
3. 브로커의 __consumer_offsets에 저장:
   { group: "order-processing-group",
     partition: order-events-0,
     offset: 3 }
```

### Consumer 재시작 시

```
4. Consumer 다운 (서버 장애)
5. Consumer 재시작 → 같은 groupId로 브로커에 접속
6. 브로커에게 "내 offset 어디야?" 조회
7. 브로커: "offset 3이야"
8. 메시지 [3]부터 수신 재개
```

offset은 Consumer 메모리가 아니라 **브로커에 저장**되므로, Consumer가 죽어도 유실되지 않는다.

### 새로운 Consumer Group 등록

새로운 groupId로 처음 접속하면 브로커에 커밋된 offset이 없다. 이때 `auto.offset.reset` 설정에 따라 시작 위치가 결정된다.

| 설정값 | 동작 |
|--------|------|
| `latest` (기본값) | 접속 시점 이후 새 메시지만 수신 |
| `earliest` | 토픽에 남아있는 가장 오래된 메시지부터 수신 |

별도 등록 절차 없이 **처음 접속하는 순간 자동으로 그룹이 생성**된다.

## 4. 메시지 삭제 — Retention 정책

RabbitMQ와 가장 큰 차이점이다. **Kafka는 consumer의 수신 여부와 무관하게 메시지를 삭제하지 않는다.**

메시지 삭제는 오직 **브로커의 retention 정책**에 의해서만 발생한다.

```
# 시간 기반 (기본값: 7일)
log.retention.hours=168

# 용량 기반
log.retention.bytes=1073741824  # 1GB
```

7일이 지나면, 모든 Consumer Group이 읽었든 아무도 안 읽었든 삭제된다.

```
Day 1: [0] [1] [2] [3] [4] [5]    ← 전부 존재
Day 8: [X] [X] [X] [3] [4] [5]    ← 7일 지난 메시지 삭제
```

## 5. Offset 커밋 타이밍과 메시지 전달 보장

offset을 언제 커밋하느냐에 따라 메시지 전달 보장 수준이 달라진다.

### at-most-once (최대 한 번)

```
1. 메시지 수신
2. offset 커밋        ← 먼저 커밋
3. 비즈니스 로직 처리   ← 여기서 실패하면 메시지 유실
```

### at-least-once (최소 한 번)

```
1. 메시지 수신
2. 비즈니스 로직 처리   ← 먼저 처리
3. offset 커밋        ← 여기서 실패하면 재시작 시 같은 메시지 재수신
```

Spring Kafka의 `@KafkaListener`는 기본적으로 **at-least-once** 방식이다. 리스너 메서드가 정상 리턴한 후에 offset을 커밋한다. 메서드에서 예외가 발생하면 offset을 커밋하지 않으므로 재시작 시 같은 메시지를 다시 수신한다.

따라서 **멱등한(idempotent) 처리**가 중요하다. 같은 메시지를 여러 번 처리해도 결과가 동일해야 한다.

```sql
-- 멱등한 연산: 여러 번 실행해도 결과 동일
UPDATE certificates SET hidden = true WHERE id = 'cert-001'

-- 멱등하지 않은 연산: 중복 실행 시 데이터 오류
UPDATE accounts SET balance = balance - 1000 WHERE id = 'acc-001'
```

## 6. Kafka vs RabbitMQ 비교

| 항목 | Kafka | RabbitMQ |
|------|-------|----------|
| 메시지 저장 | 로그 기반 (append-only) | 큐 기반 |
| 메시지 삭제 | retention 정책 (시간/용량) | consumer ack 시 삭제 |
| 같은 메시지 재소비 | offset을 되감으면 가능 | 불가능 (이미 삭제됨) |
| 다중 Consumer | Consumer Group별 독립 소비 | 큐 하나에 여러 consumer가 경쟁 |
| 순서 보장 | 파티션 내 보장 | 큐 내 보장 |

## 정리

1. **메시지는 consumer가 읽어도 삭제되지 않는다** — retention 정책에 의해서만 삭제
2. **offset은 브로커가 관리한다** — consumer가 죽어도 재시작 시 이어서 읽기 가능
3. **Consumer Group은 자동 생성된다** — 새 groupId로 접속하면 브로커가 알아서 등록
4. **같은 토픽을 여러 Group이 독립적으로 소비** — 서로 영향 없음
5. **at-least-once + 멱등한 처리**가 실무에서 가장 많이 쓰이는 패턴
