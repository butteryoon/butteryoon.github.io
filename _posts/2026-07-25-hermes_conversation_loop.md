---
layout: post
comments: true
title: "Hermes 대화 루프 심층 — tool_call 하나가 실제로 타는 경로"
description: "Hermes Agent의 agent/conversation_loop.run_conversation() 내부를 열어, LLM이 내보낸 tool_call이 model_tools.handle_function_call()까지 어떻게 흐르고 컨텍스트가 영속되는지 실제 모듈·함수 경로로 추적"
img: command-title.webp
date: 2026-07-25 21:30:00 +0900
last_modified_at: 2026-07-25 21:30:00 +0900
tags: [hermes, source code, conversation loop, tool call, agent, nous research] # add tag
related: llm
categories: dev
---

[소스 트리 투어 글]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html)에서 `agent/conversation_loop.py`의 `run_conversation()`이 툴 호출 루프임을 짚었다. 이번에는 그 루프를 열어 **LLM이 `tool_calls`를 내보낸 순간부터 실제 함수가 실행되고 결과가 돌아오는까지**의 경로를 추적한다. 로컬 소스(`AppData\Local\hermes\hermes-agent`)를 직접 열어 확인했다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** `run_conversation()`이 provider 응답에서 `tool_calls`를 꺼내(`_sanitize_tool_call_arguments`로 정화) → `model_tools.handle_function_call()`(model_tools.py:1069)로 위임 → 실제 툴 함수 실행 → `tool_result`를 메시지에 붙여 다음 턴에 재투입. 턴 종료마다 `ContextEngine.on_turn_complete()` 훅(842행)이 불려 메모리/스킬이 영속된다. 즉 "툴 호출 한 번"은 **정화 → 디스패치 → 실행 → 관측** 4단계를 거친다.

## 1. 루프 안에서 tool_calls가 등장하는 지점

`conversation_loop.py`의 `run_conversation()`은 assistant 메시지에 `tool_calls`가 있으면 그걸 순회한다. 실제 코드 위치:

```text
$ grep -nE "tool_calls|repaired_tool_calls" agent/conversation_loop.py
1082:  if _m.get("role") == "assistant" and _m.get("tool_calls"):
1097:    for tc in _m["tool_calls"]
1166:  # on assistant messages with tool_calls. We handle both cases here.
1168:  repaired_tool_calls = agent._sanitize_tool_call_arguments(
1173:  if repaired_tool_calls > 0:
```

핵심은 **1168행의 `_sanitize_tool_call_arguments`** 다. LLM이 만든 인자가 strict API나 스키마와 안 맞으면 런타임이 자동 수정한다(1273행 `_should_sanitize_tool_calls`, 1295행 `_sanitize_tool_calls_for_strict_api`). 비결정적 모델 출력이라 인자 오염이 잦으니, 루프 레벨에서 정화하는 설계다.

## 2. 디스패치 — handle_function_call

정화된 `tool_calls`는 실제 실행 함수로 넘어간다. 그 끝은 `model_tools.py`:

```text
$ grep -nE "def handle_function_call" model_tools.py
1069:def handle_function_call(
        function_name: str,
        function_args: Dict[str, Any],
        task_id: Optional[str] = None,
        tool_call_id: Optional[str] = None,
        session_id: Optional[str] = None,
        turn_id: Optional[str] = None,
        ...
        enabled_tools: Optional[List[str]] = None,
        enabled_toolsets: Optional[List[str]] = None,
        disabled_toolsets: Optional[List[str]] = None,
    ) -> str:
```

시그니처가 말해주는 것: 툴 실행은 **세션/턴 단위로 격리**되고(`session_id`, `turn_id`), **toolset 화이트리스트**(`enabled_toolsets`/`disabled_toolsets`)로 어떤 툴을 쓸 수 있는지 제한된다. 즉 오케스트레이션 글에서 본 "권한 최소화"가 이 파라미터로 구현된다. `enabled_tools`가 None이면 전체, 아니면 화이트리스트만.

## 3. 실행 → 결과 재투입

`handle_function_call`이 툴 함수를 돌리고 문자열 결과를 돌려주면, `run_conversation`은 그걸 `tool_result` 메시지로 감싸 대화 히스토리에 붙인다. 그러면 다음 턴 LLM은 "툴이 이렇게 답했으니 다음엔?"을 추론한다. 이 반복이 [에이전트 학습 메타글]({{site.baseurl}}/tools/2026/07/25/hermes_agent_learning.html)에서 말한 관측→추론→행동→결과 사이클의 코드 레벨 구현이다.

## 4. 턴 완료 훅 — 영속화 지점

루프의 가장 중요한 설계는 턴이 끝날 때마다 부르는 관측 훅이다.

```text
$ grep -nE "on_turn_complete" agent/conversation_loop.py
842:  Calls the optional ContextEngine.on_turn_complete() observation hook
852:  hook = getattr(engine, "on_turn_complete", None)
860:  from agent.context_engine import ContextEngine as _CE
861:  if getattr(hook, "__func__", None) is _CE.on_turn_complete:
874:  "Context engine on_turn_complete hook failed (session=%s)",
```

`ContextEngine.on_turn_complete()`(agent/context_engine.py)가 매 턴 불린다. 여기서 메모리 요약·스킬 후보 추출·컨텍스트 선택이 일어난다. 비결정적 LLM을 "관측 가능"하게 만드는 장치 — 에이전트가 무슨 툴을 몇 번 불렀는지, 상태가 어떻게 바뀌었는지를 턴 단위로 기록한다. AI Factory 글의 "통합 평가/트레이스 리플레이"가 코드 레벨에서 이 훅으로 살아난다.

## 5. 한 툴 호출의 전체 경로

```
LLM 응답 (tool_calls)
  → run_conversation(): 꺼냄 + _sanitize_tool_call_arguments 정화
  → model_tools.handle_function_call(name, args, session_id, enabled_toolsets)
       → 실제 툴 함수 실행 (terminal/web/file/...)
       → str 결과 반환
  → run_conversation(): tool_result 메시지로 히스토리에 추가
  → (다음 턴) LLM 재추론
  → 턴 종료: ContextEngine.on_turn_complete() 영속화
```

## 마무리

"툴 호출 한 번"은 단순 호출이 아니라 **정화→디스패치(격리+화이트리스트)→실행→관측(영속)** 로 구성된다. `conversation_loop.py`의 정화 로직과 `model_tools.py:handle_function_call`의 격리 파라미터, `context_engine.py`의 턴 훅이 합쳐져 "안전하고 관측 가능한" 에이전트 루프가 된다. 다음 글(C편)에서는 같은 에이전트 코어가 텔레그램/디스코드에서 어떻게 도는지 — `plugins/platforms/telegram/`과 `gateway/delivery.py`의 `DeliveryRouter`를 따라가 본다.

## 참고

- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent){:target="_blank"}
- [소스 트리 투어 (관련글)]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html)
- [에이전트 학습 메타글 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_agent_learning.html)
- [스킬 오케스트레이션 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html)
