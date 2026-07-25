---
layout: post
comments: true
title: "에이전트는 어떻게 '배우고' 고치는가 — 기억·스킬·피드백 루프의 실제 구조"
description: "AI 에이전트가 작업 알고리즘을 어떻게 수행하고, 대화를 어떻게 기억하고, 타 에이전트의 결과를 분석해 행동을 수정하고 새 작업을 익히는지 Hermes의 설정 파일·스킬·메모리 구조와 연결해 해설"
img: command-title.webp
date: 2026-07-25 20:00:00 +0900
last_modified_at: 2026-07-25 20:00:00 +0900
tags: [ai agent, hermes, memory, skill, learning, orchestration, llm, nous research] # add tag
related: llm
categories: dev
---

AI 에이전트가 "똑똑하다"는 말은 보통 모델 성능 이야기다. 하지만 실제 운영에서 에이전트가 쓸모 있는 이유는 **모델 자체가 아니라, 그것이 상태를 어디에 어떻게 저장하고 피드백을 어떻게 행동으로 바꾸느냐**에 있다. 이 글은 [Hermes Agent](https://github.com/NousResearch/hermes-agent)를 직접 써보며 관찰한 구조 — 작업 알고리즘, 기억 계층, 타 에이전트 결과 분석, 행동 수정·학습 — 를 실제 설정 파일·디렉토리와 연결해 해설한다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** 에이전트의 "학습"은 신경망 가중치 갱신이 아니다. ① **작업 알고리즘**은 도구 호출 루프(관측→판단→행동→결과)이고, ② **기억**은 `MEMORY.md`(규칙) + `state.db`(대화) + `SKILL.md`(절차) 3계층이며, ③ **타 에이전트 분석·행동 수정**은 스킬/메모리에 피드백을 영속화하는 루프다. Hermes에서는 이게 파일 시스템상의 구체적 경로로 관찰된다 — `AppData\Local\hermes\memories\MEMORY.md`, `skills\writing\blog-post-authoring\SKILL.md`, `state.db`.

## 1. 작업 알고리즘 — 툴 호출 루프

에이전트가 "일"을 하는 단위는 **턴(turn)**이다. 각 턴은 관측(Observation) → 추론(Reasoning) → 행동(Action: 툴 호출) → 결과(Result)의 사이클이다. 멀티 에이전트에서는 이걸 [Caller/Executor 패턴]({{site.baseurl}}/dev/2026/07/16/caller_executor_agent.html)으로 분해한다 — 오케스트레이터가 하위 에이전트에 태스크를 넘기고 결과를 모은다.

이 루프를 격리된 컨텍스트로 병렬 돌리는 게 `delegate_task`다. 오늘 블로그 4편을 쓰는 동안, 에이전트는 매 턴마다 "지금 어느 글을 쓰나 / 관련글 링크는 무엇인가 / 명령어를 실제로 돌려야 하나"를 판단하고 툴(`web_extract`, `terminal`, `write_file`)을 호출했다. 즉 알고리즘 자체는 단순한 반복이되, **상태(무엇을, 왜, 어떻게)를 어디에 두느냐**가 품질을 결정한다.

## 2. 기억의 3계층 — 무엇을 어디에

관찰한 Hermes의 기억 구조:

| 계층 | 물리 위치 | 저장 내용 | 예시 |
| --- | --- | --- | --- |
| **규칙 메모리** | `memories/MEMORY.md` | 사용자 선호·절차적 규칙(요약) | "블로그는 `<!--more-->`를 TL;DR 앞에 둔다" |
| **대화 기록** | `state.db` (SQLite+FTS5), `sessions/*.jsonl` | 실제 주고받은 대화·도구 호출 | "어제 OKF 글 썼을 때의 전체 맥락" |
| **절차 스킬** | `skills/<name>/SKILL.md` | 반복 작업을 코드화한 절차 | `blog-post-authoring` 스킬 |

핵심: **MEMORY.md만 봐서는 "오늘 무슨 일 했나"를 모른다.** 거기엔 규칙 요약만 있다(현재 1,305/2,200 bytes). 실제 작업 내용은 `state.db`에 있고, `session_search`로 꺼내 쓴다. 절차 자체를 자동화하려면 스킬로 올린다. 이 3계층 분리는 "뭔가를 기억한다"는 게 단일 파일이 아니라 **목적별 저장소 분리**임을 보여준다 — 모델 컨텍스트 창은 비싸고 휘발성이라, 영속할 것은 디스크에 규칙/절차로 빼두는 설계다.

## 3. 타 에이전트 결과 분석 — 피드백을 어떻게 읽나

오늘 관찰한 가장 흥미로운 루프는 **클로드(별도 에이전트)와의 협업**이다. 흐름은:

```
Hermes(초안+실제명령검증) → Claude(검토/발행/링크통일) → Hermes(지적사항 보완)
```

Claude가 발행하며 한 일 — 날짜 정합성 수정, 태그 추가, 관련글 링크를 `{{site.baseurl}}` 방식으로 통일, `<!--more-->` 누락 지적 — 은 **외부 에이전트의 출력(수정된 파일)을 Hermes가 다시 읽고 자기 행동을 고치는** 사례다. 즉 "타 에이전트 분석"이란, 상대방이 남긴 artifact(발행된 글, 정정 이력)를 관찰하고 그 규칙을 자기 메모리/스킬에 흡수하는 것이다.

실제로 Claude가 `{{site.baseurl}}` 링크 규칙을 적용한 걸 보고, Hermes는 그걸 `blog-post-authoring` 스킬의 `references/blog-format.md`에 **"Internal link convention (Claude-enforced)"**로 기록했다. 외부 피드백 → 스킬 갱신. 이것이 학습이다.

## 4. 행동 수정과 학습 — 파일로서의 진화

에이전트가 "잘못짚고 고쳤다"를 다음에도 반복하지 않게 하는 장치:

- **Pitfalls 기록**: 스킬 SKILL.md의 Pitfalls 섹션에 "존재하지 않는 `hermes curator pin`을 추측해 넣음 → `--help`로 발견"을 명시. 다음 초안은 처음부터 검증한다.
- **메모리 규칙화**: "미래 날짜는 GitHub Pages에서 누락"을 MEMORY.md에 저장 → 다음 글은 날짜를 오늘로 고정.
- **스킬 진화**: `blog-post-authoring` 스킬은 처음엔 포맷/검증만 있었으나, 클로드 루프를 겪고 "§6 Claude review/publish loop" 항목이 추가됨.

즉 학습 = **가중치 업데이트가 아니라, 디스크 상의 스킬/메모리 파일 편집**이다. 신경망은 에포크마다 바뀌지만, 이 에이전트는 **피드백을 받을 때마다 SKILL.md/MEMORY.md를 패치**한다. [OKF 글]({{site.baseurl}}/tools/2026/07/25/hermes_okf_knowledge.html)에서 본 "파일이 곧 지식" 구조가 에이전트 자기 자신에게도 적용된다.

## 5. 이걸 왜 아는가 — 관찰 가능성(Observability)

이 모든 게 "주장"이 아니라 **실제 경로 확인**으로 왔다. 오늘 직접 본 것:

```text
$ ls AppData\Local\hermes\memories\
MEMORY.md   MEMORY.md.lock

$ cat MEMORY.md   (1,305 bytes: 규칙 2줄, § 구분)

$ ls skills\writing\blog-post-authoring\
SKILL.md  references/  templates/
```

[AI Factory 글]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)에서 NVIDIA가 요구한 "통합 평가·트레이스 리플레이·버전 관리되는 스킬"이, 오픈소스 Hermes에서는 `state.db`(대화 재현) + `SKILL.md`(버전화된 절차) + `memories/`(규칙) 조합으로 관찰 가능하게 구현된다. 비결정적 LLM 시스템에서 "에이전트가 왜 그랬나"를 추적하려면, 상태를 코드가 아닌 **읽을 수 있는 파일**로 두는 수밖에 없다.

## 마무리

에이전트의 "지능"은 모델 추론력보다 **상태를 어디에 어떻게 두고, 피드백을 어떻게 파일로 바꾸느냐**에 있다. 작업은 툴 호출 루프, 기억은 규칙/대화/절차 3계층, 타 에이전트 분석은 상대 artifact 관찰, 학습은 스킬/메모리 패치. 오늘 블로그 4편을 쓰며 이 전 과정이 한 화면에서 관찰됐다 — 초안을 쓰고, 실제 명령어로 검증하고, 클로드가 고치고, 그 교정이 다시 스킬에 박혔다. 에이전트가 "익힌" 새 작업 내용은 결국 `SKILL.md` 한 줄로 남는다.

## 참고

- [Hermes Agent 공식 문서](https://hermes-agent.nousresearch.com/docs/){:target="_blank"}
- [Background Systems — Delegation/Cron/Curator/Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron){:target="_blank"}
- [Caller/Executor 패턴으로 멀티 에이전트 구성 (관련글)]({{site.baseurl}}/dev/2026/07/16/caller_executor_agent.html)
- [NVIDIA AI Factory — Agentic AI (관련글)]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)
- [OKF 지식 번들 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_okf_knowledge.html)
- [스킬 오케스트레이션 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html)
