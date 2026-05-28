---
title: "사용자 친화적 ES 검색 만들기 (3) — `:` 토큰의 함정과 field-aware tokenization"
date: 2026-05-28 22:00:00 +0900
categories: [Database, Elasticsearch]
tags: [elasticsearch, lexer, parser, tokenization, kotlin, search-ux, url-search, ipv6, mac-address]
toc: true
toc_sticky: true
---

## 이전 글

- [(1편) — query_string 예약 문자와 syntax 오류로부터 사용자 입력 지키기]({% post_url 2026-05-28-es-query-string-parser-1-escape-and-syntax-fallback %})
- [(2편) — 4xx와 5xx를 분리해 운영 ERROR 알람 노이즈 없애기]({% post_url 2026-05-28-es-query-string-parser-2-4xx-classification %})

1편에서 syntax 오류와 예약 문자 함정을 막았고, 2편에서 4xx/5xx 알람을 분리했다. 그래서 모든 검색이 매끄럽게 돌아갈 줄 알았다. 그런데 오늘 또 알람이 떴다.

---

## 1. 운영 사고 — `https://example.com` 검색이 400으로 떨어지다

운영 로그:

```json
POST /api/assets/search?page=1&size=10
{ "value": "https://example.com", "riskfactors": [], "labels": [] }
```

스택 트레이스 핵심:

```
org.springframework.dao.InvalidDataAccessApiUsageException: 알 수 없는 필드: https
Caused by: BadRequestException: 알 수 없는 필드: https
    at QueryStringParser.termToQuery(QueryStringParser.kt:453)
```

사용자가 검색창에 그냥 URL을 붙여넣은 것이다. 우리 파서는 이걸 `https`(field) + `://example.com`(value)로 해석하려 했고, `https`라는 field가 매핑에 없어서 `BadRequestException`을 던졌다. 2편의 분류 덕분에 500은 아니고 400으로 응답되긴 했지만 — **사용자 입장에선 검색이 그냥 실패한다**.

같은 함정에 빠지는 입력들을 줄세워 보면 한눈에 들어온다.

| 입력 | 사용자 의도 | 파서가 본 것 |
|------|----------|------------|
| `https://example.com` | URL 검색 | `https` field, `//example.com` value |
| `mailto:user@example.com` | 이메일 검색 | `mailto` field, `user@example.com` value |
| `12:30:45` | 시간 검색 | `12` field, `30` value, 그리고 `:45` 토큰 잔여 |
| `AA:BB:CC:DD:EE:FF` | MAC 검색 | `AA` field, `BB` value, 잔여 다수 |
| `2001:db8::1` | IPv6 검색 | 마찬가지로 폭망 |
| `domain:example.com` | **의도된 field 검색** | `domain` field, `example.com` value ✓ |

마지막 한 줄만 의도와 맞는다.

---

## 2. 근본 원인 — Lexer가 `:`을 무조건 구분자로 잘라낸다

문제 코드(요약).

```kotlin
private class Lexer(input: String) {
    companion object {
        private val P = Pattern.compile(
            "\\s+|(?i)AND\\b|(?i)OR\\b|(?i)NOT\\b" +
            "|\\+|\\-|\\(|\\)|:|\"(?:\\\\.|[^\"])*\"|[^\\s():+\\-]+"
        )
    }
    // ...
}
```

토크나이저가 `:`을 그 자체로 한 토큰(`COLON`)으로 잘라낸다. 컨텍스트(앞 토큰이 등록된 field인지 아닌지)를 보지 않는다.

이어서 Parser는 `TERM` 다음에 `COLON`이 오면 **무조건** `field:value` 구문으로 해석한다.

```kotlin
private fun parseTerm(): Node {
    val first = lx.peek()
    if (first.t == TT.TERM) {
        val save = lx.take()
        if (lx.peek().t == TT.COLON) {   // ← 무조건 field:value
            lx.take()
            val field = save.s
            // ...
            return Term(field, valueRaw)
        }
        return Term(null, tokenToRaw(save))
    }
    // ...
}
```

그 결과 `termToQuery` 단에서 `fieldMap[field]` 가 null이면 `BadRequestException("알 수 없는 필드: ...")` 이 떠버린다.

> 단순히 try-catch를 더 넓혀서 잡아 phrase로 fallback할 수도 있다. 그건 같은 사고가 또 발생할 여지를 그대로 두는 **땜빵**이다. URL, 시간, MAC, IPv6, 우리가 미처 생각 못 한 다음 입력 — 전부 같은 함정에 또 빠진다.

---

## 3. 근본 fix — Parser가 fieldMap을 인지하게 만든다

`:` 앞 토큰이 **등록된 field 이름일 때만** `field:value`로 해석하고, 그 외에는 `:`과 뒤 토큰들을 **단일 자유 검색어로 흡수**하도록 파서를 고친다.

### 3-1. Parser가 knownFields를 받게 한다

```kotlin
private class Parser(
    s: String,
    private val knownFields: Set<String>,
) {
    private val lx = Lexer(s)
    // ...
}

fun parse(userQuery: String): Query {
    val ast = try {
        Parser(userQuery, fieldMap.keys).parse()
    } catch (_: BadRequestException) {
        return buildDefaultQueryStringGrouped(wrapAsPhrase(userQuery))
    }
    return toQuery(ast) ?: Query.of { q -> q.matchAll { it } }
}
```

### 3-2. `TERM:` 시퀀스를 만났을 때 분기

`parseTerm()`이 너무 비대해질 뻔해 세 함수로 나눴다. 각 함수는 한 가지 일만 한다.

```kotlin
private fun parseTerm(): Node {
    val first = lx.peek()
    if (first.t == TT.TERM) {
        val save = lx.take()
        return if (lx.peek().t == TT.COLON) parseAfterTermColon(save)
        else Term(null, tokenToRaw(save))
    }
    if (first.t == TT.QUOTED) return Term(null, tokenToRaw(lx.take()))
    throw BadRequestException("Expected term/quoted, got: $first")
}

/**
 * 'TERM ":"' 시퀀스를 만났을 때 호출. ':' 앞 TERM 이 등록된 필드명일 때만
 * field:value 구문으로 해석하고, 그 외 (URL "https://..", 시간 "12:30",
 * MAC "AA:BB:..", IPv6 등) 는 ':' 과 뒤 토큰들을 단일 자유 검색어로 흡수한다.
 */
private fun parseAfterTermColon(termTok: Tok): Node {
    val candidate = termTok.s ?: ""
    if (candidate !in knownFields) return absorbAsFreeTerm(termTok)
    return parseFieldScopedTerm(candidate)
}

private fun parseFieldScopedTerm(field: String): Node {
    lx.take() // consume ':'
    // ... 기존 field:value 로직 (LP 그룹/QUOTED/TERM 처리)
}
```

### 3-3. `absorbAsFreeTerm` — `:`을 단순 흡수

이게 핵심이다. 뒤따르는 `:` 들과 그 뒤 `TERM`/`QUOTED` 들을 흡수해 단일 raw 문자열로 만든다. **연속 `::` (IPv6 shorthand) 도 처리**한다.

```kotlin
/**
 * 등록되지 않은 'TERM ":"' 토큰열을 만났을 때 호출. 이어지는 ':' 들과 그 뒤
 * TERM/QUOTED 들을 흡수해 단일 자유 검색어로 재조립한다.
 *
 * 예) https://example.com  → "https://example.com"
 *     12:30:45              → "12:30:45"
 *     AA:BB:CC:DD:EE:FF     → "AA:BB:CC:DD:EE:FF"
 *     2001:db8::1           → "2001:db8::1"  (연속 :: 도 흡수)
 */
private fun absorbAsFreeTerm(firstTok: Tok): Node {
    val sb = StringBuilder(firstTok.s ?: "")
    while (lx.peek().t == TT.COLON) {
        lx.take()
        sb.append(':')
        when (lx.peek().t) {
            TT.TERM, TT.QUOTED -> sb.append(lx.take().s ?: "")
            TT.COLON -> continue          // 연속 '::' 다음 iteration 에서 흡수
            else -> break                 // EOF / 연산자 / 괄호 → trailing colon 으로 종료
        }
    }
    return Term(null, sb.toString())
}
```

### 3-4. 도달 불가가 된 throw는 invariant로 격상

이제 parser 단에서 unknown field를 미리 흡수하기 때문에, `termToQuery`의 fieldMap 미스 throw는 **도달 불가**다. 안전망으로 남기되 의미를 격상한다.

```kotlin
private fun termToQuery(t: Term): Query {
    return if (t.fieldOrNull == null) {
        buildDefaultQueryStringGrouped(t.rawValue)
    } else {
        val fs = fieldMap[t.fieldOrNull]
            ?: error("Parser invariant violated: unknown field '${t.fieldOrNull}' should have been absorbed as free term")
        // ...
    }
}
```

`BadRequestException` 대신 `error()`(== `IllegalStateException`)를 던지게 했다. 만약 향후 누군가 parser 로직을 망가뜨려 unknown field가 흘러나오면 5xx로 뜨고 ERROR 알람이 울려야 한다 — 그건 사용자 입력 문제가 아니라 **우리 invariant 위반**이기 때문이다.

---

## 4. 동작 검증 — Before / After

| 입력 | Before | After |
|------|--------|-------|
| `https://example.com` | 400 BadRequest | 200 + phrase 검색 |
| `12:30:45` | 400 | 200 + phrase |
| `AA:BB:CC:DD:EE:FF` | 400 | 200 + phrase |
| `2001:db8::1` (IPv6 shorthand) | 400 | 200 + phrase |
| `mailto:user@example.com` | 400 | 200 + phrase |
| `unknown:foo` (단순 미정의 필드) | 400 | 200 + phrase |
| `foo:` (trailing colon) | 400 | 200 + phrase |
| `Domain:example.com` (대소문자 다름) | 400 | 200 + phrase |
| `domain:example.com` (known field) | 200 + field 검색 | 200 + field 검색 (변경 없음) |
| `do)` (syntax error) | 200 + phrase fallback | 200 + phrase fallback (1편 그대로) |

`do)` 케이스가 그대로라는 점이 중요하다. 1편의 lenient fallback은 **syntax 오류용**으로 남아 있고, 이번 fix는 **semantic 오류(미정의 field)**를 parser 단에서 미리 흡수하는 별도 레이어다. 두 레이어는 서로 간섭하지 않는다.

---

## 5. 테스트 — 회귀 방지를 촘촘하게

이번 변경은 lexer/parser의 의미론을 바꾼 만큼, 운영에서 마주칠 만한 모든 `:` 포함 패턴을 테스트로 묶어 회귀를 막았다. 총 48개 케이스를 5개 블록으로 분류했다.

```kotlin
@Nested
inner class 콜론_포함_자유_검색어_안전_처리 {

    @Test fun `https URL 은 자유 검색어로 처리 - 운영 사고 입력`() {
        val j = json(parser().parse("https://example.com"))
        assertTrue(j.contains("\"query_string\""), j)
        assertFieldsAreDefaultGroup(j)
    }

    @Test fun `시간 형식 hh-mm-ss 자유 검색 - 연속 콜론 흡수`() {
        val j = json(parser().parse("12:30:45"))
        assertFieldsAreDefaultGroup(j)
        assertTrue(extractQueries(j).joinToString("|").contains("45"))
    }

    @Test fun `MAC 주소 자유 검색`() {
        val j = json(parser().parse("AA:BB:CC:DD:EE:FF"))
        assertFieldsAreDefaultGroup(j)
    }

    @Test fun `IPv6 shorthand 연속 콜론 자유 검색`() {
        // "::" -> COLON + COLON 두 토큰. absorbAsFreeTerm 의 연속 ':' continue 분기 검증
        val j = json(parser().parse("2001:db8::1"))
        assertFieldsAreDefaultGroup(j)
    }

    @Test fun `mailto 스킴`() { ... }
    @Test fun `tel 스킴`() { ... }
    @Test fun `URL with port - https://example.com:8080`() { ... }
    @Test fun `trailing colon 입력 안전 처리`() { ... }
    // ... 17 cases
}

@Nested
inner class 혼합_검색 {
    @Test fun `known field 와 URL 혼합 - AND`() {
        val j = json(parser().parse("domain:example.com AND https://other.com"))
        assertTrue(j.contains("\"fields\":[\"domain\"]"), j)             // known field
        assertTrue(j.contains("\"fields\":[\"domain\",\"ips\"]"), j)     // free term default group
    }
    // ... 6 cases
}

@Nested
inner class 경계_케이스 {
    @Test fun `known field 정확히 일치할 때만 field 검색 - 대소문자 구분`() {
        // 'Domain' (대문자 D) 는 fieldMap 키 'domain' 과 다름 → 자유 검색어
        val j = json(parser().parse("Domain:example.com"))
        assertFieldsAreDefaultGroup(j)
    }
    // ... 6 cases
}
```

전체 테스트:

```
tests=48, failures=0, errors=0
```

---

## 6. 회고 — 땜빵과 근본 fix의 갈림길

이번 사고에서 가장 의미 있는 결정은 **try-catch를 더 넓혀 잡지 않은 것**이다. 그 길로 갔으면 코드는 한 줄로 끝났을 거고, 운영 알람도 잠잠해졌을 것이다. 다음 같은 패턴이 또 들어왔을 때(아마 곧 IPv6나 윈도우 경로 `C:\...` 같은 형태가 들어왔을 것이다)까지는.

**원인은 lexer가 `:`을 무조건 구분자로 봤다는 한 가지였고, 그 가정이 깨진 모든 케이스가 같은 사고를 반복**했다. 가정을 고치니 URL/시간/MAC/IPv6/mailto/tel/trailing colon 케이스 — 우리가 명시적으로 생각하지 못한 입력까지 — 한 번에 해결됐다.

> 사용자 입력은 우리 문법의 가정을 자주 깬다. **catch로 막기 전에, 가정이 맞는지부터 다시 본다.**

세 편의 시리즈를 마치며 우리 `QueryStringParser`가 따르는 사용자 친화 원칙을 정리해 본다.

1. **syntax 오류는 throw 대신 평문 phrase로 fallback** (1편)
2. **ES 측 예약 문자는 leaf 단에서 escape** (1편)
3. **의도된 4xx와 진짜 5xx를 도메인 예외로 분리해 운영 알람 노이즈 제거** (2편)
4. **`:` 같은 모호한 토큰은 컨텍스트(field 등록 여부)로 분기 — 가정을 안 깨는 lexer/parser** (3편)

검색은 사용자가 가장 자주 만지는 입력 지점이고, 동시에 가장 다양한 함정이 모이는 곳이다. "사용자가 안 좋게 입력해서 실패했다"는 메시지를 던지기 전에, **그 입력으로도 그럴듯한 결과를 돌려줄 수 있는 길이 있는지** 한 번 더 본다 — 그게 검색 UX의 출발이다.
