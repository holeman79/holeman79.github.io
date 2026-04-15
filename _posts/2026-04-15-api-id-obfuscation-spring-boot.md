---
title: "API 응답에서 DB ID를 숨기자 — Spring Boot + Hashids로 ID 난독화 구현기"
date: 2026-04-15T10:00:00+09:00
categories: [Spring]
tags: [spring-boot, hashids, id-obfuscation, security, jackson, hexagonal-architecture, kotlin]
toc: true
toc_sticky: true
---

## 왜 ID를 숨겨야 하는가

REST API를 설계할 때 가장 흔한 패턴은 리소스의 DB PK를 URL에 그대로 노출하는 것이다.

```
GET /api/members/1
GET /api/target-infos/42
```

이 방식에는 몇 가지 보안/비즈니스 리스크가 있다.

- **데이터 규모 추측**: `member/150`이면 가입자가 150명 이하라는 걸 알 수 있다
- **순차 접근 공격(IDOR)**: ID를 1씩 증가시키며 다른 사용자의 리소스에 접근 시도 가능
- **내부 구조 노출**: 테이블 간 관계나 생성 순서 같은 내부 정보가 드러남

이를 해결하는 대표적인 방법이 **ID 난독화(obfuscation)**다. UUID로 대체하는 방법도 있지만, 기존 auto-increment PK를 유지하면서 API 레이어에서만 변환하는 방식이 변경 범위가 적고, DB 성능에도 영향이 없다.

## 기술 선택: Hashids

[Hashids](https://hashids.org/)는 숫자를 짧고 고유한 문자열로 변환하는 라이브러리다.

```
1  → "Lx3qN7pYvR2z"
42 → "jKpEqWm9oR5x"
```

핵심 특징:
- **양방향 변환**: encode/decode가 모두 가능 (해싱이 아닌 난독화)
- **salt 기반**: 같은 숫자라도 salt가 다르면 다른 결과
- **URL-safe**: 생성된 문자열이 URL에 바로 사용 가능
- **충돌 없음**: 같은 salt에서 같은 입력은 항상 같은 출력

> Hashids는 **암호화가 아니라 난독화**다. 리버스 엔지니어링이 불가능한 건 아니므로, 반드시 서버 측 인가(authorization) 검증과 함께 사용해야 한다.

## 아키텍처 설계

헥사고날 아키텍처에서 ID 난독화를 어디에 배치할지가 핵심 설계 포인트다.

```
┌─────────────────────────────────────────────────────────┐
│                    API Layer (boot)                      │
│                                                         │
│  Request                              Response          │
│  ────────                             ────────          │
│  "Lx3qN7pYvR2z"                      Long → "Lx3qN7p" │
│       │ @DecryptId                     ↑ @EncryptId     │
│       ▼                                │                │
│  Long (원본 ID)                   Long (원본 ID)        │
├─────────────────────────────────────────────────────────┤
│                  Domain Layer                            │
│              (항상 원본 Long ID 사용)                     │
├─────────────────────────────────────────────────────────┤
│               Infrastructure Layer                      │
│            (IdObfuscator 구현체)                         │
└─────────────────────────────────────────────────────────┘
```

**원칙: 도메인은 난독화를 모른다.** 변환은 API 계층에서만 일어나고, 도메인과 인프라는 원본 Long ID를 사용한다.

## 구현

### 1단계: 도메인 포트 정의

도메인 모듈에 인터페이스만 선언한다. 구현은 인프라 모듈에서 한다.

```kotlin
// domain/port/IdObfuscator.kt
interface IdObfuscator {
    fun encode(type: ObfuscationType, id: Long): String
    fun decode(type: ObfuscationType, encoded: String): Long
}
```

엔티티 타입별로 다른 salt를 적용하기 위해 `ObfuscationType`을 함께 받는다.

```kotlin
// domain/id/ObfuscationType.kt
enum class ObfuscationType(val saltSuffix: String) {
    MEMBER("member"),
    TARGET_INFO("target-info"),
    MEMBER_PHOTO("member-photo"),
    MATCHING_RESULT("matching-result"),
}
```

같은 ID `1`이라도 `MEMBER`와 `TARGET_INFO`에서 다른 문자열이 생성된다. 이렇게 하면 한 타입의 ID를 다른 타입의 API에 넣어서 호출하는 것을 방지할 수 있다.

### 2단계: Hashids 구현체

인프라 모듈(`ma-id-obfuscator`)에서 포트를 구현한다.

```kotlin
// infrastructure/support/ma-id-obfuscator
@Component
class HashidsIdObfuscator(
    @Value("\${id-obfuscator.salt}") private val salt: String,
    @Value("\${id-obfuscator.min-length}") private val minLength: Int,
) : IdObfuscator {

    override fun encode(type: ObfuscationType, id: Long): String {
        return hashidsFor(type).encode(id)
    }

    override fun decode(type: ObfuscationType, encoded: String): Long {
        val decoded = hashidsFor(type).decode(encoded)
        if (decoded.isEmpty()) throw InvalidObfuscatedIdException(encoded)
        return decoded[0]
    }

    private fun hashidsFor(type: ObfuscationType): Hashids {
        return Hashids(salt + type.saltSuffix, minLength)
    }
}
```

포인트:
- `salt + type.saltSuffix` 조합으로 타입별 다른 인코딩 결과 생성
- `minLength`로 최소 길이 보장 (12자)
- 디코딩 실패 시 커스텀 예외 발생

설정 파일:

```yaml
# application-local.yml
id-obfuscator:
  salt: "meet-again-dev-salt-do-not-use-in-prod"
  min-length: 12
```

> **운영 환경에서는 salt를 환경변수나 Vault 등 시크릿 매니저로 관리해야 한다.** Git에 커밋되는 순간 난독화의 의미가 사라진다.

### 3단계: 응답 자동 암호화 — @EncryptId + Jackson Serializer

응답 DTO의 ID 필드에 어노테이션만 붙이면 자동으로 난독화되게 만든다.

#### 어노테이션 정의

```kotlin
@Target(AnnotationTarget.FIELD, AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
annotation class EncryptId(val value: ObfuscationType)
```

#### 커스텀 Serializer

```kotlin
class EncryptIdSerializer(
    private val idObfuscator: IdObfuscator,
    private val type: ObfuscationType,
) : JsonSerializer<Long>() {
    override fun serialize(value: Long, gen: JsonGenerator, serializers: SerializerProvider) {
        gen.writeString(idObfuscator.encode(type, value))
    }
}
```

#### AnnotationIntrospector로 자동 연결

Jackson에는 어노테이션을 해석해서 Serializer/Deserializer를 결정하는 `AnnotationIntrospector`가 있다. 이를 커스터마이징하면 `@EncryptId`를 만나면 자동으로 `EncryptIdSerializer`를 사용하게 할 수 있다.

```kotlin
class EncryptIdAnnotationIntrospector(
    private val idObfuscator: IdObfuscator,
) : NopAnnotationIntrospector() {

    override fun findSerializer(a: Annotated): Any? {
        val annotation = a.getAnnotation(EncryptId::class.java) ?: return null
        return EncryptIdSerializer(idObfuscator, annotation.value)
    }

    override fun findDeserializer(a: Annotated): Any? {
        val annotation = a.getAnnotation(EncryptId::class.java) ?: return null
        return EncryptIdDeserializer(idObfuscator, annotation.value)
    }
}
```

#### ObjectMapper에 등록

```kotlin
@Configuration
class ObfuscatedIdJacksonConfig(
    private val objectMapper: ObjectMapper,
    private val idObfuscator: IdObfuscator,
) {
    @PostConstruct
    fun registerEncryptIdIntrospector() {
        val introspector = EncryptIdAnnotationIntrospector(idObfuscator)
        val existing = objectMapper.serializationConfig.annotationIntrospector
        objectMapper.setAnnotationIntrospector(
            AnnotationIntrospector.pair(introspector, existing)
        )
    }
}
```

기존 `AnnotationIntrospector`를 유지하면서 우리의 것을 pair로 추가한다. 이렇게 하면 `@JsonProperty` 같은 기존 어노테이션도 정상 동작한다.

#### 사용 예시

```kotlin
class TargetInfoResponse(
    @EncryptId(ObfuscationType.TARGET_INFO)
    val targetInfoId: Long,
    val targetName: String,
    val targetGender: String,
)
```

응답 JSON:

```json
{
  "targetInfoId": "Lx3qN7pYvR2z",
  "targetName": "홍길동",
  "targetGender": "MALE"
}
```

### 4단계: 요청 자동 복호화 — 두 가지 경로

클라이언트가 보내는 난독화된 ID는 두 곳에서 들어올 수 있다.

| 경로 | 예시 | 처리 방법 |
|------|------|-----------|
| `@PathVariable` | `PUT /target-infos/{targetInfoId}` | Spring Converter |
| `@RequestBody` JSON | `{ "targetInfoId": "Lx3qN7p" }` | Jackson Deserializer |

#### 경로 1: @PathVariable — DecryptId + ConditionalGenericConverter

`@PathVariable`은 Jackson이 아니라 Spring의 타입 변환 시스템이 처리한다. 그래서 별도의 Converter가 필요하다.

```kotlin
@Target(AnnotationTarget.VALUE_PARAMETER)
@Retention(AnnotationRetention.RUNTIME)
annotation class DecryptId(val value: ObfuscationType)
```

```kotlin
class DecryptIdConverter(
    private val idObfuscator: IdObfuscator,
) : ConditionalGenericConverter {

    override fun matches(sourceType: TypeDescriptor, targetType: TypeDescriptor): Boolean {
        return targetType.hasAnnotation(DecryptId::class.java)
    }

    override fun getConvertibleTypes(): Set<ConvertiblePair> {
        return setOf(
            ConvertiblePair(String::class.java, Long::class.java),
            ConvertiblePair(String::class.java, Long::class.javaObjectType),
        )
    }

    override fun convert(source: Any?, sourceType: TypeDescriptor, targetType: TypeDescriptor): Any? {
        if (source == null) return null
        val annotation = targetType.getAnnotation(DecryptId::class.java)
            ?: throw IllegalStateException("@DecryptId 어노테이션을 찾을 수 없습니다.")
        return idObfuscator.decode(annotation.value, source as String)
    }
}
```

`ConditionalGenericConverter`를 사용한 이유:
- `matches()`로 **`@DecryptId`가 있는 파라미터에만** 적용
- 일반 String → Long 변환에는 영향을 주지 않음

WebMvc에 등록:

```kotlin
@Configuration
class WebConfig(
    private val idObfuscator: IdObfuscator,
) : WebMvcConfigurer {
    override fun addFormatters(registry: FormatterRegistry) {
        registry.addConverter(DecryptIdConverter(idObfuscator))
    }
}
```

사용 예시:

```kotlin
@PutMapping("/{targetInfoId}")
fun update(
    @PathVariable @DecryptId(ObfuscationType.TARGET_INFO) targetInfoId: Long,
    @RequestBody request: UpdateTargetInfoRequest,
): TargetInfoResponse
```

#### 경로 2: @RequestBody — Jackson Deserializer

JSON 바디에 포함된 ID는 `@EncryptId` 어노테이션의 Deserializer가 처리한다. 위에서 `EncryptIdAnnotationIntrospector`에 `findDeserializer()`도 함께 구현했기 때문에 추가 작업 없이 동작한다.

```kotlin
class EncryptIdDeserializer(
    private val idObfuscator: IdObfuscator,
    private val type: ObfuscationType,
) : JsonDeserializer<Long>() {
    override fun deserialize(p: JsonParser, ctxt: DeserializationContext): Long {
        return idObfuscator.decode(type, p.valueAsString)
    }
}
```

### 5단계: 예외 처리

잘못된 인코딩 값이 들어오면 명확한 에러 응답을 내려준다.

```kotlin
class InvalidObfuscatedIdException(
    encodedValue: String,
) : BusinessException(
    message = "유효하지 않은 ID입니다.",
    dataMessage = "encoded: $encodedValue",
    logLevel = LogLevel.WARN,
)
```

## 전체 데이터 흐름

하나의 요청-응답 사이클에서 ID가 어떻게 흘러가는지 정리하면 다음과 같다.

```
[클라이언트]
    │
    │  PUT /target-infos/Lx3qN7pYvR2z
    │  Body: { "matchingResultId": "jKpEqWm9oR5x" }
    ▼
[Spring MVC — PathVariable 처리]
    │  @DecryptId 감지 → DecryptIdConverter.convert()
    │  → idObfuscator.decode(TARGET_INFO, "Lx3qN7pYvR2z") → 42
    ▼
[Jackson — RequestBody 역직렬화]
    │  EncryptIdAnnotationIntrospector.findDeserializer()
    │  → @EncryptId(MATCHING_RESULT) 감지
    │  → EncryptIdDeserializer(idObfuscator, MATCHING_RESULT) 인스턴스 생성
    │  → idObfuscator.decode(MATCHING_RESULT, "jKpEqWm9oR5x") → 7
    ▼
[Controller]
    │  targetInfoId: Long = 42
    │  matchingResultId: Long = 7
    ▼
[Service / Domain]
    │  원본 Long ID로 비즈니스 로직 수행
    ▼
[Jackson — Response 직렬화]
    │  EncryptIdAnnotationIntrospector.findSerializer()
    │  → @EncryptId(TARGET_INFO) 감지
    │  → EncryptIdSerializer(idObfuscator, TARGET_INFO) 인스턴스 생성
    │  → idObfuscator.encode(TARGET_INFO, 42) → "Lx3qN7pYvR2z"
    ▼
[클라이언트]
    {
      "targetInfoId": "Lx3qN7pYvR2z",
      "matchingResultId": "jKpEqWm9oR5x"
    }
```

포인트는 `EncryptIdAnnotationIntrospector`가 각 필드의 `@EncryptId` 어노테이션을 읽고, 해당 `ObfuscationType`에 맞는 Serializer/Deserializer를 **그때그때 인스턴스로 생성**한다는 것이다. Bean으로 등록된 하나의 Serializer가 아니라, 필드마다 다른 `ObfuscationType`을 가진 별도 인스턴스가 만들어진다.

### Jackson의 Serializer 결정 순서

`AnnotationIntrospector.pair()`로 등록했기 때문에, Jackson은 필드를 직렬화할 때 체인 형태로 Serializer를 찾는다.

```
필드 직렬화 시:

1. EncryptIdAnnotationIntrospector.findSerializer() 호출
   → @EncryptId 있으면 EncryptIdSerializer 반환 ✅ 여기서 결정
   → 없으면 null 반환
       ↓
2. 기존 AnnotationIntrospector에게 위임
   → @JsonSerialize 있으면 해당 Serializer 반환
   → 없으면 null 반환
       ↓
3. 타입 기반 기본 Serializer 사용
   → String  → StringSerializer
   → Long    → NumberSerializer
   → List    → CollectionSerializer
   → Boolean → BooleanSerializer
   → 등등
```

`ObjectMapper`에 등록할 때 `pair()`를 사용하는 이유가 여기에 있다.

```kotlin
objectMapper.setAnnotationIntrospector(
    AnnotationIntrospector.pair(introspector, existing)  // 우리 것 + 기존 것
)
```

`pair`의 첫 번째 인자가 우선 적용되고, null을 반환하면 두 번째(기존)에게 위임한다. 만약 `pair` 없이 우리 것만 등록하면 `@JsonProperty` 같은 기존 Jackson 어노테이션이 동작하지 않게 된다. `@EncryptId`가 붙은 필드만 커스텀 처리되고, 나머지는 기존 Jackson 동작 그대로 유지된다.

## 테스트 전략

### 단위 테스트

```kotlin
class HashidsIdObfuscatorTest : BehaviorSpec({
    val obfuscator = HashidsIdObfuscator(salt = "test-salt", minLength = 12)

    Given("같은 ID라도") {
        When("ObfuscationType이 다르면") {
            Then("다른 인코딩 결과가 나온다") {
                val memberEncoded = obfuscator.encode(ObfuscationType.MEMBER, 1L)
                val targetEncoded = obfuscator.encode(ObfuscationType.TARGET_INFO, 1L)
                memberEncoded shouldNotBe targetEncoded
            }
        }
    }

    Given("인코딩된 값을") {
        When("디코딩하면") {
            Then("원본 ID가 복원된다") {
                val encoded = obfuscator.encode(ObfuscationType.MEMBER, 42L)
                obfuscator.decode(ObfuscationType.MEMBER, encoded) shouldBe 42L
            }
        }
    }

    Given("잘못된 인코딩 값이면") {
        When("디코딩 시도 시") {
            Then("예외가 발생한다") {
                shouldThrow<InvalidObfuscatedIdException> {
                    obfuscator.decode(ObfuscationType.MEMBER, "invalid!!!")
                }
            }
        }
    }
})
```

### 통합 테스트 — 실제 API 호출

```kotlin
@BaseApiTest
class TargetInfoApiTest : BehaviorSpec({
    Given("찾는 사람 정보를 수정할 때") {
        When("난독화된 targetInfoId로 요청하면") {
            Then("수정된 결과가 난독화된 ID와 함께 반환된다") {
                mockMvc.put("/api/target-infos/{targetInfoId}", encodedTargetInfoId) {
                    contentType = MediaType.APPLICATION_JSON
                    content = objectMapper.writeValueAsString(request)
                }.andExpect {
                    status { isOk() }
                    jsonPath("$.targetInfoId") { isString() }  // Long이 아닌 String
                }
            }
        }
    }
})
```

## 정리

| 항목 | 설명 |
|------|------|
| **라이브러리** | Hashids (양방향 난독화) |
| **적용 범위** | API 계층만 (도메인은 원본 ID) |
| **응답 암호화** | `@EncryptId` + Jackson AnnotationIntrospector |
| **PathVariable 복호화** | `@DecryptId` + ConditionalGenericConverter |
| **RequestBody 복호화** | `@EncryptId` + Jackson Deserializer |
| **타입 안전성** | ObfuscationType enum으로 엔티티별 다른 salt |
| **에러 처리** | InvalidObfuscatedIdException |

이 방식의 장점은 **기존 코드 변경이 최소화**된다는 것이다. 도메인 모델, 서비스, 리포지토리는 전혀 수정하지 않고, API 계층의 DTO에 어노테이션만 추가하면 된다. 새로운 엔티티가 추가되면 `ObfuscationType`에 값 하나 추가하고 DTO에 `@EncryptId`를 붙이면 끝이다.
