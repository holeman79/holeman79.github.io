---
title: "Spring Cloud Config Server + Vault로 중앙 설정 관리하기"
date: 2026-04-12T13:00:00+09:00
categories: [Spring]
tags: [spring-cloud-config, vault, hashicorp-vault, config-server, secrets-management]
toc: true
toc_sticky: true
---

## 왜 중앙 설정 관리가 필요한가

MSA 환경에서 각 앱마다 `application.yml`에 DB 접속 정보, API 토큰 등을 넣어두면 다음 문제가 생긴다.

- **설정 변경 시 모든 앱을 재빌드/재배포** 해야 함
- **민감 정보(비밀번호, 토큰)가 Git에 노출**됨
- **환경별(dev/stg/prod) 설정 관리가 파편화**됨

이를 해결하는 구조가 **Spring Cloud Config Server + HashiCorp Vault** 조합이다.

## 아키텍처 개요

```
┌──────────────────────────────────────────────────┐
│                   K8s Cluster                     │
│                                                   │
│  ┌────────────┐   ┌────────────────┐   ┌────────┐│
│  │   Vault    │◄──│ Config Server  │──►│Git Repo││
│  │ (시크릿)   │   │  (중앙 허브)    │   │(설정)  ││
│  │            │   │                │   │        ││
│  │ DB 비밀번호│   │ Backend:       │   │ URL    ││
│  │ API 토큰   │   │  Git + Vault   │   │ 포트   ││
│  │ username   │   │                │   │ 타임아웃││
│  └────────────┘   └───────┬────────┘   └────────┘│
│                           │                       │
│          ┌────────────────┼────────────────┐      │
│          │                │                │      │
│     ┌────┴─────┐    ┌────┴─────┐    ┌────┴─────┐│
│     │  App A   │    │  App B   │    │  App C   ││
│     │ (web)    │    │(subscriber)   │ (batch)  ││
│     └──────────┘    └──────────┘    └──────────┘│
└──────────────────────────────────────────────────┘
```

역할 분담이 명확하다:

| 구성 요소 | 역할 | 저장 대상 |
|-----------|------|----------|
| **Vault** | 시크릿 관리 | DB 비밀번호, API 토큰, 인증 정보 |
| **Git Repo** | 비민감 설정 관리 | URL, 포트, 타임아웃, 기능 플래그 |
| **Config Server** | 중앙 허브 | Git + Vault를 합쳐서 앱에 전달 |

## 1. Spring Cloud Config Server

Config Server는 설정을 제공하는 **중앙 서버**다. 앱이 기동할 때 Config Server에 HTTP 요청을 보내면, Git과 Vault에서 설정을 합쳐서 응답한다.

### 동작 원리

```
앱 기동 시:
  GET http://config-server:8888/{application}/{profile}

Config Server:
  1. Git Repo에서 {application}-{profile}.yml 읽기
  2. Vault에서 secret/{application}/{profile} 읽기
  3. 두 설정을 병합하여 응답
```

예를 들어 `my-app`이 `dev` 프로필로 기동하면:

```
GET http://config-server:8888/my-app/dev

Config Server 응답 (Git + Vault 병합):
{
  "spring.datasource.url": "jdbc:mysql://db-host:3306/mydb",   ← Git
  "spring.datasource.password": "s3cret!",                     ← Vault
  "server.port": 8080                                           ← Git
}
```

### 앱에 Config Client 적용

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-config")
}
```

```yaml
# application.yml
spring:
  application:
    name: my-app    # Config Server 조회 키
  config:
    import: optional:configserver:http://config-server:8888
```

`spring.application.name`이 Config Server가 Git/Vault 경로를 매칭하는 핵심 키다.

## 2. Git Config Repository

비민감 설정을 Git으로 관리한다. 파일명 규칙으로 앱과 환경을 구분한다.

### 파일 구조

```
config-repo/
├── application.yml           # 전체 앱 공통
├── application-dev.yml       # 전체 앱 공통 DEV
├── application-stg.yml       # 전체 앱 공통 STG
├── application-prd.yml       # 전체 앱 공통 PROD
├── my-app.yml                # my-app 전용 공통
├── my-app-dev.yml            # my-app 전용 DEV
└── my-app-prd.yml            # my-app 전용 PROD
```

### 파일명 매칭 규칙

Config Server는 앱이 요청하면 다음 순서로 파일을 찾아 병합한다:

```
요청: GET /my-app/dev

매칭 순서 (나중에 매칭된 값이 우선):
  1. application.yml          (전체 공통)
  2. application-dev.yml      (전체 공통 DEV)
  3. my-app.yml               (앱 전용 공통)
  4. my-app-dev.yml           (앱 전용 DEV)
```

이 구조 덕분에 공통 설정은 `application.yml`에 한 번만 작성하고, 앱별/환경별로 다른 부분만 오버라이드할 수 있다.

## 3. HashiCorp Vault

민감 정보(비밀번호, 토큰 등)를 안전하게 저장하고 관리하는 시크릿 관리 도구다.

### 시크릿 경로 구조

```
secret/
├── application/          # 전체 앱 공통 시크릿
│   ├── dev
│   ├── stg
│   └── prd
├── my-app/               # my-app 전용 시크릿
│   ├── dev
│   ├── stg
│   └── prd
└── another-app/          # another-app 전용
    ├── dev
    └── prd
```

Git과 동일한 매칭 규칙이다. `application/{profile}`은 모든 앱에 적용되고, `{app-name}/{profile}`은 해당 앱에만 적용된다.

### Vault의 핵심 특징

**KV v2 (Key-Value Version 2) Secrets Engine:**
- 시크릿을 key-value 쌍으로 저장
- **버전 관리** — 수정할 때마다 새 버전이 생성되고, 이전 버전 조회/롤백 가능
- 실수로 덮어써도 이전 버전으로 복구 가능

**Seal/Unseal 메커니즘:**
- Vault는 기동 시 **sealed(봉인)** 상태로 시작
- Unseal Key로 **unseal(개봉)**해야 시크릿에 접근 가능
- 보안 목적 — 서버가 탈취되어도 Unseal Key 없이는 데이터 접근 불가

```
Vault Pod 재시작
  → Sealed 상태 (시크릿 접근 불가)
  → Unseal Key 입력
  → Unsealed 상태 (정상 운영)
```

### Raft HA (고가용성)

운영 환경에서는 Vault를 3개 노드로 구성한다.

```
vault-0 (Leader)   ←── 읽기/쓰기 처리
vault-1 (Follower) ←── 데이터 복제, Leader 장애 시 승격
vault-2 (Follower) ←── 데이터 복제, Leader 장애 시 승격
```

**Raft 합의 알고리즘:**
- Leader가 죽으면 나머지 2개가 과반수(2/3)를 충족하여 새 Leader를 **자동 선출**
- 1개 노드 장애까지 자동 복구
- 2개 이상 장애 시 과반수 미달로 수동 개입 필요

## 4. 설정 우선순위

동일한 키가 여러 소스에 있으면 다음 순서로 우선 적용된다:

```
1순위: Vault (시크릿)           ← 최우선
2순위: Git (앱별 환경 설정)      ← my-app-dev.yml
3순위: Git (앱별 공통 설정)      ← my-app.yml
4순위: Git (전체 환경 설정)      ← application-dev.yml
5순위: Git (전체 공통 설정)      ← application.yml
```

예를 들어 `spring.datasource.password`가 Git과 Vault 모두에 있으면, **Vault 값이 적용**된다.

## 5. 설정 변경 반영 흐름

### 시크릿 변경 (Vault)

```
1. Vault UI/CLI에서 값 수정
2. Config Server 재시작 불필요 (매 요청마다 Vault에서 실시간 조회)
3. 앱 재시작 필요 (앱은 기동 시에만 Config Server에서 설정을 가져옴)
```

### 비민감 설정 변경 (Git)

```
1. config-repo에서 yml 수정 후 push
2. Config Server 재시작 불필요 (Git의 최신 커밋을 자동으로 pull)
3. 앱 재시작 필요
```

**핵심: Vault든 Git이든 설정을 수정한 후에는 Config Server는 건드릴 필요 없고, 앱만 재시작하면 된다.**

```
앱 재시작
  → Config Server에 HTTP 요청
  → Config Server가 Git + Vault에서 최신 값 조회
  → 앱에 전달
  → 앱이 새 설정으로 기동
```

## 6. 전체 흐름 다이어그램

```
개발자가 설정 변경:
  ├─ 비밀번호 변경 → Vault UI에서 수정
  └─ URL 변경    → Git Repo에서 yml 수정 후 push

앱 재시작 (kubectl rollout restart):
  │
  ├─ 앱: GET /my-app/dev → Config Server
  │
  ├─ Config Server:
  │   ├─ Git Repo에서 my-app-dev.yml 읽기
  │   ├─ Vault에서 secret/my-app/dev 읽기
  │   └─ 병합하여 응답
  │
  └─ 앱: 받은 설정으로 DataSource, 커넥션 풀 등 초기화
```

## 정리

| 항목 | 설명 |
|------|------|
| **Config Server** | Git + Vault를 합쳐서 앱에 설정을 제공하는 중앙 허브 |
| **Git Repo** | 비민감 설정 (URL, 포트, 타임아웃) — 파일명으로 앱/환경 구분 |
| **Vault** | 민감 정보 (비밀번호, 토큰) — KV v2로 버전 관리, Raft HA로 고가용성 |
| **우선순위** | Vault > Git 앱별 > Git 공통 |
| **변경 반영** | Config Server는 항상 최신 값을 조회, 앱만 재시작하면 됨 |
| **보안** | Vault Seal/Unseal로 물리적 탈취 방어, 토큰 정책으로 접근 권한 분리 |
