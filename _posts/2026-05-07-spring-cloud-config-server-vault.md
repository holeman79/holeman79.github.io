---
title: "Spring Cloud Config Server + HashiCorp Vault로 애플리케이션 설정 중앙화하기"
date: 2026-05-07T15:00:00+09:00
categories: [Infra]
tags: [spring, vault, config-server, kubernetes, devops, secret-management]
---

## TL;DR

- 애플리케이션 설정을 코드와 분리해서 관리할 때, **비민감 설정은 Git**에, **민감 정보(비밀번호·토큰)는 Vault**에 두면 변경 추적과 보안을 동시에 챙길 수 있다.
- Spring Cloud Config Server가 두 백엔드(Git + Vault)를 한꺼번에 묶어 클라이언트 앱에 노출하므로, 앱 입장에서는 **단일 엔드포인트**만 바라본다.
- HashiCorp Vault는 Sealed/Unsealed 모델, Shamir 분할 키, KV v2 시크릿 엔진, Raft HA로 시크릿을 안전하게 보관한다.
- 운영에서 가장 자주 마주치는 함정은 **앱 재시작 없이는 설정이 반영되지 않는다**는 것과, **Vault Pod가 재시작되면 sealed 상태로 시작**한다는 것 두 가지다.

## 왜 중앙 설정 관리가 필요한가

서비스를 K8s에 띄우다 보면 설정값이 갑자기 늘어난다. DB URL, 비밀번호, 외부 API 토큰, 큐 호스트, feature flag…. 보통 처음에는 ConfigMap·Secret을 직접 마운트하지만, 서비스가 늘어나면 다음과 같은 문제가 생긴다.

1. **중복** — 같은 DB 비밀번호가 여러 ConfigMap/Secret에 흩어진다.
2. **보안 사각지대** — Secret이 base64 인코딩만 되어 있을 뿐 평문에 가깝다. 누가 언제 어떤 값을 바꿨는지도 추적이 어렵다.
3. **변경 절차의 비대칭** — 비민감 설정은 Git PR로 관리하고 싶고, 민감 정보는 PR로 노출하고 싶지 않다. 두 흐름을 분리해야 한다.

이걸 풀어주는 조합이 **Spring Cloud Config Server + HashiCorp Vault**다.

| 분류 | 저장소 | 변경 절차 | 예시 |
|---|---|---|---|
| 비민감 설정 | Git Repo | PR + merge | DB 호스트, 포트, feature flag |
| 민감 정보 | Vault | UI/CLI에서 직접 수정 | 비밀번호, 토큰, 인증서 |

## 아키텍처 개요

```
┌────────────────────────────────────────────────────┐
│                     K8s Cluster                    │
│                                                    │
│  ┌──────────┐   ┌────────────────┐   ┌──────────┐  │
│  │  Vault   │   │ Config Server  │   │ Git Repo │  │
│  │ (Raft x3)│◄──│   (Replica x2) │──►│          │  │
│  └──────────┘   └───────┬────────┘   └──────────┘  │
│                         │                          │
│              ┌──────────┼──────────┐               │
│         ┌────┴────┐ ┌───┴───┐ ┌────┴────┐          │
│         │  App A  │ │ App B │ │  App C  │          │
│         └─────────┘ └───────┘ └─────────┘          │
└────────────────────────────────────────────────────┘
```

핵심 흐름은 단순하다.

1. 앱(Config Client)이 부팅하면서 **Config Server**에 자기 이름과 프로파일을 알려준다.
2. Config Server가 **Git Repo에서 비민감 설정**, **Vault에서 시크릿**을 각각 가져와 병합한다.
3. 병합된 결과를 앱에 응답으로 돌려준다.
4. 앱은 받은 값을 `Environment`에 채워 부팅을 완료한다.

## 구성 요소

### Spring Cloud Config Server

- 역할: Git/Vault 같은 외부 백엔드를 추상화해서, 클라이언트가 단일 엔드포인트만 보게 만든다.
- 핵심 설정 (server 측):

```yaml
# config-server: application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://git.example.com/org/config-repo
          default-label: main
          search-paths: '{application}'
        vault:
          host: vault.my-namespace.svc.cluster.local
          port: 8200
          scheme: http
          backend: secret
          kv-version: 2
```

- `search-paths: '{application}'`로 두면 앱 이름 prefix별로 Git 디렉토리를 분리할 수 있다.
- Git 백엔드는 **요청이 올 때마다 최신 커밋을 pull** 해서 응답하므로, Git에서 push만 하면 별도 reload가 필요 없다.

### Git Config Repository

```
config-repo/
├── application.yml              ← 모든 앱 공통
├── app-a/
│   ├── app-a.yml                ← app-a 공통
│   ├── app-a-dev.yml            ← dev 프로파일
│   └── app-a-prd.yml            ← prd 프로파일
└── app-b/
    └── ...
```

- 파일명은 `{spring.application.name}-{profile}.yml` 규칙을 따른다.
- **비밀번호·토큰은 절대 들어가면 안 된다**(공개 저장소가 아니어도 마찬가지). 그건 Vault 몫이다.

### HashiCorp Vault

#### 핵심 개념

| 개념 | 설명 |
|---|---|
| **Sealed / Unsealed** | Vault는 정지/재시작 후 항상 **sealed 상태**로 시작. 이 상태에선 어떤 API 호출도 거부된다. unseal key를 입력해야 정상 작동. |
| **Shamir 분할 키** | unseal에 필요한 master key를 N개 조각으로 쪼개고, T개 이상 모이면 복원하는 방식. 운영은 보통 5-of-3. |
| **Token** | API 호출 인증 수단. root token은 모든 권한을 가지며 분실/노출 시 피해가 크다. |
| **Policy** | path 단위로 read/write/list 같은 capability를 부여하는 ACL. token에 매핑된다. |
| **KV v2 Secret Engine** | key-value 형태로 시크릿을 보관. **버전 관리 + 이전 값 복원**을 지원하는 게 v1과의 차이. |

#### 시크릿 경로 구조

```
secret/
├── application/             ← 모든 앱 공통 시크릿
│   └── {profile}
└── {app-name}/              ← 앱별 시크릿
    └── {profile}
```

`spring.application.name`이 Vault path와 직결되므로, **앱 이름을 바꾸면 시크릿 path도 같이 옮겨야 한다**.

#### KV v2의 버전 관리

KV v2는 시크릿을 수정할 때마다 새 버전을 만들고 이전 값을 보존한다. UI의 *Version History*에서 과거 값을 조회·복원할 수 있어, 잘못된 값을 push했을 때 빠르게 롤백할 수 있다.

## 설정 우선순위

여러 소스에 같은 키가 있을 때 적용 순서:

1. **Vault** (시크릿) — 최우선
2. **Git의 환경별 설정** (예: `app-a-dev.yml`)
3. **Git의 앱 공통 설정** (예: `app-a.yml`)
4. **Git의 전역 공통 설정** (`application.yml`)

같은 키가 여러 곳에 있으면 **Vault 값이 항상 이긴다.** 비민감 placeholder를 Git에 두고 운영 값은 Vault에서 덮어쓰는 패턴이 자연스럽다.

## 변경사항 반영 방법

### Case 1: Vault 시크릿을 수정한 경우

1. Vault UI/CLI에서 값 수정 (`vault kv patch ...`)
2. **Config Server는 재시작 불필요** — 매 요청마다 Vault에서 실시간 조회
3. **앱은 반드시 재시작** — 앱은 부팅 시점에만 Config Server에서 설정을 가져온다

### Case 2: Git Config Repo를 수정한 경우

1. Git Repo에서 yml 수정 → `main`에 push
2. **Config Server는 재시작 불필요** — 다음 요청 시 자동으로 최신 커밋을 pull
3. **앱은 반드시 재시작** — Vault와 동일

> 요점: 어느 쪽을 수정하든 **앱만 재시작**하면 된다. Config Server는 건드릴 필요가 없다.

이게 처음에 가장 헷갈리는 부분이다. "값 바꿨는데 왜 안 먹지?" → 거의 항상 앱 재시작을 안 한 것이다. Spring Cloud Bus + 메시지 큐로 `/refresh`를 자동 트리거할 수도 있지만, 추가 인프라가 필요하므로 작은 팀에서는 K8s `rollout restart`가 더 간단하다.

## 고가용성 (HA)

| 구성 요소 | Replicas | HA 방식 | 장애 허용 |
|---|---|---|---|
| Vault | 3 | Raft 합의 (Leader 1 + Follower 2) | 1개 노드 |
| Config Server | 2 | K8s Deployment (Stateless) | 1개 노드 |

### Vault Raft 동작

- 3개 Pod 중 1개가 Leader, 2개가 Follower
- Leader가 죽으면 나머지 2개가 **과반수(2/3)** 를 충족하여 새 Leader 자동 선출
- 1개 노드 장애까지는 자동 복구. 2개 이상 장애가 동시에 나면 수동 개입 필요

```bash
vault operator raft list-peers
# Node     State     Voter
# vault-0  leader    true
# vault-1  follower  true
# vault-2  follower  true
```

### 리소스 사용량 (실측 예시)

흥미로운 점은 5개 Pod 합산이 매우 가볍다는 것이다.

| Pod | 역할 | CPU | Memory |
|---|---|---|---|
| vault-0 | Leader | 42m | 52Mi |
| vault-1 | Follower | 11m | 40Mi |
| vault-2 | Follower | 12m | 41Mi |
| config-server-1 | Active | 25m | 209Mi |
| config-server-2 | Active | 20m | 215Mi |
| **합계** | | **110m** | **557Mi** |

작은 팀에서 운영해도 부담이 적다.

## 운영 시 주의점

### 1. Vault Pod 재시작 → sealed 상태

가장 자주 만나는 함정. **3개 Pod 모두 각각 unseal**해야 한다.

```bash
kubectl exec vault-0 -- vault operator unseal <unseal-key>
kubectl exec vault-1 -- vault operator unseal <unseal-key>
kubectl exec vault-2 -- vault operator unseal <unseal-key>
```

K8s 노드 reschedule, OOM, 수동 restart 어느 쪽이든 sealed로 떨어진다. **자동 unseal**(KMS·Cloud HSM 통합)을 쓸 수 있는 환경이라면 켜두는 걸 권장한다. Shamir보다 운영 비용이 훨씬 낮다.

### 2. Unseal key / Root token 보관

- **분실 = Vault 데이터 영구 접근 불가.** 초기화 후 다시 채워야 한다.
- 채팅·티켓·위키 본문에 평문으로 적지 않는다. 1Password 같은 시크릿 매니저나 sealed envelope에 분산 보관.
- root token은 평소엔 사용하지 않고, 일상 작업은 별도의 `secret-admin` 같은 제한 정책 토큰으로 한다.

### 3. spring.application.name = 시크릿 경로 키

`spring.application.name`이 Vault/Git path와 1:1로 매칭된다. 앱 이름을 바꾸면:

- Vault에서 `secret/old-name/*` → `secret/new-name/*` 으로 이동
- Git에서 `old-name-*.yml` → `new-name-*.yml` 으로 이름 변경

둘 중 하나라도 빼먹으면 부팅이 깨진다.

### 4. Config Server를 모든 앱이 써야 하는 건 아니다

배치 잡, 단순 sidecar처럼 시크릿 의존성이 거의 없는 앱은 굳이 Config Client를 붙이지 않고 K8s Secret/ConfigMap으로 직접 주입해도 된다. 일관성보다 단순함이 더 큰 가치일 때가 있다.

## 마무리

설정 관리는 처음엔 "그냥 ConfigMap 쓰면 되지" 싶지만, 서비스가 늘어나면서 분명히 비용이 올라간다.

- 같은 비밀번호가 여러 곳에 박혀 있어 회전이 두려워지고
- 누가 언제 어떤 값을 바꿨는지 추적이 안 되고
- 비민감 설정도 PR 없이 직접 수정되고

이런 신호가 보인다면, Config Server + Vault 도입을 검토해볼 시점이다. 초기 학습 곡선은 있지만 **시크릿 회전·감사 추적·환경별 분리**가 한 번에 풀린다는 게 가장 큰 매력이다.

핵심 정리:

- **Git은 비민감, Vault는 민감 정보** — 변경 절차를 분리한다.
- **Config Server는 단일 진입점** — 앱은 백엔드를 모른다.
- **앱만 재시작하면 된다** — Config Server는 건드릴 필요 없다.
- **Vault는 sealed 모델** — Pod 재시작 시 unseal이 필요하다는 점만 잊지 말자.
