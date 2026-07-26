---
layout: post
comments: true
title: "Hermes Agent 소스 트리 투어 — 명령어 하나가 실제로 타는 모듈"
description: "Hermes Agent GitHub 저장소를 클론한 뒤 어디부터 읽을지 — cli.py 진입점부터 subcommands, agent 코어 런타임(conversation_loop), gateway, cron까지 실제 모듈 경로로 따라가는 소스 맵"
img: command-title.webp
date: 2026-07-25 21:00:00 +0900
last_modified_at: 2026-07-25 21:00:00 +0900
tags: [hermes, source code, architecture, agent, cli, github, nous research] # add tag
related: llm
categories: dev
---

[Hermes Agent](https://github.com/NousResearch/hermes-agent)를 깔고 쓰다 보면 "이 명령어가 실제로 어떤 코드를 타는가"가 궁금해진다. 이 글은 저장소를 클론한 독자가 **소스 맵을 갖고 어디부터 읽을지** 알게 하는 트리 투어다. 로컬에 설치된 소스(`AppData\Local\hermes\hermes-agent`)를 직접 열어 경로를 확인했다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** Hermes 소스는 `cli.py`(789KB 진입점) → `hermes_cli/subcommands/`(40개 CLI 명령) → `agent/`(120개 코어 모듈) → `gateway/`(멀티플랫폼) → `cron/`(스케줄러)로 계층화된다. 명령어 하나(예: `hermes skills`)는 `subcommands/skills.py` 파서에서 받아 `agent/curator.py`로 위임되고, 대화는 `agent/conversation_loop.py`의 `run_conversation()`이 툴 호출 루프를 돈다. 읽기 순서: `cli.py` 헤더 → `subcommands/` → `agent/conversation_loop.py` → `gateway/`.

## 1. 진입점 — cli.py

저장소 루트의 `cli.py`가 메인 진입점이다(약 789KB). 헤더 주석부터 흥미로운 점이 있다.

```text
$ head -12 cli.py
#!/usr/bin/env python3
"""Hermes Agent CLI - Interactive Terminal Interface
...
# IMPORTANT: hermes_bootstrap must be the very first import — UTF-8 stdio
# on Windows.
try:
    import hermes_bootstrap
except ModuleNotFoundError:
    pass
```

첫 import가 `hermes_bootstrap` — Windows UTF-8 stdio 설정을 위한 강제 import다. `hermes update` 중 `git-reset`으로 새 코드가 내려왔는데 venv 설치가 안 끝난 상황을 `ModuleNotFoundError`로 우아하게 fallback한다. 즉 **설치 중간 상태까지 고려한 방어 코드**가 진입점에 박혀 있다.

## 2. 명령어 라우팅 — hermes_cli/subcommands/

실제 명령어는 `hermes_cli/subcommands/`에 모듈 하나당 하나씩 있다. 현재 40개.

```text
$ ls hermes_cli/subcommands/ | wc -l
40

$ ls hermes_cli/subcommands/
acp.py auth.py backup.py config.py console.py cron.py dashboard.py
debug.py doctor.py dump.py gateway.py gui.py hooks.py import_cmd.py
insights.py login.py logout.py logs.py mcp.py memory.py model.py
pairing.py plugins.py profile.py prompt_size.py security.py setup.py
skills.py skin.py slack.py status.py tools.py uninstall.py update.py
version.py webhook.py
```

각 파일은 `build_<cmd>_parser(subparsers)` 형태로 자기 서브커맨드를 붙인다. 예: `skills.py`는 `skills browse/search/install/audit`를 `subparsers.add_parser("skills")` 아래 달고, 실제 로직은 `agent/curator.py`로 위임한다. 즉 **CLI 계층은 얇은 라우터** — 비즈니스 로직은 `agent/`에 있다.

## 3. 코어 런타임 — agent/

`agent/` 디렉토리에 핵심 모듈 120개가 있다. 대화 루프, 컨텍스트, 위임, 스킬 생명주기 등.

```text
$ ls agent/ | head -20
conversation_loop.py   # 툴 호출 루프 (작업 알고리즘의 실체)
context_engine.py      # 컨텍스트/메모리 관리
context_compressor.py  # 컨텍스트 압축
delegation_context.py  # delegate_task 위임 컨텍스트
curator.py              # 스킬 생명주기 (skills 명령 백엔드)
error_classifier.py    # 외부 루프 에러 분류
anthropic_adapter.py   # provider 어댑터
azure_identity_adapter.py
bedrock_adapter.py
codex_adapter.py
```

### 3.1 대화 루프 — conversation_loop.run_conversation

`agent/conversation_loop.py`의 `run_conversation()`(약 880행부터)이 에이전트의 심장이다. 이전 [에이전트 학습 메타글]({{site.baseurl}}/tools/2026/07/25/hermes_agent_learning.html)에서 설명한 "관측→추론→행동→결과" 사이클이 여기서 돌아간다. 모듈 docstring과 실제 카운터를 보면 루프 구조가 드러난다.

```text
$ grep -nE "Main conversation loop|_turn_exit_reason" agent/conversation_loop.py
976:  # Main conversation loop counters (pure locals consumed by the loop below)
995:  _turn_exit_reason = "unknown"  # Diagnostic: why the loop ended
```

턴 종료 이유(`_turn_exit_reason`)를 진단용으로 추적하는 점이 눈에 띈다 — 비결정적 LLM 루프라 "왜 멈췄나"를 남기는 게 운영상 필수다([AI Factory 글]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)의 통합 평가/트레이스와 같은 맥락). 또한 턴 완료마다 `ContextEngine.on_turn_complete()` 관측 훅을 부르는 구조(842행 근처)여서, 메모리/스킬이 매 턴 영속화될 수 있다.

### 3.2 provider 어댑터 분리

`*_adapter.py`가 provider별 변환기를 담는다(Anthropic/Azure Bedrock/Codex/OpenAI). 프로바이더 무관성([설치기 글]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html)에서 강조한 차별점)은 이 어댑터 계층으로 구현된다 — 하나의 에이전트 코어가 여러 벤더 API를 같은 인터페이스로 본다.

## 4. 멀티플랫폼 — gateway/

같은 에이전트 코어가 텔레그램/디스코드/슬랙 등에서 도는 건 `gateway/` 덕분이다.

```text
$ ls gateway/
platforms/        # 플랫폼별 어댑터
delivery.py       # 발송(전달) 계층
kanban_watchers.py
memory_monitor.py
profile_routing.py
```

`platforms/`에 채널별 구현이, `delivery.py`에 발송 추상화가, `profile_routing.py`에 프로필(격리된 설정/세션/메모리) 라우팅이 있다. 즉 CLI·데스크톱·메신저가 **동일한 `agent/` 코어**를 공유하고, 차이는 `gateway/` 계층에만 있다.

## 5. 백그라운드 시스템 — cron/

정기 작업은 `cron/` 디렉토리.

```text
$ ls cron/
jobs.py          # 작업 정의/저장
scheduler.py      # 스케줄 틱
executions.py     # 실행 이력
lifecycle_guard.py
scheduler_provider.py
```

`subcommands/cron.py`가 파서만 들고 있고, 실제 로직은 `cron/jobs.py`·`scheduler.py`로 위임된다(2장 라우팅 규칙과 동일). [오케스트레이션 글]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html)에서 다룬 `cronjob`이 여기서 돌아간다.

## 6. 읽기 순서 제안

저장소를 클론했다면 이 순서를 권한다.

```
1. cli.py (헤더 ~50행)        — 부트스트랩·진입점
2. hermes_cli/subcommands/     — 명령어가 어떻게 파서로 쪼개지는가
3. agent/conversation_loop.py  — run_conversation() 툴 호출 루프 (핵심)
4. agent/context_engine.py     — 메모리/컨텍스트가 어떻게 영속되는가
5. agent/curator.py            — 스킬 생명주기 (skills 명령 백엔드)
6. gateway/                     — 멀티플랫폼 라우팅
7. cron/                        — 백그라운드 스케줄러
```

`docs/` 디렉토리에도 `design/`, `kanban/`, `observability/`, `security/` 설계 문서가 있어, 코드 읽기 전에 먼저 훑어보는 게 좋다(`docs/design/profile-builder.md` 등).

## 마무리

Hermes 소스는 "거대한 단일 파일"이 아니라 **얇은 CLI 라우터(`subcommands/`) + 두터운 코어(`agent/`) + 플랫폼 격리(`gateway/`) + 백그라운드(`cron/`)** 로 계층화된다. 명령어 하나가 실제로 타는 경로(`skills` → `subcommands/skills.py` → `agent/curator.py`)를 알면, "에이전트가 무슨 일을 하는가"가 추상이 아니라 **디스크상의 구체적 모듈 경로**로 읽힌다. 오늘 쓴 [에이전트 학습 메타글]({{site.baseurl}}/tools/2026/07/25/hermes_agent_learning.html)의 "기억 3계층"도 사실 `agent/context_engine.py`(런타임)와 `memories/MEMORY.md`(영속)의 분리를 코드 레벨에서 보는 것과 같다. 다음 글에서는 `conversation_loop.run_conversation()` 내부를 열어 툴 호출 한 번이 실제로 어떤 함수 호출을 거치는지 심층 추적해볼 예정이다.

## 참고

- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent){:target="_blank"}
- [Hermes 공식 문서](https://hermes-agent.nousresearch.com/docs/){:target="_blank"}
- [에이전트 학습 메타글 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_agent_learning.html)
- [스킬 오케스트레이션 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html)
- [NVIDIA AI Factory — 통합 평가/트레이스 (관련글)]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)
