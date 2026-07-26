---
layout: post
comments: true
title: "Hermes 게이트웨이 아키텍처 — 같은 코어가 텔레그램·디스코드에서 도는 법"
description: "Hermes Agent가 하나의 에이전트 코어를 텔레그램/디스코드/슬랙 등 멀티플랫폼에서 어떻게 라우팅하는지 — plugins/platforms/telegram, gateway/delivery.DeliveryRouter, profile_routing 구조를 실제 경로로 해설"
img: command-title.webp
date: 2026-07-25 22:00:00 +0900
last_modified_at: 2026-07-25 22:00:00 +0900
tags: [hermes, gateway, telegram, discord, multi-platform, agent, nous research] # add tag
related: llm
categories: dev
---

[대화 루프 심층 글]({{site.baseurl}}/dev/2026/07/25/hermes_conversation_loop.html)에서 `agent/` 코어 루프를 추적했다. 이번에는 **같은 코어가 텔레그램·디스코드·슬랙에서 어떻게 도는지** — 게이트웨이 계층을 따라간다. 로컬 소스(`AppData\Local\hermes\hermes-agent`)를 직접 열어 확인했다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** 플랫폼별 로직은 `plugins/platforms/<name>/`(텔레그램은 `adapter.py`+`telegram_network.py`+`plugin.yaml`)에 격리되고, 발송 추상화는 `gateway/delivery.py`의 `DeliveryRouter`(deliver/_deliver_to_platform/_deliver_local)가 맡는다. 프로필(격리된 설정/세션/메모리) 라우팅은 `gateway/profile_routing.py`의 `ProfileRoute`가 담당. 즉 **코어는 하나, 차이는 플러그인+라우터** 다.

## 1. 플랫폼은 플러그인으로 분리

흥미로운 점: 텔레그램은 `gateway/platforms/`가 아니라 **`plugins/platforms/telegram/`** 에 있다.

```text
$ ls plugins/platforms/telegram/
__init__.py
adapter.py          # 에이전트 코어 ↔ 텔레그램 메시지 변환
plugin.yaml         # 플러그인 메타(이름/권한/진입점)
telegram_ids.py     # chat/user id 파싱
telegram_network.py # 네트워크(폴링/웹훅) 전송
```

즉 채널 연동은 **코어가 아니라 플러그인**이다. `gateway/platforms/`는 `base.py`(공통 베이스)와 `ADDING_A_PLATFORM.md`(신규 플랫폼 추가 가이드)만 두고, 실제 구현은 `plugins/`로 나뉜다. 핵심은 `agent/`를 건드리지 않고 채널을 추가할 수 있다는 것 — 개방형 확장 모델.

## 2. 발송 추상화 — DeliveryRouter

에이전트 응답이 사용자에게 가는 경로는 `gateway/delivery.py`의 `DeliveryRouter`가 추상화한다.

```text
$ grep -nE "def (route|deliver|_deliver|send|resolve)" gateway/delivery.py
74:   async def send(
92:   def resolve_delivery_transport(
318:  async def deliver(
393:  def _deliver_local(
460:  async def _deliver_to_platform(
```

- `resolve_delivery_transport`(92): 대상(chat_id 등)을 보고 어느 전송 방식인지 결정
- `deliver`(318): 진입점 — 로컬(터미널/TUI)인지 플랫폼인지 분기
- `_deliver_local`(393): 로컬 표시
- `_deliver_to_platform`(460): 플랫폼 어댑터(`adapter.py`)로 위임

즉 에이전트 코어는 "답을 만들었다"까지만 책임지고, **어디로 보낼지는 DeliveryRouter가 결정**한다. CLI·데스크톱·텔레그램이 같은 `deliver()`를 쓰되 내부 분기만 다르다.

## 3. 프로필 라우팅 — 격리 경계

여러 프로필(독립 설정/세션/메모리)을 쓸 때 어느 프로필이 메시지를 받는지는 `gateway/profile_routing.py`가 정한다.

```text
$ grep -nE "class ProfileRoute|def (parse_profile_routes|match_profile_route)" gateway/profile_routing.py
51:  class ProfileRoute:
105: def parse_profile_routes(raw):
154: def match_profile_route(
```

`ProfileRoute`(51)가 라우팅 규칙 하나를, `match_profile_route`(154)가 들어온 메시지를 규칙과 매칭해 적절한 프로필로 보낸다. 메모리에 저장한 "프로필별 격리" 원칙이 코드 레벨에서 이 클래스로 구현된다.

## 4. 전체 흐름

```
사용자(텔레그램) → plugins/platforms/telegram/adapter.py (수신 변환)
  → gateway/profile_routing.ProfileRoute (프로필 매칭)
  → agent/conversation_loop.run_conversation() (코어 루프)
  → agent/model_tools.handle_function_call() (툴 실행)
  → gateway/delivery.DeliveryRouter.deliver() (발송 결정)
  → _deliver_to_platform → telegram/adapter.py (송신 변환)
  → 사용자(텔레그램)
```

핵심: **가운데 `agent/` 세 줄만 빼면 나머지는 모두 플랫폼/라우팅 계층**이다. 코어는 플랫폼을 모른다.

## 마무리

Hermes의 멀티플랫폼은 "거대한 분기문"이 아니라 **플러그인 격리(`plugins/platforms/`)+발송 추상화(`DeliveryRouter`)+프로필 라우팅(`ProfileRoute`)** 로 풀린다. 그래서 [설치기 글]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html)에서 본 "터미널·데스크톱·메신저가 같은 에이전트 코어" 문구가 코드 레벨에서 성립한다. 소스 트리 투어(A) → 대화 루프(B) → 게이트웨이(C)로 이어진 이 3부작은, Hermes가 "모듈 경로로 읽히는 에이전트"임을 보여준다.

## 참고

- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent){:target="_blank"}
- [소스 트리 투어 (관련글)]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html)
- [대화 루프 심층 (관련글)]({{site.baseurl}}/dev/2026/07/25/hermes_conversation_loop.html)
- [Hermes Agent 설치부터 설정 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html)
