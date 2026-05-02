---
title: "순수 도메인 객체 모델, 무엇이 좋고 무엇을 잃는가"
date: 2026-05-02T10:00:00+09:00
categories: [Clean Code/Architecture]
tags: [hexagonal-architecture, ports-and-adapters, ddd, domain-model, kotlin, clean-architecture]
---

## TL;DR

- **순수 도메인 객체 모델**이란 도메인 레이어가 프레임워크(Spring, JPA 등)에 의존하지 않고, 비즈니스 규칙과 행위를 객체 스스로 가지는 설계다.
- 헥사고날(포트 앤 어댑터) 아키텍처와 짝을 이뤄, 도메인은 **포트(인터페이스)** 만 알고 **어댑터(인프라 구현체)** 는 바깥에 둔다.
- **장점**: 테스트 속도/단순성, 비즈니스 규칙의 응집, 인프라 교체 용이성, 도메인 언어의 명확화.
- **단점**: 보일러플레이트(도메인 ↔ 영속 모델 매핑), ORM 편의기능 포기, 학습 곡선, 작은 프로젝트엔 과한 구조.
- 결론: **도메인 규칙이 진짜로 복잡하고 오래 살아남을 코드**라면 투자할 가치가 충분하다. CRUD 위주의 단순 앱이라면 구조 비용이 더 클 수 있다.

## 순수 도메인 객체 모델이란

흔히 "Anemic Domain Model(빈혈 도메인 모델)"의 반대말로 불린다.

```
[빈혈 모델]                       [순수 도메인 모델]
- Entity는 getter/setter 덩어리   - Entity가 행위를 가짐
- Service가 모든 로직을 가짐       - Service는 흐름 조율만
- 프레임워크 어노테이션 가득       - 프레임워크와 분리된 POJO/POKO
- "데이터 + 로직" 분리             - "데이터 + 로직" 한 객체에 응집
```

순수 도메인 객체 모델은 단순히 행위를 가진 객체에 그치지 않는다. 보통 다음 원칙들과 함께 간다.

- **원시값 포장(Value Object)**: `String email` 대신 `Email` 클래스
- **일급 컬렉션**: `List<Comment>` 대신 `Comments` 클래스
- **포트(Port)와 어댑터(Adapter)**: 도메인은 인터페이스만 알고, 구현은 인프라 모듈에서
- **디미터 법칙**: `a.b.c.doSomething()` 대신, 협력 객체에 메시지를 보냄

## 우리 프로젝트의 구조

지금 운영 중인 백엔드는 다음과 같이 모듈을 쪼개 두었다.

```
meet-again/
├── boot/
│   ├── ma-boot-web/          # REST API (Controller, Security)
│   └── ma-boot-batch/        # 배치 잡
├── domain/
│   └── ma-domain-core/       # 프레임워크-독립 비즈니스 로직
│       ├── model/            # 도메인 객체 (행위 포함)
│       └── port/             # 외부 의존성 인터페이스
├── infrastructure/
│   ├── storage/
│   │   ├── ma-db-core/       # MariaDB + Exposed
│   │   └── ma-redis-core/    # Redis
│   └── support/
│       ├── ma-jwt-core/      # JWT
│       ├── ma-crypto-core/   # 비밀번호 암호화
│       └── ma-sms-sender/    # SMS (CoolSMS)
└── config/
```

핵심 규칙은 단 하나다.

> **`ma-domain-core`는 어떤 인프라/프레임워크 모듈에도 의존하지 않는다.**

이 규칙이 깨지면 모듈 의존 그래프가 그대로 빌드 에러로 알려준다. 즉, 컴파일러가 아키텍처를 강제해준다.

### 도메인이 인프라를 호출하는 방법: 포트

예를 들어 SMS 인증코드 발송 로직은 도메인의 관심사다. 하지만 실제 SMS 게이트웨이(CoolSMS) 호출은 인프라의 일이다. 이를 다음처럼 분리한다.

```kotlin
// domain/ma-domain-core - 도메인이 정의하는 인터페이스
interface SmsSender {
    fun send(phoneNumber: String, code: VerificationCode)
}
```

```kotlin
// infrastructure/support/ma-sms-sender - 인프라가 구현
class CoolSmsSender(...) : SmsSender {
    override fun send(phoneNumber: String, code: VerificationCode) { ... }
}
```

도메인은 `SmsSender`라는 인터페이스의 존재만 알 뿐, CoolSMS인지 Twilio인지, 심지어 로컬 환경에서는 그냥 로그만 찍는지 모른다. 실제 prod 환경 분리도 이 포트 덕분에 어댑터만 바꿔 끼우면 된다.

### 행위를 가진 도메인 객체

원시값을 그대로 두지 않는다. 의미 있는 값은 객체로 감싸 검증과 행위를 응집시킨다.

```kotlin
data class Email(val value: String) {
    init {
        if (value.isBlank()) throw InvalidValueException(...)
        if (!EMAIL_REGEX.matches(value)) throw InvalidValueException(...)
    }
}

data class VerificationCode(val value: String) {
    init {
        if (!PATTERN.matches(value)) throw InvalidValueException(...)
    }

    companion object {
        fun random(): VerificationCode = ...   // 6자리 코드 생성
    }
}
```

`Email`이라는 타입이 살아있는 한, 시스템 어디에서도 빈 문자열이나 형식 오류 이메일이 흘러다닐 수 없다. **타입 시스템이 곧 도메인 제약**이 된다.

일급 컬렉션도 마찬가지다.

```kotlin
class Comments(val data: List<Comment>) {
    fun extractAuthorEmails(): Set<Email> = data.map { it.authorEmail }.toSet()

    fun groupByRootComment(members: Members): List<CommentWithAuthor> { ... }
}
```

서비스 레이어 곳곳에서 `comments.filter { ... }.map { ... }`이 흩어지지 않고, 컬렉션을 다루는 의도가 `Comments`라는 도메인 개념 안에 모인다.

## 장점

### 1. 테스트가 가볍고 빠르다

도메인이 Spring을 모르므로, 도메인 테스트에 `@SpringBootTest`가 필요 없다. 그냥 객체를 new 해서 메서드를 호출한다. 수십 개 테스트가 1초 안에 끝난다.

```kotlin
class EmailTest : StringSpec({
    "유효하지 않은 형식이면 예외" {
        shouldThrow<InvalidValueException> { Email("not-email") }
    }
})
```

Spring 컨텍스트 띄우는 데만 5초 걸리는 통합 테스트와 비교하면, 도메인 단위 테스트의 피드백 속도는 차원이 다르다.

### 2. 비즈니스 규칙이 한 곳에 응집된다

`Email`의 형식 검증, `VerificationCode`의 6자리 제약, `Comments`의 그룹핑 로직이 전부 자기 클래스 안에 있다. "이 검증이 어디 있더라" 하고 Service, Validator, Util을 헤맬 필요가 없다. 도메인을 읽으면 비즈니스가 보인다.

### 3. 인프라 교체가 진짜로 쉽다

포트로 분리되어 있다면, MariaDB → PostgreSQL, CoolSMS → Twilio, JWT → 세션 같은 교체가 **도메인 코드 한 줄 안 건드리고** 어댑터 모듈만 바꾸면 끝난다. "이론적으로" 가능한 게 아니라, 실제로 prod/local 환경별 SMS 어댑터를 갈아끼우며 검증된다.

### 4. 도메인 언어가 코드에 드러난다

`String email` 대신 `Email`, `Int code` 대신 `VerificationCode`, `List<Comment>` 대신 `Comments`. 메서드 시그니처만 봐도 무엇을 다루는지 명확하다. 코드가 곧 도메인 사전이 된다.

### 5. 의존성 방향이 강제된다

멀티 모듈로 도메인을 분리해두면, **도메인이 인프라를 import 하는 순간 컴파일이 깨진다**. 사람의 규율이 아니라 빌드 시스템이 아키텍처를 지켜준다. PR 리뷰에서 "이거 레이어 위반인데요" 같은 지적이 사라진다.

### 6. 멀티 보트(Boot)에서 도메인 재활용

`ma-boot-web`과 `ma-boot-batch`가 동일한 `ma-domain-core`를 공유한다. 같은 비즈니스 규칙을 API와 배치 잡에서 일관되게 쓸 수 있다. 코드 중복도, 동작 불일치도 없다.

## 단점

### 1. 도메인 ↔ 영속 모델 매핑 보일러플레이트

도메인 `Post`와 Exposed의 `PostTable` 간에 변환 코드를 직접 짜야 한다.

```kotlin
fun ResultRow.toPost(): Post = Post(
    id = this[PostTable.id].value,
    authorEmail = Email(this[PostTable.authorEmail]),
    ...
)
```

JPA처럼 엔티티 자체를 영속화하는 모델에서는 없는 비용이다. 필드 하나 추가될 때마다 매퍼도 같이 손봐야 한다.

### 2. ORM의 편의기능을 포기한다

JPA를 쓰면 누릴 수 있는 dirty checking, lazy loading, cascade 같은 기능이 사라진다. 우리는 Exposed DSL로 명시적 SQL을 짜는 쪽을 선택했지만, 그만큼 "조회하고 → 도메인으로 변환하고 → 수정하고 → 다시 SQL 만들어 update" 흐름을 직접 관리해야 한다.

### 3. 학습 곡선과 팀 합의 비용

빈혈 모델 + Service 패턴에 익숙한 팀에게 "Service에서 if 한 줄 쓰지 말고 도메인 메서드로 옮겨라"는 요구는 처음에 거부감을 부른다. 코드 리뷰에서 매번 "이건 객체 안으로 들어가야 한다"는 피드백이 반복되고, 합의된 가이드(스킬 문서, CLAUDE.md 등)가 없으면 일관성이 깨진다.

### 4. 작은 프로젝트에는 과한 구조

도메인 규칙이 거의 없는 CRUD 앱이라면, 모듈 분리/포트/어댑터/매퍼를 다 두는 비용이 효익을 넘어선다. `Email` 같은 VO 하나 만드는 데 검증 + 테스트 + import 변경이 수반되는데, 그 검증이 한 곳에서만 쓰인다면 그냥 `String`이 낫다.

### 5. 트랜잭션 경계가 흐려질 위험

`@Transactional`은 보통 인프라(또는 application service)에 붙는다. 도메인 메서드 여러 개를 조합하는 흐름에서 "어디서부터 어디까지가 한 트랜잭션인가"를 application service가 명확히 잡지 않으면, 도메인의 순수성을 지키려다 트랜잭션이 헐거워질 수 있다.

### 6. 빌드 시간과 모듈 간 의존 관리

모듈을 잘게 쪼갤수록 Gradle 의존 그래프가 복잡해지고, 풀 빌드 시간이 길어진다. IntelliJ가 모듈 간 변경을 인덱싱하는 데도 시간이 더 든다. 모듈을 너무 잘게 쪼개기보다는, 의미 있는 경계 단위로만 분리하는 판단이 필요하다.

### 7. 매핑 비용(런타임)

도메인 ↔ 영속 ↔ DTO 변환이 곳곳에서 일어난다. 대규모 페이징 응답처럼 객체를 수만 개 만드는 경로에서는 측정 가능한 오버헤드가 생긴다. 핫패스에서는 의도적으로 매핑을 줄이거나 직접 `ResultRow → Response`로 가는 별도 경로를 두기도 한다.

## 언제 쓰고, 언제 피할까

| 적합한 상황 | 부적합한 상황 |
|---|---|
| 도메인 규칙이 복잡하다 | 단순 CRUD가 대부분이다 |
| 코드가 오래 살아남을 예정이다 | 단기 프로토타입이다 |
| 인프라 교체 가능성이 있다 | 인프라가 완전히 고정이다 |
| 여러 진입점(API, 배치, 이벤트)에서 같은 도메인을 쓴다 | 진입점이 단 하나의 API뿐이다 |
| 팀이 OOP/DDD에 학습 의지가 있다 | 빠른 출시가 최우선이다 |

## 우리 프로젝트에서 체감한 것

- **장점이 압도적이었던 부분**: SMS 어댑터를 prod/local로 분리하던 작업. 도메인 코드는 한 줄도 안 건드리고 어댑터만 추가하니 끝났다. 또 `Email`, `VerificationCode` 같은 VO 덕에 컨트롤러 ↔ 서비스 ↔ 리포지토리 어디에서도 형식 오류가 흘러다니지 않는다.
- **불편했던 부분**: 작은 필드 하나 추가할 때 도메인, 영속 매퍼, Request/Response DTO 세 군데를 손대야 한다. 그래도 이 비용은 "버그가 어디서 들어왔는지 한눈에 보인다"는 효익으로 대체로 회수된다.
- **가장 큰 교훈**: **객체에 행위를 부여하는 습관**이 가장 핵심이다. 모듈을 아무리 잘 쪼개도 도메인이 빈혈이면 결국 Service 비대화로 끝난다. 반대로, 단일 모듈이라도 도메인 객체가 행위를 가지면 코드의 응집도는 충분히 높일 수 있다.

## 마치며

순수 도메인 객체 모델은 "유행하는 좋은 패턴"이라서 쓰는 게 아니다. **도메인 규칙이 자주 바뀌고 오래 살아남을 코드**에서, 비즈니스 의도를 코드로 보존하기 위한 투자다.

- 빈혈 모델은 처음 1년이 빠르고, 3년 차부터 느려진다.
- 순수 도메인 모델은 처음 6개월이 느리고, 그 뒤로는 변경 비용이 안정적이다.

지금 만드는 코드가 어느 쪽인지 솔직하게 판단하고, 둘 사이에서 의식적으로 선택하는 것이 핵심이다. "도메인을 잘 모델링하면 다 좋아진다"는 막연한 믿음보다는, **무엇을 얻기 위해 무엇을 포기하는가**를 명확히 하는 편이 낫다.
