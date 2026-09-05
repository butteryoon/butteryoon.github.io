---
layout: post
comments: true
title: "Compound Engineering 플러그인 — AI 코딩 루프를 누적 학습으로 바꾸기"
description: "Every의 오픈소스 Compound Engineering 플러그인(33개 스킬, 14개 에이전트 호스트 지원) 해설. brainstorm→plan→work→simplify→review→compound 6단계 루프와 docs/solutions/ 지식 축적 구조, Claude Code 실제 설치 확인까지."
img: compound-engineering-title.webp
date: 2026-09-05 20:30:00 +0900
last_modified_at: 2026-09-06 00:00:00 +0900
tags: [compound-engineering, ai-coding, claude-code, codex, cursor, plugin, knowledge-management] # add tag
related: llm
categories: tools
---

AI 코딩의 고질적 문제는 세션이 끝나면 배운 것이 사라진다는 점이다. [Every](https://every.to){:target="_blank"}가 공개한 오픈소스 **Compound Engineering 플러그인**(MIT, 24.8k 스타)은 이 문제를 정면으로 다룬다. 매 작업 루프의 끝에 **배운 것을 파일로 남겨서, 다음 루프가 그것을 읽고 더 똑똑하게 시작**하게 만드는 33개 스킬 묶음이다. Claude Code·Cursor·Codex를 포함해 14개 에이전트 호스트를 지원한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 저장소 대조·실제 설치 검증 후 발행했다.)

<!--more-->

> **TL;DR:** Compound Engineering = **"80% 계획·리뷰, 20% 실행"** 철학을 33개 스킬로 구현한 플러그인. 핵심 루프는 `/ce-brainstorm → /ce-plan → /ce-work → /ce-simplify-code → /ce-code-review → /ce-compound` 6단계이고, 마지막 `/ce-compound`가 이번 작업에서 배운 패턴을 `docs/solutions/`에 문서화해 **다음 세션의 컨텍스트 자산**으로 만든다. Claude Code에서는 `/plugin marketplace add EveryInc/compound-engineering-plugin` 두 줄로 설치된다.

## 핵심 아이디어 — 실행보다 계획과 리뷰

일반적인 AI 코딩은 실행(코드 생성)에 시간을 쏟는다. Compound Engineering은 이를 뒤집는다:

> **80%는 계획과 리뷰에, 20%만 실행에.** 그리고 매 루프의 끝에 배운 것을 기록해 다음 루프의 시작점을 올린다.

이 블로그에서 다룬 [Andrew Ng의 "에이전트 시대 소프트웨어 기초"]({{site.baseurl}}/dev/2026/08/29/andrew-ng-agentic-coding-fundamentals.html)와 정확히 같은 방향이다. 구현이 싸질수록 계획·리뷰·트레이드오프 판단의 비중이 커진다는 것.

## 6단계 핵심 루프

| 단계 | 스킬 | 역할 |
|------|------|------|
| 1. 아이디어 | `/ce-brainstorm` | 대화형 Q&A로 요구사항만 담은 통합 계획 초안 작성 |
| 2. 계획 | `/ce-plan` | 요구사항을 구현 가능한 계획으로 구체화 |
| 3. 실행 | `/ce-work` | 계획대로 구현 — 호스트의 검증·커밋·배포 절차 유지 |
| 4. 단순화 | `/ce-simplify-code` | 리뷰 전에 갓 쓴 코드의 명료성·재사용성 정리 |
| 5. 리뷰 | `/ce-code-review` | 계획 대비 멀티 에이전트 리뷰 (보고 전용 — 수정 반영은 명시적으로) |
| 6. 지식 축적 | `/ce-compound` | 배운 것을 `docs/solutions/`에 문서화 — **다음 루프가 더 똑똑하게 시작** |

버그에서 시작하면 `/ce-debug`, 뭘 만들지부터 모호하면 `/ce-ideate`로 진입한다. 전체 33개 스킬에는 PR 돌봄(`/ce-babysit-pr`), 커밋·PR 자동화(`/ce-commit-push-pr`), 워크트리 관리(`/ce-worktree`), 그리고 6단계를 한 번에 모는 `lfg`까지 포함된다.

## 설치 — 실제 확인

공식 설치 방법은 도구별로 다르다 (README 기준):

```text
# Claude Code (세션 안에서)
/plugin marketplace add EveryInc/compound-engineering-plugin
/plugin install compound-engineering

# Cursor (Agent 채팅에서)
/add-plugin compound-engineering

# Codex CLI
codex plugin marketplace add EveryInc/compound-engineering-plugin
codex plugin add compound-engineering@compound-engineering-plugin
```

Claude Code에 직접 설치해 확인해봤다 (v3.24.0 기준):

- `~/.claude/plugins/cache/.../skills/` 아래 **스킬 디렉토리가 정확히 33개** — 각 스킬은 SKILL.md + 세분화된 references/ 문서로 구성돼 있어 구조가 상당히 정교하다 (`ce-brainstorm` 하나에 참조 문서 20여 개)
- 저장소 자체가 도그푸딩 중이다 — 리포 안의 `docs/solutions/`에 이 플러그인을 만들며 축적한 솔루션 문서("스킬 게이트는 git 명령이 아니라 상태 조건으로" 등)가 실제로 쌓여 있다
- 산출물 폴더(`docs/solutions/`, `docs/plans/`)는 기본값이고, `docs_root` 설정으로 위치를 옮길 수 있다

## 왜 `docs/solutions/`가 핵심인가

`/ce-compound`가 만드는 솔루션 문서는 "이번에 어떤 패턴으로 문제를 풀었는지"를 마크다운으로 남긴다. 이 폴더가 Git에 커밋되면:

- **개인**: 다음 세션이 이전 해결 패턴을 컨텍스트로 참조 — 같은 시행착오 반복 없음
- **팀**: 공유 저장소의 `docs/solutions/`가 온보딩 문서 겸 지식베이스로 자동 성장

이 블로그가 Hermes 검수 체크리스트를 메모리에 쌓아온 방식이나, [NotebookLM 스킬 엔지니어링]({{site.baseurl}}/dev/2026/08/29/notebooklm-skill-engineering.html)에서 다룬 "프롬프트를 자산화"하는 흐름과 같은 계보다. 차이는 이걸 **코딩 루프의 정식 단계로 강제**한다는 점이다.

## 한계와 주의점

- **학습 곡선**: 스킬 33개의 역할과 체이닝을 익히는 초기 비용이 있다 — 핵심 루프 6개부터 시작하는 걸 권한다
- **컨텍스트 예산**: `docs/solutions/`가 커질수록 로드 비용이 늘어난다. 주기적 아카이브·정리가 필요
- **공유 전제**: 솔루션 폴더가 커밋되어야 팀 학습이 성립한다. 로컬에만 두면 효과가 반감된다
- **업데이트 절차**: 마켓플레이스를 먼저 갱신하지 않고 `/plugin update`만 돌리면 구버전에 머문다 (README가 명시적으로 경고하는 함정)

## 아키텍처 다이어그램

6단계 루프와 스킬 그룹, 지식 베이스 피드백 구조를 한눈에 볼 수 있는 다이어그램:

<iframe src="{{site.baseurl}}/assets/html/compound-engineering-diagram.html" width="100%" style="aspect-ratio: 13 / 9; height: auto; border: 1px solid #1e293b; border-radius: 8px; background: #020617;"></iframe>

[전체 화면으로 열기]({{site.baseurl}}/assets/html/compound-engineering-diagram.html){:target="_blank"}

## 마무리

Compound Engineering의 요점은 프레임워크가 아니라 습관의 강제다 — **"끝났으면 배운 걸 적어라"를 루프의 정식 단계로 만든 것**. 이 블로그의 발행 파이프라인이 검수 교훈을 메모리와 체크리스트로 쌓아온 것과 같은 원리를, 코딩 워크플로우 전반에 일반화한 도구라고 보면 된다. Claude Code 사용자라면 설치가 두 줄이니 핵심 루프 6개부터 돌려보는 걸 권한다.

## 참고

- [EveryInc/compound-engineering-plugin (GitHub)](https://github.com/EveryInc/compound-engineering-plugin){:target="_blank"}
- [Every](https://every.to){:target="_blank"} — 메인테이너: Kieran Klaassen, Trevin Chow
- [Andrew Ng의 에이전트 시대 소프트웨어 기초 (관련글)]({{site.baseurl}}/dev/2026/08/29/andrew-ng-agentic-coding-fundamentals.html)
- [NotebookLM 스킬 엔지니어링 (관련글)]({{site.baseurl}}/dev/2026/08/29/notebooklm-skill-engineering.html)
- [Agent Skills 엔지니어링 워크플로우 (관련글)]({{site.baseurl}}/tools/2026/08/11/agent_skills_engineering_workflows.html)
