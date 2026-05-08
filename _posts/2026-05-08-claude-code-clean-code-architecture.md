---
title: "클로드코드로 clean-code/architecture 프로젝트 구축하면서 느낀점"
date: 2026-05-08T10:00:00+09:00
categories: [AI]
tags: [claude-code, clean-code, clean-architecture, hexagonal-architecture, kotlin, spring-boot]
---

clean-code, clean-architecture로 이 meet-again 프로젝트를 구축 진행하면서 느낀점입니다.

처음에는 "AI가 짜준 코드는 일단 돌아가긴 해도, 결국 사람이 다 갈아엎어야 하는 것 아닌가" 하는 의심으로 시작했습니다. 그런데 막상 헥사고날 아키텍처 위에서 도메인 객체에 행위를 부여하고, 원시값을 포장하고, 포트와 어댑터를 분리하는 원칙을 클로드코드에게 명확히 전달하고 나니 — 생각보다 훨씬 결이 맞는 코드가 나왔습니다. 오히려 사람이 손으로 짤 때 슬쩍 타협하던 부분(빈혈 모델, getter 체이닝, 서비스에 로직 몰아넣기)을 더 엄격히 지켜주더군요.

이 글은 그 시리즈의 첫 번째 글로, **클로드코드와 함께 일하면서 클린 코드/아키텍처 원칙이 어떻게 강제되고, 어떤 부분에서 사람의 개입이 여전히 필요한지**를 정리하려고 합니다.
