---
title: "1:N에서 '1'을 페이징하며 'N'을 검색하기 — JOIN 행 증폭과 EXISTS 서브쿼리"
date: 2026-06-06 22:54:00 +0900
categories: [Database, RDB]
tags: [rdb, sql, paging, pagination, exists, subquery, semi-join, querydsl, kotlin, jpa, spring-boot]
toc: true
toc_sticky: true
---

## 개요

실무에서 정말 자주 만나는 요구가 있다.

> "**주문(Order)** 목록을 페이지네이션해서 보여주는데, **주문 항목(OrderItem)** 의 조건으로 검색도 되게 해줘. 예를 들어 '전자제품을 포함한 주문'만 최신순으로."

페이징하는 단위는 **부모(1쪽) `Order`** 인데, 검색 조건은 **자식(N쪽) `OrderItem`** 에 걸린다. 별 생각 없이 `JOIN`으로 풀면 **부모 행이 자식 매칭 수만큼 중복**되면서 `LIMIT`/`OFFSET`/`COUNT`가 전부 어긋난다. 같은 주문이 두 페이지에 걸쳐 나오거나, 전체 개수가 부풀려진다.

이 글은 그 문제의 정체를 SQL 레벨에서 보여주고, **`EXISTS` 서브쿼리(semi-join)** 로 깔끔하게 푸는 방법을 정리한다. 원리는 RDB 공통이라 raw SQL → JPQL → QueryDSL(Kotlin) 순으로 layering 해서 보인다.

---

## 1. 도메인과 요구사항

전형적인 쇼핑몰 1:N이다. 주문 하나에 주문 항목이 여러 개 달린다.

```kotlin
@Entity
@Table(name = "orders")
class Order(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    val ordererName: String,
    val orderedAt: LocalDateTime,

    @Enumerated(EnumType.STRING)
    val status: OrderStatus,

    @OneToMany(mappedBy = "order", cascade = [CascadeType.ALL], orphanRemoval = true)
    val orderItems: MutableList<OrderItem> = mutableListOf(),
)

@Entity
@Table(name = "order_item")
class OrderItem(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    val order: Order,

    val productName: String,
    val category: String,   // 예: "전자제품", "도서"
    val quantity: Int,
    val price: Long,
)
```

요구사항을 한 문장으로:

> **`category = '전자제품'` 인 항목을 하나라도 포함한 주문**을, 주문 단위로 `orderedAt` 순으로 페이징해서 반환한다.

데이터는 다음과 같다고 하자. (`pageSize = 2`, `orderedAt` 오름차순)

| order_id | ordered_at | '전자제품' 항목 |
|---|---|---|
| 1 | 2026-06-01 | 노트북, 마우스, 키보드 (3개) |
| 2 | 2026-06-02 | 모니터 (1개) |

기대하는 정답은 명확하다. **주문 2건** — `[Order#1, Order#2]`, `totalElements = 2`.

---

## 2. 나이브한 접근 — JOIN, 그리고 행 증폭

가장 먼저 떠오르는 건 자식 테이블을 조인해서 `WHERE` 로 거르는 것이다.

```sql
SELECT o.*
FROM orders o
JOIN order_item i ON i.order_id = o.id
WHERE i.category = '전자제품'
ORDER BY o.ordered_at
LIMIT 2 OFFSET 0;
```

문제는 조인 결과가 **주문이 아니라 "주문 × 매칭된 항목" 단위**라는 점이다. 위 데이터로 조인하면 중간 결과는 이렇게 된다.

| (row) | order_id | item |
|---|---|---|
| 1 | 1 | 노트북 |
| 2 | 1 | 마우스 |
| 3 | 1 | 키보드 |
| 4 | 2 | 모니터 |

**Order#1 하나가 3행으로 불어났다.** 여기에 `LIMIT 2` 를 걸면:

- **page 0** → row1, row2 → `[Order#1, Order#1]` ... 화면에 **같은 주문이 두 번**
- `COUNT(*)` → **4** → `totalElements = 4`, `totalPages = 2` (정답은 2건, 1페이지)
- **page 1** → row3, row4 → `[Order#1, Order#2]` ... Order#1이 **0페이지와 1페이지에 걸쳐 중복**, Order#2는 맨 끝에야 한 번 등장

페이징이 주문 단위가 아니라 **항목 단위로 잘리면서 완전히 망가진다.** 검색 조건이 없을 때(전체 목록)는 멀쩡하다가, N쪽 조인이 끼는 순간 터지는 게 더 골치 아프다.

---

## 3. `DISTINCT` 반창고와 그 비용

"중복 행이 문제니 `DISTINCT` 로 지우면 되지 않나?"

```sql
SELECT DISTINCT o.*
FROM orders o
JOIN order_item i ON i.order_id = o.id
WHERE i.category = '전자제품'
ORDER BY o.ordered_at
LIMIT 2 OFFSET 0;
```

증상은 가려지지만 대가가 따른다.

- **count 쿼리도 같이 바꿔야 한다.** `COUNT(*)` 가 아니라 `COUNT(DISTINCT o.id)` 여야 정확하다. 빠뜨리면 totalElements가 계속 부풀려진다.
- **정렬 컬럼 제약.** PostgreSQL 등에서 `SELECT DISTINCT` + `ORDER BY` 는 정렬 표현식이 select list에 포함돼야 한다. `o.*` 라 보통은 통과하지만, 자식 컬럼으로 정렬하려는 순간 막힌다.
- **DISTINCT 자체가 정렬/해시 비용.** 불어난 행을 다 만든 뒤 중복을 제거하므로, 자식이 많을수록 만들었다 버리는 일이 커진다.
- **페치 조인이면 더 위험하다.** `JOIN FETCH` + 페이징을 같이 쓰면 Hibernate가 이 경고를 남기고

  ```
  HHH000104: firstResult/maxResults specified with collection fetch;
             applying in memory
  ```

  **DB가 아니라 애플리케이션 메모리에서 페이징**한다. 즉 조건에 맞는 행 전체를 메모리로 끌어온 뒤 자른다. 대형 테이블에서는 그대로 사고다.

`DISTINCT` 는 "이미 증폭된 행"을 사후에 청소하는 접근이다. 애초에 **증폭을 만들지 않는** 방법이 있다.

---

## 4. 해결 — `EXISTS` 서브쿼리 (semi-join)

핵심 발상의 전환: 자식을 **결과에 끌고 오지(JOIN)** 말고, **"조건을 만족하는 자식이 존재하는가"만 묻는다(EXISTS).**

```sql
SELECT o.*
FROM orders o
WHERE EXISTS (
    SELECT 1
    FROM order_item i
    WHERE i.order_id = o.id          -- 상관 조건: 이 주문의 항목
      AND i.category = '전자제품'      -- 검색 조건
)
ORDER BY o.ordered_at
LIMIT 2 OFFSET 0;
```

이게 바로 **semi-join** 이다. 각 `orders` 행은 "매칭되는 자식이 하나라도 있나?" 라는 **불리언 판정으로 1회만 평가**된다.

- 주문 행이 **절대 증폭되지 않는다.** Order#1에 전자제품 항목이 3개든 30개든 결과는 1행.
- 따라서 `LIMIT`/`OFFSET` 이 **주문 단위로 정확**하다.
- `COUNT(*)` 도 `orders` 기준이라 **그대로 정확**하다. `DISTINCT` 불필요.
- 옵티마이저는 매칭되는 자식을 **하나 찾는 즉시 탐색을 멈춘다**(`SELECT 1` 의 사영은 무시됨). 다 훑지 않는다.

결과는 정답 그대로 `[Order#1, Order#2]`, `count = 2`.

> `EXISTS` 안의 `SELECT 1` 은 관례다. `SELECT *`, `SELECT id` 무엇을 써도 옵티마이저는 사영을 버리고 "존재 여부"만 본다. 가독성을 위해 `1` 을 쓴다.

---

## 5. JPQL / QueryDSL 구현

### 5-1. JPQL

JPA를 쓴다면 JPQL로 그대로 옮길 수 있다.

```kotlin
@Query(
    """
    SELECT o FROM Order o
    WHERE EXISTS (
        SELECT 1 FROM OrderItem i
        WHERE i.order = o AND i.category = :category
    )
    """
)
fun searchByItemCategory(category: String, pageable: Pageable): Page<Order>
```

Spring Data JPA가 `Pageable` 로 `LIMIT`/`OFFSET` 과 count 쿼리를 알아서 만들어준다. count도 `EXISTS` 가 붙은 `orders` 기준이라 정확하다.

### 5-2. QueryDSL — 명시적 `exists()`

동적 조건(검색 필터가 있을 때만 적용)이 필요하면 QueryDSL이 깔끔하다. 핵심은 `JPAExpressions` 로 상관 서브쿼리를 만들고 `.exists()` 를 붙이는 것이다.

```kotlin
import com.querydsl.jpa.JPAExpressions
import com.querydsl.core.types.dsl.BooleanExpression
import com.example.order.QOrder.order
import com.example.order.QOrderItem.orderItem

private fun containsCategory(category: String?): BooleanExpression? {
    if (category.isNullOrBlank()) return null
    return JPAExpressions
        .selectOne()
        .from(orderItem)
        .where(
            orderItem.order.eq(order)          // 상관 조건
                .and(orderItem.category.eq(category)),
        )
        .exists()
}
```

페이징 본체는 **content 쿼리와 count 쿼리를 분리**하고 `PageableExecutionUtils` 로 묶는다. (마지막 페이지 등에서 count 쿼리를 생략할 수 있게 해준다.)

```kotlin
fun search(category: String?, pageable: Pageable): Page<Order> {
    val where = containsCategory(category)   // null이면 조건 미적용

    val content = queryFactory
        .selectFrom(order)
        .where(where)
        .orderBy(order.orderedAt.asc())
        .offset(pageable.offset)
        .limit(pageable.pageSize.toLong())
        .fetch()

    val countQuery = queryFactory
        .select(order.count())
        .from(order)
        .where(where)

    return PageableExecutionUtils.getPage(content, pageable) { countQuery.fetchOne() ?: 0L }
}
```

`selectFrom(order)` 는 부모만 조회하므로 행 증폭이 없고, `order.count()` 역시 부모 기준이라 정확하다.

### 5-3. `.any()` 단축형과 그 함정

QueryDSL은 컬렉션 경로에 `.any()` 를 제공한다. 내부적으로 `exists` 상관 서브쿼리로 변환되므로 **단일 조건**이면 한 줄로 끝난다.

```kotlin
// 위 containsCategory 와 동등 — 단일 조건일 때
order.orderItems.any().category.eq(category)
```

**주의:** 자식 한 행이 **여러 조건을 동시에** 만족해야 할 때 `.any()` 를 여러 번 쓰면 의미가 달라진다.

```kotlin
// ❌ "category=전자제품 인 항목이 있고" AND "price>=100만 인 항목이 있다"
//    → 서로 '다른' 항목이어도 통과한다
order.orderItems.any().category.eq("전자제품")
    .and(order.orderItems.any().price.goe(1_000_000))

// ✅ "category=전자제품 이면서 price>=100만 인 '단일' 항목이 존재한다"
JPAExpressions.selectOne().from(orderItem)
    .where(
        orderItem.order.eq(order)
            .and(orderItem.category.eq("전자제품"))
            .and(orderItem.price.goe(1_000_000)),
    )
    .exists()
```

`.any()` 두 개는 각각 독립된 서브쿼리라 **다른 자식 행**에 매칭될 수 있다. "동일한 한 항목이 두 조건을 모두 만족"을 원하면 반드시 **하나의 `EXISTS` 안에서 `AND`** 로 묶어야 한다.

---

## 6. 심화 — N쪽이 둘 이상일 때 EXISTS의 진가

주문에 결제 수단이 여러 개(분할 결제)인 `Payment` 컬렉션이 하나 더 있다고 하자.

```kotlin
@OneToMany(mappedBy = "order")
val payments: MutableList<Payment> = mutableListOf()   // 카드/포인트/상품권 ...
```

요구가 두 개의 N쪽에 동시에 걸린다.

> "**전자제품을 포함**하고(`OrderItem`), **카드로 결제**한(`Payment`) 주문"

이걸 JOIN 두 개로 풀면 **곱집합이 폭발**한다. 주문이 전자제품 항목 `M`개 × 카드 결제 `K`건이면 한 주문이 `M × K` 행으로 불어난다.

```sql
-- ❌ 곱집합 폭발: 주문당 M×K 행
SELECT DISTINCT o.*
FROM orders o
JOIN order_item i ON i.order_id = o.id AND i.category = '전자제품'
JOIN payment    p ON p.order_id = o.id AND p.method   = 'CARD'
...
```

`EXISTS` 는 **각 조건이 독립된 서브쿼리**라 부모 행을 건드리지 않는다. 컬렉션이 몇 개로 늘든 부모는 1행이다.

```sql
-- ✅ 곱집합 없음: 각 EXISTS가 독립적으로 존재 여부만 판정
SELECT o.*
FROM orders o
WHERE EXISTS (SELECT 1 FROM order_item i WHERE i.order_id = o.id AND i.category = '전자제품')
  AND EXISTS (SELECT 1 FROM payment    p WHERE p.order_id = o.id AND p.method   = 'CARD')
ORDER BY o.ordered_at
LIMIT 2 OFFSET 0;
```

QueryDSL로도 `BooleanExpression` 을 `.and()` 로 이어 붙이기만 하면 된다.

```kotlin
val where = containsCategory("전자제품")
    ?.and(paidWith(PaymentMethod.CARD))

private fun paidWith(method: PaymentMethod): BooleanExpression =
    JPAExpressions.selectOne().from(payment)
        .where(payment.order.eq(order).and(payment.method.eq(method)))
        .exists()
```

여러 1:N 컬렉션에 걸친 검색일수록 JOIN 방식 대비 EXISTS의 이점이 커진다.

---

## 7. 다른 선택지와 비교

| 방식 | 부모 행 증폭 | count 정확성 | 페이징 | 비고 |
|---|---|---|---|---|
| JOIN (naive) | **O (곱집합)** | ✗ | 깨짐 | 그대로 쓰면 안 됨 |
| JOIN + DISTINCT | 사후 제거 | `count(distinct)` 필요 | 동작하나 비용↑·정렬 제약·**페치조인 시 메모리 페이징** | 반창고 |
| **EXISTS 서브쿼리** | **없음** | **정확** | **정확** | semi-join, 매칭 즉시 중단, 다중 N쪽에 강함 |
| `IN` (서브쿼리) | 없음 | 정확 | 정확 | 동등하게 동작. 단 `NOT IN` + `NULL` 함정, 초대형 IN 리스트 주의 |

`EXISTS` 와 `IN` 은 "존재"를 묻는 같은 계열이고 옵티마이저가 비슷하게 처리하는 경우가 많다. 다만 **부정 조건("~를 포함하지 않는 주문")** 에서는 `NOT EXISTS` 가 `NOT IN` 보다 안전하다. `NOT IN` 은 서브쿼리 결과에 `NULL` 이 섞이면 전체가 `UNKNOWN` 이 되어 **아무 행도 안 나오는** 고전적 함정이 있다.

---

## 8. 인덱스 한 가지

`EXISTS` 서브쿼리는 부모 행마다 상관 조건으로 자식을 조회한다. 그래서 자식 테이블의 인덱스가 성능을 좌우한다.

```sql
-- 상관 조건(order_id) + 검색 조건(category)을 함께 태우는 복합 인덱스
CREATE INDEX idx_order_item_order_category ON order_item (order_id, category);
```

FK라서 `order_id` 단독 인덱스는 보통 있지만, 그것만 있으면 `order_id` 로 좁힌 뒤 `category` 는 필터로 훑는다. `(order_id, category)` 복합 인덱스면 서브쿼리가 인덱스만으로 존재 여부를 즉시 판정한다.

---

## 9. 정리 / 교훈

- **1:N에서 부모(1)를 페이징하면서 자식(N) 조건으로 검색**할 때, 자식을 `JOIN` 하면 부모 행이 자식 매칭 수만큼 **증폭**되어 `LIMIT`/`OFFSET`/`COUNT` 가 모두 깨진다. 같은 부모가 여러 페이지에 중복되거나 전체 개수가 부풀려진다.
- `DISTINCT` 는 증상만 가린다. count도 `count(distinct)` 로 바꿔야 하고, 정렬 제약과 정렬·해시 비용이 붙으며, **페치 조인과 만나면 메모리 페이징**으로 번진다.
- **`EXISTS` 서브쿼리(semi-join)** 가 정공법이다. 부모 행을 "자식 존재 여부"로 1회만 판정하므로 증폭이 없고, 페이징과 count가 그대로 정확하며, 매칭을 찾는 즉시 멈춘다.
- **N쪽 조건이 둘 이상**이면 EXISTS의 이점이 결정적이다. JOIN은 곱집합으로 폭발하지만, 독립된 EXISTS는 컬렉션이 몇 개든 부모를 1행으로 유지한다.
- QueryDSL에서는 `JPAExpressions.selectOne().from(child).where(child.parent.eq(parent).and(...)).exists()` 로 짠다. `.any()` 단축형은 **단일 조건**일 때만 안전하고, **한 자식이 여러 조건을 동시에** 만족해야 하면 하나의 `EXISTS` 안에서 `AND` 로 묶어야 한다.
- 자식 테이블에 `(부모FK, 검색컬럼)` **복합 인덱스**를 걸어 상관 서브쿼리를 인덱스만으로 끝내자.

### 참고

- QueryDSL `JPAExpressions` 서브쿼리, `.exists()` / 컬렉션 경로 `.any()`
- Spring Data `PageableExecutionUtils.getPage` — count 쿼리 지연/생략
- Hibernate `HHH000104` — collection fetch + 페이징 시 in-memory paging 경고
