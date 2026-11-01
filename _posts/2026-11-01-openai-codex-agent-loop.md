---
layout: post
comments: true
title: "Codex는 프롬프트를 이렇게 쌓는다 — Responses API 위의 에이전트 루프 해부"
description: "OpenAI가 공개한 Codex CLI 에이전트 루프 분해. instructions·tools·input 세 필드를 무엇으로 채우는지, 캐시 프리픽스를 깨뜨리는 변경이 무엇인지, ZDR과 /responses/compact 컴팩션까지 원문과 대조해 정리했다."
img: codex_agent_loop_title.webp
date: 2026-11-01 18:00:00 +0900
last_modified_at: 2026-11-01 18:00:00 +0900
tags: [openai, codex, coding-agent, responses-api, prompt-caching, compaction, mcp, llm-agent]
related: llm-agent
categories: dev
source_url: https://openai.com/index/unrolling-the-codex-agent-loop/
source_date: 2026-01-23
---

에이전트를 직접 만들어 본 사람이면 같은 자리에서 막힌다. 프롬프트를 어떤 순서로 쌓아야 캐시가 살아 있고, 대화 중간에 설정이 바뀌면 그걸 어디에 끼워 넣어야 하며, 컨텍스트가 넘칠 때 무엇을 버려야 하는가. OpenAI가 Codex CLI의 에이전트 루프를 열어 이 셋에 답했다. Michael Bolin이 쓴 2026년 1월 23일 글이고, 시리즈의 첫 편이다.

(이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Codex는 Responses API의 `instructions`·`tools`·`input` 세 필드만 쓴다. `input`에는 사용자 메시지 **앞에** 네 가지 항목(샌드박스 설명 → `developer_instructions` → AGENTS.md·스킬 메타데이터 → 환경 컨텍스트)을 고정 순서로 깔아 둔다. `previous_response_id`는 **일부러 안 쓴다** — 무상태 유지와 ZDR 지원 때문이고, 대신 프롬프트 캐싱으로 샘플링 비용을 2차에서 선형으로 떨어뜨린다. 대화 중 설정이 바뀌면 기존 메시지를 고치지 않고 **새 메시지를 append** 한다. 컨텍스트가 넘치면 `auto_compact_limit`를 넘는 시점에 `/responses/compact`를 불러 `type=compaction` + `encrypted_content`로 교체한다.

## 루프 자체는 평범하다

구조는 교과서적이다. 사용자 입력 → 추론 → 툴 호출 → 결과를 프롬프트에 덧붙여 재추론. 모델이 툴 호출을 멈추고 어시스턴트 메시지를 내놓으면 한 턴이 끝난다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"As the result of the inference step, the model either (1) produces a final response to the user's original input, or (2) requests a tool call that the agent is expected to perform (e.g., "run ls and report the output"). In the case of (2), the agent executes the tool call and appends its output to the original prompt."</blockquote></details>

중요한 건 그다음 문장이다. **새 턴마다 이전 턴의 메시지와 툴 호출이 전부 프롬프트에 다시 들어간다.** 대화가 길어질수록 프롬프트가 같이 자라고, 컨텍스트 윈도우 관리가 에이전트의 책임이 된다. 글의 나머지는 사실상 이 하나에서 파생된 문제들을 푸는 이야기다.

## 어느 엔드포인트를 두드리는가

Codex CLI가 쓰는 Responses API 엔드포인트는 설정으로 바꿀 수 있고, 인증 방식에 따라 기본값이 다르다.

| 방식 | 엔드포인트 |
|---|---|
| ChatGPT 로그인 | `https://chatgpt.com/backend-api/codex/responses` |
| API 키 인증 | `https://api.openai.com/v1/responses` |
| `--oss` (gpt-oss 로컬) | `http://localhost:11434/v1/responses` |
| 클라우드 제공자 | Azure 등 Responses API 호환 호스팅 |

로컬 경로는 **ollama 0.13.4 이상 또는 LM Studio 0.3.39 이상**을 요구한다. 바꿔 말하면 Responses API를 구현한 엔드포인트라면 무엇이든 꽂을 수 있다는 뜻이고, 자체 게이트웨이를 세운 환경에서는 이 지점이 그대로 적용점이 된다.

## 세 필드를 무엇으로 채우나

Responses API 페이로드에는 많은 파라미터가 있지만 Codex가 쓰는 건 셋이다.

**`instructions`** — `~/.codex/config.toml`의 `model_instructions_file`을 지정했으면 그 파일을, 아니면 모델에 딸린 `base_instructions`를 쓴다. 모델별 지침은 Codex 저장소에 들어 있고 CLI에 번들로 묶여 나간다(`gpt-5.2-codex_prompt.md` 같은 파일).

**`tools`** — Codex 내장 툴(`shell`, `update_plan`), Responses API가 제공하는 툴(`web_search`), 그리고 사용자가 MCP 서버로 붙인 툴이 한 배열에 섞인다. MCP 툴은 `mcp__weather__get-forecast` 꼴로 이름이 붙는다.

**`input`** — 여기가 핵심이다. Codex는 사용자 메시지를 넣기 **전에** 네 항목을 고정 순서로 깐다.

1. `role=developer` — 샌드박스 설명. 단 이건 **Codex가 제공하는 `shell` 툴에만 적용된다.** MCP 서버가 붙인 툴은 Codex가 샌드박싱하지 않으며 각자 가드레일을 책임진다
2. `role=developer`(선택) — `config.toml`의 `developer_instructions` 값
3. `role=user`(선택) — "사용자 지침". 한 파일이 아니라 여러 곳에서 모은다
4. `role=user` — `<environment_context>` 로 현재 작업 디렉터리와 셸을 알린다

<details class="evidence"><summary>원문 근거</summary><blockquote>"A message with role=developer that describes the sandbox that applies only to the Codex-provided shell tool defined in the tools section. That is, other tools, such as those provided from MCP servers, are not sandboxed by Codex and are responsible for enforcing their own guardrails."</blockquote></details>

3번의 수집 규칙이 실무에서 제일 자주 걸리는 대목이다. **구체적인 지침일수록 뒤에 온다**는 원칙 아래 이 순서로 모은다.

- `$CODEX_HOME`의 `AGENTS.override.md`·`AGENTS.md`
- cwd의 Git/프로젝트 루트부터 cwd까지 각 폴더를 훑으며 `AGENTS.override.md`, `AGENTS.md`, 또는 `config.toml`의 `project_doc_fallback_filenames`에 적힌 파일명 — **기본 한도 32 KiB**
- 스킬을 설정해 뒀다면 짧은 머리말 + 스킬별 메타데이터 + 사용법 절

32 KiB 한도는 기억해 둘 값이다. 프로젝트 문서를 계속 불리면 어느 지점부터 뒤쪽이 조용히 잘린다.

## 캐시를 깨는 것들

프롬프트가 턴마다 커지므로, 순진하게 짜면 비용이 2차로 늘어난다. Responses API에는 이를 완화하는 `previous_response_id` 파라미터가 있는데 **Codex는 쓰지 않는다.**

<details class="evidence"><summary>원문 근거</summary><blockquote>"Codex does not use it today, primarily to keep requests fully stateless and to support Zero Data Retention (ZDR) configurations. [...] storing the data required to support previous_response_id would be at odds with ZDR."</blockquote></details>

요청을 완전히 무상태로 두면 Responses API 제공자 쪽 구현이 단순해지고, 대화 상태를 서버에 저장하지 않으니 ZDR과 충돌하지 않는다. 여기에 덧붙은 설명이 흥미롭다 — **ZDR 고객도 이전 턴의 추론 메시지 이점을 포기하지 않는다.** 해당 `encrypted_content`를 서버가 복호화할 수 있기 때문인데, OpenAI는 ZDR 고객의 **복호화 키는 보관하되 데이터 자체는 보관하지 않는다**.

대신 성능은 프롬프트 캐싱으로 받는다. 캐시 히트가 나면 모델 샘플링이 2차가 아니라 선형으로 돈다. 그리고 캐시 히트의 조건이 까다롭다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Cache hits are only possible for exact prefix matches within a prompt. To realize caching benefits, place static content like instructions and examples at the beginning of your prompt, and put variable content, such as user-specific information, at the end."</blockquote></details>

**정확한 프리픽스 일치**가 조건이고, 이미지와 툴도 요청 간에 완전히 같아야 한다. 원문이 꼽은 캐시 미스 유발 요인은 셋이다.

- 대화 도중 모델이 쓸 수 있는 **툴 목록 변경**
- 요청 대상 **모델 변경** — 모델별 지침이 프롬프트의 세 번째 항목이라 그 자리가 바뀐다
- **샌드박스 구성·승인 모드·현재 작업 디렉터리 변경**

MCP가 특히 까다롭다. MCP 서버는 `notifications/tools/list_changed` 알림으로 툴 목록을 실시간으로 바꿀 수 있는데, 긴 대화 중간에 이걸 반영하면 비싼 캐시 미스가 난다. Codex 팀도 초기 MCP 지원에서 **툴을 일관된 순서로 나열하지 못하는 버그**로 캐시 미스를 낸 적이 있다고 적어 뒀다.

그래서 쓰는 해법이 단순하면서 중요하다. **대화 중 설정이 바뀌어도 이전 메시지를 고치지 않는다.** 샌드박스나 승인 모드가 바뀌면 원래와 같은 형식의 `<permissions instructions>` 를 담은 `role=developer` 메시지를 새로 끼우고, 작업 디렉터리가 바뀌면 `<environment_context>` 형식의 `role=user` 메시지를 새로 끼운다. 앞을 건드리지 않으니 프리픽스가 살아 있다.

## 컨텍스트가 넘칠 때

전략은 **토큰 수가 임계치를 넘으면 대화를 컴팩션**하는 것이다. `input`을 대화를 대표하는 더 짧은 항목 목록으로 갈아 끼워, 에이전트가 지금까지의 맥락을 유지한 채 계속 돌게 한다.

초기 구현은 사용자가 `/compact` 를 직접 치는 방식이었다. 기존 대화에 요약 전용 지침을 붙여 Responses API를 다시 부르고, 그 결과 어시스턴트 메시지를 새 `input`으로 삼았다. 지금은 다르다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Since then, the Responses API has evolved to support a special /responses/compact endpoint that performs compaction more efficiently. It returns a list of items that can be used in place of the previous input to continue the conversation while freeing up the context window. This list includes a special type=compaction item with an opaque encrypted_content item that preserves the model's latent understanding of the original conversation."</blockquote></details>

`/responses/compact` 가 돌려주는 목록에는 `type=compaction` 항목이 들어 있고, 그 안의 불투명한 `encrypted_content` 가 **원래 대화에 대한 모델의 잠재적 이해를 보존**한다. 요약문 한 덩이로 뭉개는 것과는 다르다. Codex는 `auto_compact_limit` 를 넘기면 이 엔드포인트를 자동으로 호출한다.

직접 에이전트를 만든다면 여기서 갈린다. 요약 프롬프트를 손으로 짜 붙이는 쪽과, 컴팩션 엔드포인트에 맡겨 잠재 상태를 보존하는 쪽은 긴 대화에서 결과가 달라진다.

## 자체 에이전트 점검 목록

원문이 "Responses API 위에 에이전트 루프를 올리는 누구에게나 해당한다"고 못 박은 만큼, 그대로 옮겨올 수 있는 것들이다.

- `instructions`와 `tools`를 대화 초기에 고정하고, 런타임 변경은 **기존 메시지 수정이 아니라 `input` append** 로 처리하는가
- 툴 배열의 **순서가 요청 간에 동일**한가 — 특히 MCP 툴을 동적으로 모을 때
- `tools/list_changed` 알림을 대화 중간에 그대로 반영하고 있지는 않은가
- ZDR이 필요한 환경에서 `previous_response_id` 없이 **프리픽스 캐싱만으로** 성능 목표를 맞출 수 있는가
- 컨텍스트 초과 대응이 손으로 짠 요약인가, 컴팩션 엔드포인트인가
- 팀 공통 지침을 `model_instructions_file`·`developer_instructions` 로 배포하고 있는가, 그리고 프로젝트 문서가 **32 KiB** 한도 안에 드는가

`~/.codex/config.toml` 에서 이 글이 언급한 키만 추리면 이렇다(값은 예시).

```toml
# instructions 필드를 통째로 덮어쓴다
model_instructions_file = "~/.codex/my_instructions.md"

# input 2번 항목(role=developer)으로 들어간다
developer_instructions = "팀 코딩 컨벤션: ..."

# AGENTS.md 대신 찾을 파일명들
project_doc_fallback_filenames = ["CLAUDE.md"]

# 이 토큰 수를 넘으면 /responses/compact 자동 호출
auto_compact_limit = 100000
```

## 덧붙임

이 글은 2026년 1월에 나온 시리즈 첫 편이다. 원문은 이어서 CLI 아키텍처, 툴 사용 구현, 샌드박싱 모델을 다루겠다고 예고했다. 또한 Codex CLI는 [오픈소스](https://github.com/openai/codex){:target="_blank"}라, 본문에 나온 동작은 저장소의 이슈·PR에서 세부까지 확인할 수 있다.

## 참고 자료

| 구분 | 링크 |
|---|---|
| 원문 | [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/){:target="_blank"} (Michael Bolin, 2026-01-23) |
| Codex 저장소 | [openai/codex](https://github.com/openai/codex){:target="_blank"} |
| Responses API | [API 레퍼런스](https://developers.openai.com/api/reference/resources/responses){:target="_blank"} |
| 프롬프트 캐싱 | [공식 문서](https://platform.openai.com/docs/guides/prompt-caching){:target="_blank"} |
| MCP 툴 변경 알림 | [tools/list_changed](https://modelcontextprotocol.io/specification/2025-06-18/server/tools#list-changed-notification){:target="_blank"} |
