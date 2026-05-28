---
title: "사용자 친화적 ES 검색 만들기 (2) — 4xx와 5xx를 분리해 운영 ERROR 알람 노이즈 없애기"
date: 2026-05-28 21:30:00 +0900
categories: [Database, Elasticsearch]
tags: [elasticsearch, exception-handling, spring-boot, aop, observability, grafana-loki, slack-alert, kotlin]
toc: true
toc_sticky: true
---

## 이전 글

[(1편) — query_string 예약 문자와 syntax 오류로부터 사용자 입력 지키기]({% post_url 2026-05-28-es-query-string-parser-1-escape-and-syntax-fallback %})

1편에서 검색 파서가 사용자 입력에 throw하지 않게 만들었다. 그런데도 운영 ERROR 알람은 사라지지 않았다. 이번 글의 주제는 **5xx로 찍히는 4xx**를 어떻게 분리해 알람 노이즈를 없앴는지다.

---

## 1. 문제 — "정상 4xx 응답"인데 알람이 울리는 이유

운영 모니터링은 보통 다음 룰을 쓴다.

```
{app="my-app-prd"} |= "ERROR" != "Broken pipe"
```

5분 동안 위 라인이 1건 이상 잡히면 Slack으로 알람. 단순하고 잘 작동하는 패턴이다. 그런데 우리 코드의 두 군데가 이 룰에 자꾸 걸렸다.

### 1-1. `ApiExecutionTimeAspect` — 모든 예외를 ERROR로 찍는다

API 실행 시간을 측정하는 AOP가 있다. 예외가 던져지면 `[API ERROR] ...` 라인을 ERROR 레벨로 남긴다.

```kotlin
@Around("@within(org.springframework.web.bind.annotation.RestController)")
fun measureExecutionTime(joinPoint: ProceedingJoinPoint): Any? {
    val start = System.currentTimeMillis()
    var thrown: Throwable? = null
    return try {
        joinPoint.proceed()
    } catch (e: Exception) {
        thrown = e
        throw e
    } finally {
        val elapsed = System.currentTimeMillis() - start
        if (thrown != null) {
            logger.error { "[API ERROR] $method $uri - ${elapsed}ms" }
        } else if (elapsed >= 5000) {
            logger.warn { "[SLOW API] $method $uri - ${elapsed}ms" }
        } else {
            logger.info { "[API] $method $uri - ${elapsed}ms" }
        }
    }
}
```

문제는 **GlobalExceptionHandler가 4xx로 정상 응답해도** AOP의 `finally`는 이미 예외를 본 상태라 ERROR로 찍는다는 점. 클라이언트는 `404 Not Found` 같은 멀쩡한 응답을 받았는데, 우리는 ERROR 알람을 받는다.

### 1-2. 표준 예외를 4xx로 굳혀버린 핸들러

처음 만든 `GlobalExceptionHandler`는 표준 예외를 그대로 4xx로 매핑했다.

```kotlin
@ExceptionHandler(IllegalArgumentException::class)
@ResponseStatus(HttpStatus.BAD_REQUEST)
fun handleIllegalArgumentException(e: IllegalArgumentException): ApiResponseDto<Nothing> {
    logger.warn { "Illegal argument: ${e.message}" }
    return ApiResponseDto.badParameter(e.message)
}

@ExceptionHandler(NoSuchElementException::class)
@ResponseStatus(HttpStatus.NOT_FOUND)
fun handleNoSuchElementException(e: NoSuchElementException): ApiResponseDto<Nothing> {
    logger.warn { "Not found: ${e.message}" }
    return ApiResponseDto.notFound(e.message)
}
```

겉으로는 깔끔해 보이지만 두 가지 부작용이 있었다.

1. **진짜 버그를 가린다** — 코드 어디선가 `null!!` 같은 케이스로 `NullPointerException → IllegalArgumentException` 으로 변환된 예외가 흘러나오면, "서버 버그"인데 400으로 응답해 모니터링이 못 잡는다.
2. **검색 파서 예외도 4xx로 흘렀다** — 1편의 lenient fallback 이전에는 `IllegalArgumentException("Unexpected: RP())")` 같은 파서 예외가 위 핸들러에 잡혀 400으로 응답되긴 했지만, AOP는 여전히 ERROR로 찍어 알람을 울렸다.

요약: 4xx가 운영 ERROR 알람에 새고 있었다.

---

## 2. 설계 — 도메인 예외로 의도된 4xx를 분리

핵심 아이디어는 두 가지다.

1. **표준 예외(`IllegalArgumentException`, `NoSuchElementException`)는 더 이상 4xx로 매핑하지 않는다.** 이 예외들이 떠야 진짜 코드 버그가 운영에서 5xx ERROR로 시각화된다.
2. **의도된 4xx는 도메인 예외로 명시한다** — `BadRequestException`, `NotFoundException`, `ForbiddenException`, `DuplicateException` 등. 핸들러는 이 도메인 예외만 4xx로 매핑한다.

### 도메인 예외

```kotlin
class BadRequestException(message: String) : RuntimeException(message)
class NotFoundException(message: String) : RuntimeException(message)
class ForbiddenException(message: String) : RuntimeException(message)
class DuplicateException(message: String) : RuntimeException(message)
```

### 호출부 — 의도된 4xx는 도메인 예외로 명시적 던지기

전에는 service에서 이렇게 썼다.

```kotlin
fun findAsset(assetId: String): Asset {
    return assetRepository.findOneOrNull(assetId)
        ?: throw NoSuchElementException("Asset not found: $assetId")
}
```

이제 의도가 명확하게 드러난다.

```kotlin
fun findAsset(assetId: String): Asset {
    return assetRepository.findOneOrNull(assetId)
        ?: throw NotFoundException("Asset not found: $assetId")
}
```

### 핸들러 — 도메인 예외만 4xx, 표준 예외는 fallback

```kotlin
@ControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(BadRequestException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleBadRequest(e: BadRequestException): ApiResponseDto<Nothing> {
        logger.warn { "Bad request: ${e.message}" }
        return ApiResponseDto.badParameter(e.message)
    }

    @ExceptionHandler(NotFoundException::class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleNotFound(e: NotFoundException): ApiResponseDto<Nothing> {
        logger.warn { "Not found: ${e.message}" }
        return ApiResponseDto.notFound(e.message)
    }

    // IllegalArgumentException / NoSuchElementException 핸들러는 제거
    // → Exception fallback 으로 빠져 500 ERROR 로 처리됨

    @ExceptionHandler(Exception::class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    fun handleException(e: Exception, req: HttpServletRequest): ApiResponseDto<Nothing> {
        logger.error(e) { "Unhandled exception: ${req.method} ${req.requestURI} - ${e.message}" }
        return ApiResponseDto.serverError()
    }
}
```

### AOP — 4xx와 5xx를 로깅 레벨로 분리

가장 결정적인 변경은 AOP에서 분기를 둔 것.

```kotlin
@Around("@within(org.springframework.web.bind.annotation.RestController)")
fun measureExecutionTime(joinPoint: ProceedingJoinPoint): Any? {
    val start = System.currentTimeMillis()
    var thrown: Throwable? = null
    return try {
        joinPoint.proceed()
    } catch (e: Exception) {
        thrown = e
        throw e
    } finally {
        val elapsed = System.currentTimeMillis() - start
        val t = thrown
        when {
            t != null && isClientError(t) ->
                logger.warn { "[API 4XX] $method $uri - ${elapsed}ms - ${t.javaClass.simpleName}: ${t.message}" }
            t != null ->
                logger.error { "[API ERROR] $method $uri - ${elapsed}ms" }
            elapsed >= 5000 ->
                logger.warn { "[SLOW API] $method $uri - ${elapsed}ms" }
            else ->
                logger.info { "[API] $method $uri - ${elapsed}ms" }
        }
    }
}

private fun isClientError(e: Throwable): Boolean = when (e) {
    is NotFoundException,
    is BadRequestException,
    is DuplicateException,
    is ForbiddenException,
    is MethodArgumentNotValidException,
    is HandlerMethodValidationException,
    is MissingRequestHeaderException,
    is MissingServletRequestParameterException,
    is HttpMessageNotReadableException -> true
    else -> false
}
```

이제 운영 알람 룰(`|= "ERROR"`)에는 진짜 5xx만 잡힌다. 4xx는 `[API 4XX]` WARN으로 따로 떨어진다.

---

## 3. 검색 파서와의 연결고리

1편에서 본 lenient fallback도 이 분류와 함께 가야 깔끔해진다. 검색 파서 내부의 `IllegalArgumentException`을 모두 `BadRequestException`으로 바꿔, fallback에 도달하지 않은 진짜 4xx 케이스(예: 의도적으로 던진 검증 실패)는 400으로 응답하고 ERROR 알람을 안 울리게 했다.

```kotlin
class QueryStringParser(...) {

    private class Parser(s: String) {
        fun parse(): Node {
            val n = parseOr()
            if (lx.peek().t != TT.EOF) throw BadRequestException("Unexpected: ${lx.peek()}")
            return n
        }
        // ... 다른 throw 도 모두 BadRequestException
    }

    fun parse(userQuery: String): Query {
        val ast = try {
            Parser(userQuery).parse()
        } catch (_: BadRequestException) {
            // 사용자 입력 syntax error → 평문 phrase 로 lenient fallback
            return buildDefaultQueryStringGrouped(wrapAsPhrase(userQuery))
        }
        return toQuery(ast) ?: Query.of { q -> q.matchAll { it } }
    }
}
```

`BadRequestException`을 던지는 코드가 늘어도, fallback catch가 잡아주면 사용자에겐 200 OK, 운영에는 ERROR 노이즈 없음.

---

## 4. 효과 — ERROR 알람이 진짜 의미를 되찾는다

배포 후 며칠 동안 운영 ERROR 알람 빈도:

- 변경 전: 하루 평균 20~50건 (대부분 4xx 노이즈)
- 변경 후: 하루 0~3건 (대부분 진짜 5xx — 외부 API 타임아웃, 마이그레이션 누락된 데이터 등)

알람이 적게 울리는 게 끝이 아니다. **알람이 울리면 거의 항상 진짜 사고**라는 신뢰가 회복된 것이 가장 큰 효과였다. 이전에는 ERROR 알람을 봐도 "또 사용자 입력 오류겠지"라며 처음엔 가볍게 넘기다 진짜 5xx를 놓치는 일이 종종 있었다.

---

## 5. 부수효과 — 진짜 버그가 5xx로 빨리 드러난다

`IllegalArgumentException` 핸들러를 제거한 부수효과로, 누군가 실수로 `require(value > 0)` 같은 검증을 service에 두면 그 예외가 5xx로 떠 운영 알람을 울린다. 그게 정상이다. **사용자 입력 검증은 도메인 예외로 명시적으로**, **불변 보장은 표준 예외(buggy code면 5xx)** 로 분리되면서 의도가 코드에 드러난다.

```kotlin
// 의도된 4xx — 사용자 입력 검증
fun requireValidPort(port: Int) {
    if (port !in 1..65535) throw BadRequestException("port out of range: $port")
}

// 의도되지 않은 5xx — 불변 위반은 코드 버그
fun applyDiscount(rate: Double) {
    require(rate in 0.0..1.0) { "discount rate must be in [0.0, 1.0]" }
    // ...
}
```

---

## 정리

| Before | After |
|--------|-------|
| `IllegalArgumentException` → 400 핸들러 | 핸들러 제거 → fallback이 500으로 처리 (진짜 버그 가시화) |
| `NoSuchElementException` → 404 핸들러 | `NotFoundException` 도메인 예외만 404 |
| AOP가 모든 예외를 `[API ERROR]` ERROR로 로깅 | 도메인 4xx 예외는 `[API 4XX]` WARN, 나머지만 ERROR |
| 운영 ERROR 알람 일평균 수십 건 | 진짜 5xx만 잡혀 일평균 0~3건 |

다음 편에서는 **여전히 남아있던 마지막 함정** — `:` 토큰 처리 — 을 다룬다. 이번 4xx 분류로 5xx가 400으로 내려갔지만, 사용자가 검색창에 URL을 그대로 붙여넣었을 때 **400 자체가 잘못된 응답**이라는 점을 알게 된 사고였다.

다음 글: [(3편) — `:` 토큰의 함정과 field-aware tokenization]({% post_url 2026-05-28-es-query-string-parser-3-field-aware-tokenization %})
