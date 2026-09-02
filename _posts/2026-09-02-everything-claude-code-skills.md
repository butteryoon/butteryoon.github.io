---
layout: post
comments: true
title: "Everything Claude Code (ECC) 스킬 가이드: AI 코딩 에이전트 워크플로우와 모듈형 하네스 분석"
description: "Claude Code 및 AI 코딩 에이전트의 생산성을 높이는 Everything Claude Code(ECC) 하네스의 구조, 핵심 스킬(TDD, 보안, 코드 리뷰)과 설치·활용법을 정리한다."
img: api_code_title.jpg
date: 2026-09-02 00:50:00 +0900
last_modified_at: 2026-09-02 18:20:00 +0900
tags: [claude-code, ecc, ai-agents, agent-skills, tdd, code-review, security]
related: llm
categories: dev
---
AI 코딩 에이전트(Claude Code, Codex, Cursor, OpenCode 등)가 단순한 챗봇을 넘어 실제 엔지니어링 워크플로우를 주도하는 수준으로 진화하면서 에이전트의 작업 규율과 표준 절차를 제어하는 **에이전트 하네스(Agent Harness)**의 중요성이 커지고 있다. 그중 널리 쓰이는 오픈소스 프레임워크인 **Everything Claude Code (ECC)**와 그 **스킬(Skills)** 생태계의 구조와 실무 적용 방안을 정리한다.

(이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** ECC는 스킬·서브에이전트·메모리·훅의 네 레이어로 코딩 에이전트에 SDLC 모범 사례를 주입하는 MIT 라이선스 하네스다. 필요한 `SKILL.md`만 온디맨드로 로드해 컨텍스트를 절약하고, 플래너·코드 리뷰어·보안 리뷰어 같은 역할별 서브에이전트에 검증을 위임한다. Claude Code에서는 플러그인 마켓플레이스 두 줄로 설치할 수 있다.

## 1. Everything Claude Code (ECC) 개요

**ECC(Everything Claude Code)**는 [affaan-m/ECC](https://github.com/affaan-m/ECC){:target="_blank"} 저장소에서 개발하는 오픈소스 프로젝트로, 범용 AI 코딩 에이전트를 규율 있는 **소프트웨어 엔지니어링 플랫폼**으로 바꿔주는 에이전트 하네스 및 스킬 모음집이다. README에 따르면 68개 서브에이전트, 286개 스킬, 94개 슬래시 명령을 담았으며 MIT 라이선스로 공개돼 있다. Claude Code를 1차 대상으로 하고, Codex는 정식 지원, Cursor와 OpenCode는 베타 단계다.

AI 코딩 에이전트에게 단순 프롬프트로 "이 기능을 구현해줘"라고 요청하면 테스트를 건너뛰거나, 프로젝트의 기존 규칙을 무시하고 보안 취약점이 포함된 코드를 작성하기 쉽다. ECC는 이 한계를 메우려고 **소프트웨어 개발 라이프사이클(SDLC) 전반의 모범 사례(Best Practice)를 모듈화된 '스킬(Skill)'과 '서브에이전트(Sub-agent)' 형태로 주입**한다. 프로젝트가 내세우는 철학은 "컨텍스트 윈도우는 최적화하고, 나머지는 모두 영속화한다(Optimize the context window. Persist everything else.)"이다.

## 2. ECC 스킬의 핵심 아키텍처 및 동작 원리

ECC는 컨텍스트 낭비를 막고 작업의 일관성을 유지하려고 4가지 레이어로 구성된다.

```
┌──────────────────────────────────────────────────────────────┐
│                     ECC Agent Harness                        │
├──────────────────────────────────────────────────────────────┤
│ 1. Skills (skills/)        : 모듈형 행동 지침 및 체크리스트     │
│ 2. Sub-agents (agents/)    : 역할별 에이전트 위임 (Reviewer 등) │
│ 3. Memory (.ecc/memory/)   : 세션 간 지속되는 프로젝트 컨텍스트 │
│ 4. Hooks (hooks/)          : SessionStart/PostToolUse/Stop 자동화│
└──────────────────────────────────────────────────────────────┘
```

이 외에 `commands/`(슬래시 명령 진입점), `rules/`(언어별 코딩 표준), `mcp-configs/`(MCP 서버 정의) 디렉터리가 함께 있다.

### 1) 토큰 효율적인 온디맨드 로딩 (Token Efficiency)
모든 개발 지침(TDD 규칙, 보안 체크리스트, 스타일 가이드)을 단일 시스템 프롬프트에 넣으면 컨텍스트 윈도우가 빠르게 잠식된다. ECC 스킬은 사용자가 특정 작업(예: `/ecc:plan`, `/code-review`)을 요청하거나 해당 상황이 감지될 때 **필요한 `SKILL.md` 파일만 선택적으로 메모리에 로드**해 실행한다.

### 2) 역할 기반 서브에이전트 위임 (Sub-agent Delegation)
메인 코딩 에이전트가 모든 추론과 검증을 동시에 떠맡지 않는다. 작업 성격에 맞는 서브에이전트(`planner`, `code-reviewer`, `security-reviewer`, `tdd-guide` 등)에게 컨텍스트를 분리해 위임하고, 그만큼 환각을 줄이고 검증 신뢰도를 높인다. 코드 리뷰는 구현 컨텍스트가 섞이지 않은 "새 컨텍스트(fresh-context)"에서 수행하는 것이 특징이다.

### 3) 세션 간 메모리 영속화 (Memory & Persistence)
세션이 끝나면 `/save-session`이나 `/learn-eval` 명령으로 작업 내용을 요약·학습된 인스팅트(instinct)·재사용 스킬로 증류한다. 선택적으로 `.ecc/memory/` 아래 로컬 마크다운 형식의 통합 메모리 볼트를 두어 여러 하네스가 컨텍스트를 공유할 수 있다.

## 3. 주요 ECC 스킬 카테고리

| 카테고리 | 대표 스킬·명령 | 핵심 역할 및 동작 메커니즘 |
| :--- | :--- | :--- |
| **개발 방법론** | **`tdd-workflow`** 스킬 / `tdd-guide` 에이전트 | 테스트 주도 개발(Red-Green-Refactor) 강제. 구현 전 실패 테스트 작성 필수 |
| **설계/기획** | **`/ecc:plan`** / `planner`·`architect` 에이전트 | 작업 요구사항 분석, 서브태스크 분할, 승인 게이트를 거친 구현 계획 수립 |
| **품질 검사** | **`/code-review`** / `code-reviewer` 에이전트 | 새 컨텍스트에서 품질·보안 관점의 코드 리뷰. 언어별 `/go-review`, `/python-review`, `/java-review` 제공 |
| **보안 검증** | **`/security-scan`** / `security-review` 스킬 | OWASP Top 10 점검, 에이전트 설정 자체를 검사하는 AgentShield 스캐닝 |
| **빌드/오류 복구** | **`/build-fix`** / `build-error-resolver` 에이전트 | 빌드 실패 로그 추적, 언어별 리졸버(Go·Rust·Java·Kotlin·Swift 등)로 자동 복구 |
| **UI/UX 접근성** | **`frontend-a11y`**·`accessibility` 스킬 | 웹 접근성 표준 준수 여부 검사 |
| **테스트 자동화** | **`e2e-testing`** 스킬 / `e2e-runner` 에이전트 | Playwright 기반의 브라우저 E2E 테스트 시나리오 자동 작성 |

이 밖에 `/refactor-clean`(죽은 코드 제거), `/update-docs`(문서 동기화) 등 유지보수 명령도 있다.

## 4. 설치 및 실무 활용 방법

Claude Code 환경에서는 플러그인 마켓플레이스로 설치한다.

```bash
# 1. Claude Code 플러그인 마켓플레이스 등록
/plugin marketplace add https://github.com/affaan-m/ECC

# 2. ECC 플러그인 및 스킬 설치
/plugin install ecc@ecc

# 3. 실무 워크플로우 슬래시 명령어 호출 예시
/ecc:plan "사용자 인증 JWT 토큰 만료 갱신 API 설계해줘"
# (구현 단계에서는 tdd-workflow 스킬이 실패 테스트 → 구현 → 검증 순서를 강제한다)
/code-review
/security-scan
```

터미널에서 `npx ecc-universal setup`을 실행하는 방법도 있다. Codex는 `codex plugin marketplace add affaan-m/ECC` 후 `codex plugin add ecc@ecc`로, Cursor 등 다른 하네스는 저장소의 `install.sh --profile minimal --target <대상>`으로 설치한다. 에이전트 설정 파일의 보안 점검은 별도 패키지인 `npx ecc-agentshield scan`으로 수행한다.

## 5. 결론 및 시사점

* **에이전트 규율의 표준화:** 코딩 에이전트의 성능 차이는 단순히 LLM 모델 파라미터 크기뿐만 아니라, **어떤 하네스와 스킬로 에이전트의 작업 루프를 통제하느냐**에 크게 좌우된다.
* **지속 가능한 코드베이스 유지:** ECC 스킬을 도입하면 팀 단위 협업에서 AI가 생성하는 코드의 품질 표준(TDD, 보안, 문서화)을 일관되게 유지할 수 있다.

