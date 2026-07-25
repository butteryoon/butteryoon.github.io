---
layout: post
comments: true
title: "Hermes Agent 스킬 저장과 멀티 에이전트 오케스트레이션"
description: "Hermes Agent의 스킬 저장, delegate_task 병렬 실행, cronjob 예약, kanban 협업 큐를 활용해 에이전트를 일회성 채팅이 아닌 버전 관리되는 절차·작업 큐로 운영하는 법"
img: command-title.webp
date: 2026-07-25 14:00:00 +0900
last_modified_at: 2026-07-25 14:00:00 +0900
tags: [hermes, ai agent, skill, delegate, cron, orchestration, nous research] # add tag
related: llm
categories: tools
---

[Hermes Agent 설치부터 설정까지]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html) 글에서 설치와 모델 연결까지 다뤘다. 이번에는 "한 번 쓴 도구 호출 경험을 다음에도 쓴다"는 차별점을 실제로 써먹는 법 — **스킬 저장**과, 여러 에이전트를 하나로 묶는 **오케스트레이션**(`delegate_task`, `cronjob`, `kanban`) 활용기를 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** Hermes는 반복 절차를 `SKILL.md` 하나로 모아 다음 세션에 자동 로드한다(스킬). 무거운 병렬 작업은 `delegate_task`로 격리된 서브에이전트에 넘기고, 정기 작업은 `cronjob`으로, 다중 에이전트 협업은 `kanban` 보드로 관리한다. 핵심은 **"에이전트를 일회성 채팅이 아니라, 버전 관리되는 절차+작업 큐로 운영한다"**는 것 — 앞선 [NVIDIA AI Factory 글]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)의 AgentOps(스킬=버전화된 행동 단위)와 정확히 같은 철학이다.

## 왜 오케스트레이션인가

단발 채팅으로 툴을 몇 번 쓰는 건 누구나 한다. 문제는 **같은 일을 다음 주에도, 더 크고 병렬인 형태로** 하고 싶을 때다. AI Factory 가이드가 에이전트를 "버전 관리·테스트·모니터링·롤백이 가능한 1급 서비스"로 취급하라고 한 것과 같은 맥락에서, Hermes가 제공하는 세 계층을 정리한다.

| 계층 | 도구 | 수명 | 격리 | 쓰임 |
| --- | --- | --- | --- | --- |
| 절차 기억 | **Skill** | 세션 간 영속 | — | 반복 절차를 재사용 가능한 지식으로 저장 |
| 병렬 실행 | **delegate_task** | 분(부모 루프 종속) | 대화·터미널 격리 | 빠른 병렬 서브태스크 |
| 예약 실행 | **cronjob** | 영속(프로세스 독립) | 독립 세션 | 주간/정기 파이프라인 |
| 협업 큐 | **kanban** | 영속(SQLite) | 프로필/테넌트 격리 | 다중 워커 작업 분배 |

## 1. 스킬 저장 — "파일이 곧 지식"

[OKF 리뷰]({{site.baseurl}}/dev/2026/07/22/okf_review.html)에서 정리한 "파일 경로가 곧 식별자, 사람과 에이전트가 같은 문서를 본다" 철학이 Hermes 스킬에서도 그대로 나타난다. 스킬은 디렉토리 + `SKILL.md` 명세다.

```
~/.hermes/skills/<name>/
└── SKILL.md          # YAML frontmatter + 마크다운 본문
```

`SKILL.md`의 최소 구조:

```yaml
---
name: pdf-to-markdown
description: "Use when converting PDF reports to Markdown for the blog pipeline."
version: 1.0.0
author: butteryoon
---
# PDF → Markdown 변환 절차
1. `pdftotext -layout $1 - | ...` 로 텍스트 추출
2. 표 구조 보존을 위해 청킹 대신 페이지 단위 분할
3. 출력은 `_drafts/` 에 저장
```

스킬이 만들어지면 다음 세션부터 Hermes가 목록을 보고 **관련 작업 시 자동 로드**한다. 즉 "이번에 터미널에서 jq 필터 10번 돌려서 알아낸 순서"를 스킬로 남기면, 다음엔 그 순서를 다시 발견할 필요가 없다. AI Factory 글에서 지적한 "AgentOps의 핵심은 버전 관리 가능한 행동 단위"를 Hermes는 이 디렉토리+명세 방식으로 충족한다.

관리(자동 정리)는 `curator`가 맡는다 — 사용량 추적, 유휴 스킬 보관(archive). **삭제는 never**, 최대 파괴적 동작이 archive이므로 안전하다.

```
hermes curator usage        # 스킬별 사용 횟수·조회·패치 횟수 확인
hermes curator list-archived # 보관된 스킬 목록
```

이 글을 쓰는 환경에서 실제로 돌려본 출력:

```text
$ hermes curator usage
skills: 71 total  (agent=2  bundled=69  hub=0)

  skill                                     origin     use  view  patch   act  last_activity
  hermes-agent                              bundled      7     7      0    14  3h ago
  repo-audit                                agent        0     0      1     1  6h ago
  repo-orientation                          agent        0     0      1     1  6h ago
  arxiv                                     bundled      0     0      0     0  never
  ...

$ hermes curator list-archived
curator: no archived skills
```

`act`(자동 전환/정리 동작 횟수) 컬럼이 보이는 점이 핵심 — curator가 실제로 스킬 수명을 추적하고 있음을 보여준다. agent origin 스킬(repo-audit 등)만 빈도가 잡히고 bundled는 `never`인 점도, "내가 만든 스킬만 curator 관리 대상"이라는 앞선 설명과 일치한다.

## 2. delegate_task — 병렬 서브에이전트

부모 대화를 오염시키지 않고 독립 컨텍스트+터미널 세션으로 서브에이전트를 돈다.

**단일 / 배치 / 백그라운드**

```
# 단일 서브태스크
delegate_task(goal="RFP 분석해서 요구사항 표 추출", context="...원본 경로...")

# 배치 — 최대 max_concurrent_children(기본 3)병렬
delegate_task(tasks=[
  {goal="논문 A 요약", ...},
  {goal="논문 B 요약", ...},
  {goal="논문 C 요약", ...},
])

# 백그라운드 — 핸들 즉시 반환, 끝나면 대화로 재진입
delegate_task(goal="CI/CD 구축", background=true)
```

**역할(role)** 도 있다. `leaf`(기본, 재위임 불가) vs `orchestrator`(자신의 워커를 다시 spawn). 깊이는 `delegation.max_spawn_depth`로 묶인다.

`delegate_task` vs `hermes` 프로세스 직접 spawn (tmux) 선택 기준:

| | delegate_task | tmux spawn |
| --- | --- | --- |
| 격리 | 대화 격리, 동일 프로세스 | 완전 독립 프로세스 |
| 수명 | 분 단위(부모 종속) | 시간/일 단위 |
| 도구 | 부모 도구 일부 | 전체 도구 |
| 대화형 | 불가 | PTY 모드 가능 |
| 쓰임 | 빠른 병렬 서브태스크 | 장시간 자율 미션 |

빠른 병렬은 `delegate_task`, 며칠 도는 자율 에이전트는 tmux spawn이 맞다. **중요:** 백그라운드 `delegate_task`도 부모 프로세스가 죽으면 사라진다. 프로세스 밖에서 살아야 하면 `cronjob`이나 `terminal(background=True)`을 쓴다.

## 3. cronjob — 영속 예약 파이프라인

`delegate_task`가 "지금 병렬로"라면, `cronjob`은 "매주 자동으로"다. 프로세스와 무관하게 살아남는 유일한 스케줄러다.

```
cronjob(action="create",
        name="weekly-rag-digest",
        schedule="0 9 * * 0",          # 매주 일 09:00 (UTC)
        prompt="...자기완결형 프롬프트...",
        skills=["pdf-to-markdown", "arxiv-survey"],
        deliver="telegram")
```

스케줄은 `30m`/`every monday 9am`/5필드 cron/ISO 타임스탬프를 모두 받는다. `.tick.lock`으로 중복 틱을 막아 이중 발행이 안 나고, `context_from`으로 "작업 A 출력 → 작업 B 입력" 체이닝도 된다. [블로그 자동화 글]({{site.baseurl}}/tools/2026/07/19/blog_auto_publish.html)의 Windows 작업 스케줄러 방식이 Hermes 안에서는 이 `cronjob` 하나로, PC 전원과 무관하게 동작한다.

이 글을 쓰는 시점의 스케줄러 상태(아직 등록된 작업 없음):

```text
$ hermes cron list
No scheduled jobs.
Create one with 'hermes cron create ...' or the /cron command in chat.
```

프롬프트엔 역시 **안전장치**를 넣는다 — 보고서 없으면 종료, 중복 발행 금지, 미래 날짜 금지, 사실 창작 금지. 자동화 글에서 검증된 패턴 그대로다.

## 4. kanban — 다중 에이전트 작업 큐

여러 프로필/워커가 같은 보드에서 일할 때 쓴다. SQLite 기반이라 영속되고, 워커는 `HERMES_KANBAN_BOARD`로 격리된다.

```
hermes kanban init
hermes kanban create "위성영상 캡셔닝 벤치마크" --assign research
hermes kanban dispatch      # 게이트웨이 디스패처가 할당된 프로필 spawn
```

디스패처가 죽은 claim을 회수하고, `failure_limit`(기본 2) 연속 실패 시 태스크를 자동 block한다. "사람이 관리자, 에이전트가 작업자" 구조를 만들고 싶을 때 딱 맞다.

이 글을 쓰는 환경의 `hermes kanban --help`가 보여주는 실제 서브커맨드(일부):

```text
$ hermes kanban --help
Durable SQLite-backed task board shared across Hermes profiles. Tasks are
claimed atomically, can depend on other tasks, and are executed by a named
profile in an isolated workspace.

  {init,boards,create,swarm,list,ls,show,assign,reclaim,reassign,link,
   unlink,claim,comment,attach,complete,edit,block,schedule,unblock,
   promote,archive,tail,dispatch,daemon,watch,stats,notify-subscribe,
   log,runs,heartbeat,assignees,context,specify,decompose,gc,repair}
    init      Create kanban.db if missing (idempotent)
    create    Create a new task
    swarm     Create a Kanban Swarm v1 graph (parallel workers → verifier → synthesizer)
    dispatch  Gateway dispatcher spawns assigned profiles
    ...
```

`swarm` 서브커맨드가 눈에 띈다 — "parallel workers → verifier → synthesizer" 그래프를 한 번에 만든다. 단순 작업 큐를 넘어 [Caller/Executor 패턴 글]({{site.baseurl}}/dev/2026/07/16/caller_executor_agent.html)의 구조를 kanban 위에서 바로 올릴 수 있다.

## 예시 파이프라인 — 주간 연구 동향을 끝까지 자동화

세 계층을 엮어 [블로그 자동화 글]({{site.baseurl}}/tools/2026/07/19/blog_auto_publish.html)의 "남은 과제(보고서 생성 자동화)"를 채운다.

```
cronjob(매주 토 "arxiv 수집+요약" → _reports/YYYYMMDD_report.md 커밋)
   └─ 내부에서 delegate_task(배치)로 논문 N편 병렬 요약
cronjob(매주 일 "보고서 → 블로그 글" → _posts/ 발행, 스킬 로드)
   └─ pdf-to-markdown 스킬로 포맷 통일
kanban(장기 테마 조사 — 위성영상/온프레미스 파이프라인) 병렬 추적
```

수집→요약→발행이 모두 에이전트+스킬로 처리되고, 사람은 kanban 보드만 들여다본다.

## 마무리

Hermes에서 "오케스트레이션"이란 채팅창을 더 크게 쓰는 게 아니다. **스킬로 절차를 자산화하고, delegate로 병렬화하고, cron/kanban으로 수명과 협업을 분리**하는 것. AI Factory 가이드가 벤더 중립적으로 정의한 AgentOps 컴포넌트(버전화된 스킬, 트레이스, 롤백)가 오픈소스 Hermes에서는 `curator`+`SKILL.md`+`cronjob` 조합으로 그대로 매핑된다. 다음 글에서는 이 파이프라인에 [오픈소스 가드레일]({{site.baseurl}}/dev/2026/07/22/guardrail_onprem_pipeline.html)을 Security 계층으로 끼우는 방법을 다뤄볼 예정이다.

## 참고

- [Hermes Agent 공식 문서](https://hermes-agent.nousresearch.com/docs/){:target="_blank"}
- [Background Systems (Delegation / Cron / Curator / Kanban)](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron){:target="_blank"}
- [NVIDIA AI Factory — Agentic AI in the Factory (관련글)]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)
- [오픈소스 가드레일 파이프라인 (관련글)]({{site.baseurl}}/dev/2026/07/22/guardrail_onprem_pipeline.html)
- [Claude Code로 블로그 자동화 (관련글)]({{site.baseurl}}/tools/2026/07/19/blog_auto_publish.html)
