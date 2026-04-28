---
title: "Outbox + CDC 환경에서 CQRS 분리되어 있을 때 sync 문제"
date: 2026-04-28T14:00:00+09:00
categories: [Infra]
tags: [outbox-pattern, cdc, cqrs, eventual-consistency, kafka]
---

## TL;DR

- Outbox + CDC + CQRS 구조에서는 command가 outbox 저장만 하고 끝나면, read DB(ES) 갱신은 subscriber가 처리한 뒤에야 일어난다.
- 사용자는 API 응답 직후 결과가 보이길 기대하는데, 실제 read DB 반영까지 **1~2초의 지연 창**이 생긴다.
- 해결: command 시점에 **read DB도 동기로 직접 갱신**하고, subscriber에서 같은 작업이 다시 돌아도 안전하도록 **모든 cascade 단계를 멱등하게** 만든다.

## 우리가 쓰던 구조

운영 중인 ASM 시스템은 다음과 같은 분산 구조를 갖고 있다.

```
[사용자 요청]
   ↓
[Web 앱 (command)]
   ├─ MongoDB outbox collection에 이벤트 저장 (트랜잭션)
   └─ 200 응답
        ↓
[CDC 앱]
   └─ MongoDB Change Stream으로 outbox insert 감지
   └─ Kafka로 publish
        ↓
[Subscriber 앱]
   └─ Kafka 메시지 수신
   └─ ES (read DB) 갱신 / cascade 처리
```

- **Write side(command)**: MongoDB에 outbox 이벤트 저장
- **Read side(query)**: Elasticsearch에서 검색/조회
- 두 store 사이를 outbox + CDC + Kafka가 비동기로 동기화

이 구조의 장점은 명확하다. 두 데이터소스 간 단일 트랜잭션이 불가능한 환경에서, outbox가 단일 진실 공급원(SoT) 역할을 하면서 결국 일관성에 도달한다.

## 그런데 사용자 시점에서 문제가 생겼다

운영 중에 이런 제보가 들어왔다.

> "Seed를 삭제하면 화면에서 안 사라지고 한참 후에야 사라진다."

코드를 따라가 보니 이런 흐름이었다.

```kotlin
@Transactional
fun deleteSeeds(seedIds: List<String>, customerId: String, username: String) {
    val targetAssetIds = ...
    val candidateCertificateIds = ...
    outboxRepository.save(
        OutboxEvent.create(
            eventType = OutboxEventType.SEED_DELETED,
            payload = SeedStateChangePayload.create(...)
        )
    )
    // ← 여기서 끝. ES Seed doc은 그대로 남아있다.
}
```

API는 **outbox 저장만 하고** 즉시 200을 응답한다. 실제 ES에서 Seed 문서가 사라지는 시점은 subscriber가 메시지를 받아 cascade를 다 돌린 후다.

운영 로그로 측정해보니 가벼운 케이스는 ~500ms, Asset 7개와 Cert 1개를 cascade로 정리하는 무거운 케이스는 ~2.1초가 걸렸다. 사용자 입장에서 API는 분명 성공했는데 새로고침해도 그대로 보이니 "삭제가 안 됐나?" 싶어 다시 누르고, 같은 outbox 이벤트가 5번씩 쌓이는 일도 있었다.

## 무엇이 문제였는가

CQRS 패턴 자체의 본질적인 트레이드오프다. **Write side와 Read side가 분리되어 있으면 둘 사이의 sync에 항상 lag이 존재한다.** Outbox + CDC는 정합성은 잘 보장해주지만 "사용자가 보는 데이터가 즉시 갱신되어야 한다"는 UX 기대를 만족시키진 못한다.

생각해본 접근법은 두 가지였다.

### 접근법 1 — Soft delete 플래그

Read DB에 `deletionStatus` 같은 플래그 컬럼을 두고, command에서 동기로 그 플래그를 set한다. 검색 쿼리는 해당 플래그를 필터링으로 제외한다. 실제 물리 삭제는 subscriber가 비동기로 처리한다.

**장점**: 사용자에게 즉시 사라진 것처럼 보임, 추가 작업이 단순함.
**단점**: 모든 검색/리스트 쿼리에 `deletionStatus != DELETING` 필터를 추가해야 함. subscriber 영구 실패 시 "삭제 중" 상태로 박혀버릴 위험.

### 접근법 2 — Sync command + Idempotent async cascade

Command 시점에 **read DB를 직접 동기로 갱신**한다. Subscriber는 평소대로 비동기 cascade를 돌리되, 모든 단계가 **멱등**하게 동작하도록 만든다.

**장점**: 스키마 변경 없음, 검색 쿼리도 그대로. 사용자 시점에서 즉시 반영.
**단점**: command 안에 read DB I/O가 추가되어 응답 시간이 약간 길어짐. 멱등성 검증 비용이 들어감.

우리는 접근법 2로 갔다. 이미 cascade 코드 대부분이 실질적으로 멱등하게 짜여 있었고, 스키마와 검색 코드를 건드리지 않는 게 매력적이었다.

## 적용한 구조

```kotlin
@Transactional
fun deleteSeeds(seedIds: List<String>, customerId: String, username: String) {
    if (seedIds.isEmpty()) return

    val targetAssetIds = ...
    val candidateCertificateIds = ...

    // 1) outbox 저장 (Mongo 트랜잭션 안)
    outboxRepository.save(
        OutboxEvent.create(
            eventType = OutboxEventType.SEED_DELETED,
            payload = SeedStateChangePayload.create(...)
        )
    )

    // 2) ES Seed doc 즉시 삭제
    seedRepository.deleteSeeds(seedIds, listOf(customerId))
}
```

순서가 중요하다. **outbox 저장이 먼저**다.

- outbox 저장 실패(트랜잭션 rollback) → ES 삭제도 실행 안 됨 → 일관된 상태 유지.
- outbox 저장 성공 → ES 삭제 실패 → outbox 이벤트는 이미 기록됐으니 subscriber가 cascade에서 결국 ES Seed를 지움 (eventual consistency).

반대 순서로 하면 ES만 삭제되고 outbox 저장이 실패했을 때 cascade가 영영 실행되지 않아 Asset/Cert 등이 orphan으로 남는다.

## Cascade는 정말 멱등한가?

Sync update를 추가했으니 subscriber는 같은 작업을 한 번 더 돌게 된다. 모든 단계가 멱등인지 점검했다.

| 단계 | 동작 | 멱등성 |
|---|---|---|
| `removeSeedIdsAndDeleteIfEmpty` | Asset/Domain의 seedIds 배열에서 제거, 빈 doc 삭제 | OK — 이미 제거된 배열에서 또 제거해도 변화 없음 |
| `deleteBySeedIds` (AssetCert) | seedId로 ES deleteByQuery | OK — `Conflicts.Proceed`, 매치 0건도 success |
| `deleteCertificatesFullyOwnedByRemovedAssets` | payload 스냅샷 기준으로 잔존 AssetCert 재평가 | OK — 이미 정리된 후에도 재계산 결과 동일 |
| `removeAffectedAssetsByAssetIds` (Vulnerability) | assetId 배열에서 제거 | OK |
| `deleteForRemovedAssets` (RiskFactor) | assetId 기준 RF 삭제 | OK — 이미 삭제된 docs는 매치 0건 |
| `deleteSeeds` (ES) | deleteByQuery | OK — 이미 사라진 doc는 no-op |

핵심은 **모든 작업이 "현재 상태를 목표 상태로 만든다"는 멱등 형태**로 짜여 있다는 점이다. "X를 +1 한다" 같은 delta 연산이 아니라 "X를 N으로 set한다" 또는 "X를 삭제한다" 형태.

자산 숨김/복원도 같은 패턴으로 처리했다.

```kotlin
@Transactional
fun hiddenAssets(assetIds: List<String>, customerId: String, username: String) {
    if (assetIds.isEmpty()) return
    outboxRepository.save(...)
    assetRepository.updateAssetHidden(assetIds, customerId, username, true)
}
```

`updateAssetHidden`은 `hidden` boolean을 set하는 ES updateByQuery라 두 번 돌려도 결과가 같다.

## 남는 트레이드오프

이 방식에도 그림자가 있다.

### 1. DLT 영구 실패 시 cascade 일부 누락

Sync 단계는 이미 적용됐는데(예: Seed 삭제), 비동기 cascade가 영구히 실패하면 자식 Asset/Domain의 seedId 참조나 Cert/RF/Note 정리가 안 된 채 남는다. 이건 sync 추가 전에도 마찬가지였다(다만 그땐 Seed 자체도 안 지워져서 사용자가 "안 됐다"고 인지할 수 있었음). 이제는 **사용자에겐 끝난 것처럼 보이는데 백그라운드 정리가 묵묵히 실패하는** 시나리오가 가능해졌다. **DLT 모니터링은 필수**.

### 2. 빠른 toggle 시 짧은 regression 창

```
T0: hidden=true 동기 적용
T1: hidden=false 동기 적용 (사용자가 빨리 토글)
T2: subscriber가 T0 이벤트 처리 → hidden=true 재적용
T3: subscriber가 T1 이벤트 처리 → hidden=false 재적용
```

T2~T3 사이에 잠깐 hidden=true로 돌아간다. 단, Kafka가 customerId를 key로 같은 파티션에 모으므로 같은 컨슈머 스레드에서 순차 처리되어 창이 밀리초 수준이다. 일반적인 사용자 행동에서는 거의 보이지 않는다.

### 3. Audit 필드의 시점 어긋남

`updateAssetHidden`이 `hiddenAt`도 set하는 경우, sync 시점과 subscriber 시점에 두 번 set되면서 마지막 시점으로 덮어써진다. 사용자가 클릭한 시점이 아니라 subscriber가 재처리한 시점이 기록된다. 정확도가 중요한 audit이라면 별도 audit log를 두는 게 낫다.

## 정리

CQRS + Outbox + CDC는 분산 데이터 정합성에는 강력하지만, 사용자 시점의 즉각성을 그냥은 보장하지 못한다. 구조를 그대로 두고 UX를 개선하려면:

1. **Command가 read DB도 직접 갱신**한다 (sync update).
2. **Subscriber의 모든 단계를 멱등하게** 만든다 ("set" 또는 "delete" 형태로 짠다).
3. **Outbox 저장이 먼저, read DB 갱신이 그 뒤** 순서로 둔다 — read DB만 갱신되고 outbox가 빠지는 사고를 방지.
4. **DLT 모니터링과 백필 잡(reconciliation)을 운영에 포함**시킨다 — sync 적용 후엔 사용자가 cascade 실패를 체감하지 못하므로 모니터링이 더 중요해진다.

이번 작업에서 가장 큰 교훈은, "멱등성이 곧 안전성"이라는 점이었다. cascade가 멱등하기만 하면 sync update를 자유롭게 추가해서 UX를 개선할 수 있다. 반대로 멱등하지 않은 코드(특히 delta 연산)가 끼어 있으면 이 패턴은 즉시 망가진다. 새로운 cascade 단계를 추가할 때마다 "이거 두 번 돌아도 괜찮나?"를 가장 먼저 묻는 습관을 들였다.
