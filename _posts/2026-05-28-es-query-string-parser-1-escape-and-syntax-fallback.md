---
title: "사용자 친화적 ES 검색 만들기 (1) — query_string 예약 문자와 syntax 오류로부터 사용자 입력 지키기"
date: 2026-05-28 21:00:00 +0900
categories: [Database, Elasticsearch]
tags: [elasticsearch, query-string, lucene, kotlin, parser, lenient-fallback, query-builder, side-project]
toc: true
toc_sticky: true
---

## 들어가며

실무 프로젝트에서 Elasticsearch를 검색 백엔드로 쓰면서, "사용자가 검색창에 그대로 붙여넣을 만한 모든 문자열을 안전하게 받아낼 수 있을까?"라는 질문을 자주 마주쳤다. 결론부터 말하면 **그냥 ES `query_string` 쿼리에 사용자 입력을 통째로 넘기면 운영에서 무조건 사고가 난다.**

총 세 편 시리즈로, 우리가 만든 자체 `QueryStringParser`가 어떤 사고들을 만나며 점점 사용자 친화적으로 진화해 왔는지를 정리한다.

- **1편 (이 글)** — 예약 문자와 syntax 오류로 ES 쿼리가 거절되거나 정규식으로 해석되는 함정, 그리고 lenient fallback 설계
- 2편 — 사용자 입력 오류(400)와 진짜 서버 오류(500)를 분리해 운영 ERROR 알람 노이즈를 없애기
- 3편 — `:` 토큰의 함정과 field-aware tokenization으로 URL/시간/MAC 자유 검색 처리하기

---

## 1. 왜 자체 파서를 만들었나

대다수 검색 UI는 사용자에게 "구글처럼" 검색어 + 몇 가지 연산자(`AND`, `OR`, `NOT`, `+`, `-`, 따옴표, 괄호)를 제공한다. ES에는 이미 [`query_string`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html) 쿼리가 있어 이 표현식을 거의 그대로 받아주지만, 두 가지 한계가 있다.

1. **사용자 입력의 정합성을 보장하지 않는다** — 파싱 실패(짝 안 맞는 괄호, 예약 문자 오용 등) 시 ES가 그대로 `400`을 던지거나, 더 나쁘게는 `all shards failed`로 떨어진다.
2. **field alias가 ES 매핑과 1:1로 묶여 있어 도메인 모델과 분리되지 않는다** — 사용자가 입력하는 `domain:...`을 우리가 정의한 nested 필드(`softwares.port` 같은 것)로 매핑해 주고 싶다.

그래서 사용자 표현식 → AST → ES `query_string` 으로 두 단계 변환하는 자체 파서를 만들었다. 대략 이런 모양이다.

```kotlin
class QueryStringParser(
    private val fieldMap: Map<String, FieldSpec>,
    private val defaultFields: List<FieldSpec>,
) {
    fun parse(userQuery: String): Query {
        val ast = Parser(userQuery).parse()
        return toQuery(ast) ?: Query.of { q -> q.matchAll { it } }
    }
    // ... Lexer / Parser / AST → ES Query 변환
}
```

여기까지는 잘 동작했다. 그런데 운영을 돌리다 보니 두 종류의 사고가 계속 났다.

---

## 2. 운영 사고 #1 — 짝 안 맞는 괄호 `do)` 한 줄에 500

어느 날 운영 ERROR 알람이 떴다. 검색 API 본문:

```json
POST /api/assets/search
{ "value": "do)" }
```

스택 트레이스 핵심:

```
java.lang.IllegalArgumentException: Unexpected: RP())
    at QueryStringParser$Parser.parse(QueryStringParser.kt:114)
    at QueryStringParser.parse(QueryStringParser.kt:289)
```

원인은 단순했다. 사용자가 검색어를 입력하다 `(`를 빼먹고 `)`만 남겨둔 채 전송한 케이스. 파서가 `Missing '('` 같은 메시지로 즉시 throw했고, Spring `@Repository` 클래스 안에서 발생한 예외라 `InvalidDataAccessApiUsageException`으로 래핑돼 `GlobalExceptionHandler`의 fallback이 `500 INTERNAL_SERVER_ERROR`로 응답한 것이다.

사용자 입장에서는 자기가 무슨 잘못을 했는지도 모른 채 빈 화면을 보고 페이지를 새로고침했을 것이다. 우리 입장에서는 "사용자 입력 오류일 뿐인데 ERROR 알람이 시도 때도 없이 울린다"는 문제였다.

### 어떻게 고쳤나 — lenient fallback

핵심은 단순하다. **파서가 실패하면 throw 대신 입력 전체를 phrase로 감싸서 평문 검색으로 떨어뜨리자.** 사용자가 의도한 검색은 거의 항상 "이 문자열을 찾고 싶다"이기 때문이다.

```kotlin
fun parse(userQuery: String): Query {
    val ast = try {
        Parser(userQuery).parse()
    } catch (_: BadRequestException) {
        // 사용자 입력에 문법 오류(짝 안 맞는 괄호 등)가 있으면
        // 평문 phrase 로 검색해 500/400 대신 정상 응답 반환.
        // 운영 사고 케이스: `do)` 입력.
        return buildDefaultQueryStringGrouped(wrapAsPhrase(userQuery))
    }
    return toQuery(ast) ?: Query.of { q -> q.matchAll { it } }
}

private fun wrapAsPhrase(s: String): String {
    val escaped = s.replace("\\", "\\\\").replace("\"", "\\\"")
    return "\"$escaped\""
}
```

테스트로 회귀를 막아두는 게 중요하다.

```kotlin
@Test
fun `짝 안 맞는 닫는 괄호는 예외 대신 평문 검색`() {
    val q = parser().parse("do)")
    val j = json(q)
    assertTrue(j.contains("\"query_string\""), "예외 대신 query_string 생성되어야 함")
}

@Test
fun `짝 안 맞는 여는 괄호도 평문 검색`() {
    val j = json(parser().parse("(unclosed"))
    assertTrue(j.contains("\"query_string\""), j)
}
```

이제 사용자가 어떤 깨진 표현식을 보내도 200 OK + (대체로 빈) 결과를 받는다. 운영 ERROR도 사라졌다.

---

## 3. 운영 사고 #2 — CIDR 표기 `211.233.70.0/26`에 `all shards failed`

다른 날, 또 다른 알람. 사용자가 IP 대역을 검색해 보려고 `211.233.70.0/26`을 입력했다. 응답은 500.

```
ElasticsearchException: ... all shards failed
Caused by: ... query string parsing exception:
  Failed to parse query [211.233.70.0/26]:
    Cannot parse '211.233.70.0/26': '/' was found
```

원인: ES `query_string`은 슬래시로 시작하는 토큰을 **정규식으로 해석**한다. `/26` 한 글자 뒤가 끝나지 않은 채라 ES 측에서 regex 컴파일이 실패한 것이다. 사용자는 "CIDR 검색"을 의도했을 뿐 정규식을 짤 의도가 없었다.

비슷한 함정의 후보들:

| 문자 | ES query_string 의미 | 사용자가 무심코 넣을 때 |
|------|---------------------|---------------------|
| `/` | 정규식 시작 | CIDR, URL 경로 |
| `[`, `]` | range 시작/끝 | UUID 옵션 표기, 배열 표기 |
| `<`, `>` | range bound | 비교, HTML tag |
| `^` | boost | 정규식 의도 외 사용 |

### 어떻게 고쳤나 — leaf 단계에서 전부 escape

ES 측에 전달하기 직전(query 문자열을 leaf 단으로 빌드할 때), 우리 표현식 문법이 의미를 부여하지 않는 ES 예약 문자를 모두 backslash로 escape한다.

```kotlin
private fun buildQueryString(rawValue: String, fields: List<String>): Query {
    return Query.of { q ->
        q.queryString { qs ->
            qs.query(escapeEsReservedChars(rawValue))
                .quoteFieldSuffix(".exact")
                .defaultOperator(Operator.And)
                .fields(fields)
        }
    }
}

private val ES_RESERVED = setOf('/', '[', ']', '<', '>', '^', '~')

private fun escapeEsReservedChars(s: String): String {
    val sb = StringBuilder(s.length + 4)
    for (ch in s) {
        if (ch in ES_RESERVED) sb.append('\\')
        sb.append(ch)
    }
    return sb.toString()
}
```

> 우리는 wildcard(`*`, `?`)는 의도적으로 보존했다. 사용자 기대 동작(`kakao*` → `kakao`로 시작) 때문이다. 어떤 문자를 escape하고 어떤 문자를 보존할지는 검색 UX에 따라 다르다 — 우리 도메인에서 "이 문자에 의미를 부여하는가?"를 기준으로 정한다.

회귀 방지 테스트:

```kotlin
@Test
fun `슬래시 포함 CIDR 표기는 ES 정규식으로 해석되지 않도록 escape`() {
    // 실제 사고 입력: "211.233.70.0/26"
    val j = json(parser().parse("211.233.70.0/26"))
    assertTrue(j.contains("\"query_string\""), j)
    // ES 쿼리에 "211.233.70.0\/26" 가 들어가야 하고
    // JSON 직렬화 시 backslash 가 한 번 더 escape 되어 "\\\\/" 로 보임
    assertTrue(
        j.contains("211.233.70.0\\\\/26"),
        "/ 가 escape 되어 ES regex 해석이 막혀야 함"
    )
}

@Test
fun `대괄호 range 표기는 평문 검색으로 처리`() {
    val j = json(parser().parse("[8000"))
    assertTrue(j.contains("\"query_string\""), j)
}

@Test
fun `별표 와일드카드는 의도적으로 보존`() {
    val j = json(parser().parse("kakao*"))
    assertTrue(j.contains("\"query\":\"kakao*\""), "wildcard 보존")
}
```

---

## 4. 정리 — 입력을 거절하지 않는 검색

두 종류의 사고는 같은 교훈을 남겼다.

> 사용자는 우리 쿼리 문법을 모른다. 모르고 있다는 사실을 모른다.

검색은 폼 validation과 다르다. 폼은 "회원가입 password 형식이 틀렸으니 다시 입력하라"고 돌려보낼 수 있지만, **검색은 항상 결과(빈 결과라도)를 반환하는 게 사용자 경험**이다. 그래서 우리 파서가 가져야 할 두 가지 원칙은 다음과 같다.

1. **syntax 오류는 throw하지 말고 평문 phrase 검색으로 fallback**한다.
2. **ES 측 예약 문자는 leaf 단에서 모두 escape**해, 사용자 입력이 정규식/range로 우연히 해석되지 않게 한다.

다만 이걸로 끝이 아니었다. 다음 편에서는 또 다른 종류의 함정 — **사용자 입력 오류가 운영 ERROR 알람을 울리는 문제**를 어떻게 분리했는지 이야기한다.

다음 글: [사용자 친화적 ES 검색 만들기 (2) — 4xx와 5xx를 분리해 운영 ERROR 알람 노이즈 없애기]({% post_url 2026-05-28-es-query-string-parser-2-4xx-classification %})
