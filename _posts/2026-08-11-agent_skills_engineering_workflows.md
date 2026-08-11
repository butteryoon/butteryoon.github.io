---
layout: post
comments: true
title: "Agent Skills: AI 코딩 에이전트용 프로덕션급 24개 엔지니어링 스킬 모음 — SDLC 전체를 커버하는 워크플로우"
description: "Google Chrome DevRel 리드 Addy Osmani가 공개한 agent-skills 프로젝트 분석. 소프트웨어 개발 생명주기 6단계(Define/Plan/Build/Verify/Review/Ship)를 24개 구조화된 스킬로 구현. 안티-합리화 테이블, 전문가 페르소나, 7개 슬래시 커맨드까지 실제 설치·사용법 정리."
img: tools_title.jpg
date: 2026-08-11 23:56:00 +0900
last_modified_at: 2026-08-12 00:50:00 +0900
tags: [ai-agent, agent-skills, engineering-workflow, production-grade, sdlc, addy-osmani, claude-code] # add tag
related: llm
categories: tools
---

[Hermes 대화 루프 심층]({{site.baseurl}}/dev/2026/07/25/hermes_conversation_loop.html) 글에서 에이전트 루프의 핵심으로 **툴 호출이 정화→디스패치→실행→관측 4단계를 거치는 것**을 다뤘다. 이번에는 그 원칙을 **실제 프로덕션 워크플로우로 구현한 스킬셋** — Addy Osmani의 **agent-skills** — 를 분석한다. 소프트웨어 개발 생명주기(SDLC) 전체를 6단계 24개 스킬로 분해하고, 각 단계에 품질 게이트와 안티-패턴 방지 장치를 내장한 이 프로젝트는 "에이전트가 지름길을 택하지 않게 하는" 구체적 메커니즘을 보여준다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 사실 검증 후 발행했다.)

<!--more-->

> **TL;DR:** agent-skills는 Google 엔지니어링 문화에서 검증된 원칙(Hyrum's Law, 테스트 피라미드, 체스터턴의 울타리, 트렁크 기반 개발)을 **AI 에이전트가 실행 가능한 24개 구조화된 워크플로우**로 변환한 오픈소스 프로젝트다. Define(명세) → Plan(계획) → Build(구현) → Verify(검증) → Review(리뷰) → Ship(배포) 6단계를 8개 슬래시 커맨드(`/spec`, `/plan`, `/build`, `/test`, `/review`, `/code-simplify`, `/ship`, `/webperf`)로 조작하며, 4개 전문가 페르소나(code-reviewer/test-engineer/security-auditor/web-performance-auditor)로 심층 리뷰를 위임한다. MIT 라이선스, 86.1k 스타, Claude Code/Cursor/Gemini/Codex/OpenCode/Windsurf/Copilot/Kiro/Command Code/Antigravity 전 플랫폼 지원.

## 1. agent-skills가 해결하려는 문제

일반적인 AI 코딩 프롬프트는 **"이 기능을 구현해줘"** 같은 **목표만 지정**한다. 반면 agent-skills는 **목표 + 과정 + 품질 게이트**까지 명시한다.

| 일반 프롬프트 | agent-skills 워크플로우 |
|--------------|------------------------|
| 목표만 지정 | 목표 + 단계별 실행 규칙 + 검증 기준 + 안티-패턴 방지 |
| 에이전트가 명세 없이 바로 코딩 | `/spec`으로 PRD 작성 강제 후 `/plan`으로 작업 분해 |
| 테스트 생략·보안 무시 흔함 | TDD 내장(`/build` 중 테스트 병행), 보안 스킬 별도(`/review` 단계) |
| 리뷰는 사람이 직접 | code-reviewer/test-engineer/security-auditor 페르소나로 자동 위임 |
| 배포는 수동 | `/ship`으로 CI/CD·기능 플래그·롤백·ADR까지 포함 |

핵심 철학: **"방법론을 산문(prose)이 아닌 프로세스로 담아야 한다"** — Hyrum's Law, 테스트 피라미드, 체스터턴의 울타리, 트렁크 기반 개발 같은 검증된 원칙이 **추상적 가이드라인이 아닌, 에이전트가 단계별로 실행할 수 있는 구체적 워크플로우**로 내장돼 있다.

## 2. 6단계 개발 생명주기와 24개 스킬 상세

agent-skills는 소프트웨어 개발 생명주기를 **6개 단계**로 구분하고, 각 단계에 **전용 스킬**을 배치한다. 24개 스킬 중 1개는 메타 스킬(`using-agent-skills`)이다.

### 2.1 Define(정의) — 무엇을 만들지 명확히

| 스킬 | 역할 | 사용 시점 |
|------|------|-----------|
| **using-agent-skills** | 들어오는 작업을 적절한 스킬 워크플로에 매핑, 공통 운영 규칙 정의 | 세션 시작, 어떤 스킬을 쓸지 결정할 때 |
| **interview-me** | 한 질문씩 던지며 사용자가 *진짜 원하는 것*을 추출(~95% 신뢰도까지) | 요구사항이 모호하거나 "interview me" 호출 시 |
| **idea-refine** | 발산/수렴 구조화 사고로 막연한 아이디어를 구체적 제안으로 | 러프한 컨셉이 탐색이 필요할 때 |
| **spec-driven-development** | 목적·명령·구조·코드 스타일·테스트·경계를 담은 PRD 작성 | 새 프로젝트/기능/중대 변경 시작 전 |

> **안티-합리화 테이블 예시** (spec-driven-development): "명세 쓸 시간이 없다" → **"명세 없는 구현은 리팩터링 비용 10배 증가"** — 에이전트가 단계를 건너뛰려 할 때 구체적 반박 근거를 제시한다.

### 2.2 Plan(계획) — 작업 분해와 완료 기준

| 스킬 | 역할 | 사용 시점 |
|------|------|-----------|
| **planning-and-task-breakdown** | 명세를 검증 가능한 작은 태스크로 분해, 수락 기준·의존성 순서 부여 | 명세가 있고 구현 단위가 필요할 때 |

### 2.3 Build(구현) — 가장 많은 7개 스킬

| 스킬 | 역할 | 사용 시점 |
|------|------|-----------|
| **incremental-implementation** | 얇은 수직 슬라이스: 구현→테스트→검증→커밋. 기능 플래그, 안전한 기본값, 롤백 친화적 변경 | 둘 이상의 파일을 건드리는 모든 변경 |
| **test-driven-development** | Red-Green-Refactor, 테스트 피라미드(80/15/5), 테스트 크기, DAMP over DRY, Beyonce Rule, 브라우저 테스트 | 로직 구현, 버그 수정, 동작 변경 시 |
| **context-engineering** | 에이전트에게 적절한 정보를 적절한 시점에 제공 — 규칙 파일, 컨텍스트 패킹, MCP 통합 | 세션 시작, 태스크 전환, 출력 품질 저하 시 |
| **source-driven-development** | 모든 프레임워크 결정을 공식 문서에 근거 — 검증, 출처 인용, 미검증 플래그 | 권위 있고 출처가 명시된 코드가 필요할 때 |
| **doubt-driven-development** | 비자명 결정에 대한 적대적 프레시 컨텍스트 리뷰 — CLAIM→EXTRACT→DOUBT→RECONCILE→STOP, 선택적 교차 모델 에스컬레이션 | 스테이크가 높음(프로덕션/보안/불가역), 낯선 코드, 지금 검증이 나중에 디버깅보다 쌀 때 |
| **frontend-ui-engineering** | 컴포넌트 아키텍처, 디자인 시스템, 상태 관리, 반응형, WCAG 2.1 AA 접근성 | 사용자 대면 인터페이스 구축/수정 시 |
| **api-and-interface-design** | 계약 우선 설계, Hyrum's Law, One-Version Rule, 에러 시맨틱, 경계 검증 | API, 모듈 경계, 공개 인터페이스 설계 시 |

### 2.4 Verify(검증) — 실제로 동작하는지 증명

| 스킬 | 역할 | 사용 시점 |
|------|------|-----------|
| **browser-testing-with-devtools** | Chrome DevTools MCP로 런타임 실데이터 획득 — DOM 검사, 콘솔 로그, 네트워크 트레이스, 성능 프로파일링 | 브라우저에서 돌아가는 모든 것 구축/디버깅 시 |
| **debugging-and-error-recovery** | 5단계 트리아지: 재현→국소화→축소→수정→가드. 스톱-더-라인 규칙, 안전한 폴백 | 테스트 실패, 빌드 깨짐, 예기치 않은 동작 시 |

### 2.5 Review(리뷰) — 머지 전 품질 게이트

| 스킬 | 역할 | 사용 시점 |
|------|------|-----------|
| **code-review-and-quality** | 5축 리뷰, 변경 크기(~100줄), 심각도 라벨(Nit/Optional/FYI), 리뷰 속도 규범, 분할 전략 | 모든 변경 머지 전 |
| **code-simplification** | 체스터턴의 울타리, 500줄 규칙, 정확한 동작 보존하며 복잡도 감소 | 코드는 동작하나 읽기/유지보수 어려울 때 |
| **security-and-hardening** | OWASP Top 10 방지, 인증 패턴, 시크릿 관리, 의존성 감사, 3계층 경계 시스템 | 사용자 입력/인증/데이터 저장/외부 연동 처리 시 |
| **performance-optimization** | 측정 우선 — Core Web Vitals 타겟, 프로파일링 워크플로, 번들 분석, 안티-패턴 탐지 | 성능 요구사항 있거나 회귀 의심 시 |

### 2.6 Ship(배포) — 확신을 가지고 프로덕션으로

| 스킬 | 역할 | 사용 시점 |
|------|------|-----------|
| **git-workflow-and-versioning** | 트렁크 기반 개발, 원자적 커밋, 변경 크기(~100줄), 커밋-세이브포인트 패턴 | 모든 코드 변경 시(항상) |
| **ci-cd-and-automation** | Shift Left, Faster is Safer, 기능 플래그, 품질 게이트 파이프라인, 실패 피드백 루프 | 빌드/배포 파이프라인 구축/수정 시 |
| **deprecation-and-migration** | 코드=부채 마인드셋, 필수/권고 디프리케이션, 마이그레이션 패턴, 좀비 코드 제거 | 구 시스템 제거, 사용자 마이그레이션, 기능 선셋 시 |
| **documentation-and-adrs** | 아키텍처 결정 기록(ADR), API 문서, 인라인 문서 표준 — **왜**를 문서화 | 아키텍처 결정, API 변경, 기능 출시 시 |
| **observability-and-instrumentation** | 구조화 로깅, RED 메트릭, OpenTelemetry 트레이싱, 증상 기반 알림 — 만들면서 계측 | 텔레메트리 추가, 프로덕션 실행 코드 출시 시 |
| **shipping-and-launch** | 사전 출시 체크리스트, 기능 플래그 라이프사이클, 단계적 롤아웃, 롤백 절차, 모니터링 설정 | 프로덕션 배포 준비 시 |

## 3. 4개 전문가 페르소나 — 심층 리뷰 위임

기본 스킬 외에 **특정 역할에 특화된 4개 에이전트 페르소나**를 제공한다. 하나의 에이전트가 모든 역할을 흉내 내는 것보다 **전문가 관점의 깊이 있는 피드백**을 보장한다.

| 페르소나 | 역할 | 관점 |
|----------|------|------|
| **code-reviewer** | 시니어 스태프 엔지니어 | 5축 코드 리뷰, "스태프 엔지니어가 승인할까?" 기준 |
| **test-engineer** | QA 스페셜리스트 | 테스트 전략, 커버리지 분석, Prove-It 패턴 |
| **security-auditor** | 보안 엔지니어 | 취약점 탐지, 위협 모델링, OWASP 평가 |
| **web-performance-auditor** | 웹 성능 엔지니어 | Core Web Vitals 감사(Quick/Deep 모드), 메트릭 정직성 규칙, `/webperf`로 호출 |

`docs/agents.md`에 **의사결정 매트릭스, 오케스트레이션 규칙, 페르소나-스킬-슬래시 커맨드 조합법**이 상세히 문서화돼 있다.

## 4. 지원 플랫폼 및 설치 — 어디서든 동작

agent-skills는 **마크다운 기반**이라 시스템 프롬프트/지시 파일을 받는 모든 에이전트에서 동작한다. 주요 플랫폼별 설치법:

```bash
# Claude Code (마켓플레이스)
/plugin marketplace add addyosmani/agent-skills

# 또는 로컬 클론
git clone https://github.com/addyosmani/agent-skills ~/.claude/skills/agent-skills

# Cursor
# .cursor/skills/에 동기화, .cursor/rules/*.mdc에 짧은 정책만

# Gemini CLI
gemini skills install https://github.com/addyosmani/agent-skills.git --path skills

# Codex (CLI v0.122+)
codex plugin marketplace add addyosmani/agent-skills
codex plugin add agent-skills@agent-skills

# Command Code
cmd skills add addyosmani/agent-skills            # 프로젝트 레벨
cmd skills add addyosmani/agent-skills --global   # 전역(~/.commandcode/skills/)

# Antigravity CLI
agy plugin install https://github.com/addyosmani/agent-skills.git

# Windsurf / OpenCode / GitHub Copilot / Kiro IDE
# 각 플랫폼 docs/*-setup.md 참조
```

**슬래시 커맨드 8개**로 워크플로우 진입:

```bash
/spec            # 명세 작성부터 시작 (Define)
/plan            # 작업 계획 수립 (Plan)
/build           # 점진적 구현 (Build)
/test            # 테스트 및 검증 (Verify)
/review          # 코드 리뷰 (Review)
/code-simplify   # 복잡도 감소 (Review)
/ship            # 프로덕션 배포 (Ship)
/webperf         # Core Web Vitals 감사 (web-performance-auditor)
```

## 5. 실전 적용 포인트 — 우리 프로젝트에 어떻게 쓸까

| 우리 상황 | 적용 스킬 | 기대 효과 |
|-----------|-----------|-----------|
| 문서 파싱 → 검증 → 산출물 생성 에이전트 파이프라인 | `spec-driven-development` → `incremental-implementation` → `test-driven-development` → `debugging-and-error-recovery` | 명세 없는 구현 방지, 얇은 슬라이스로 리스크 격리, 테스트 내장 |
| 에이전트 루프 분리 아키텍처 문서화 | `documentation-and-adrs` + `code-simplification` | ADR로 아키텍처 결정 근거 보존, 복잡도 관리 |
| 보안 강화(개인정보/인증/외부 연동) | `security-and-hardening` + `security-auditor` 페르소나 | OWASP Top 10 체계적 차단, 전문가 관점 감사 |
| 배포 자동화(CI/CD/기능 플래그/롤백) | `ci-cd-and-automation` + `shipping-and-launch` + `git-workflow-and-versioning` | Shift Left, 단계적 롤아웃, 100줄 단위 원자적 커밋 |
| 성능 병목 분석(프롬프트 캐싱/응답 지연) | `performance-optimization` + `web-performance-auditor` | 측정 우선, Core Web Vitals 타겟, 안티-패턴 탐지 |
| 신규 팀원 온보딩/워크플로우 표준화 | `using-agent-skills` + `context-engineering` | 적절한 스킬 자동 매핑, 컨텍스트 엔지니어링으로 품질 균일화 |

## 6. 비교: agent-skills vs Superpowers vs Matt Pocock's Skills

프로젝트 공식 문서(`docs/comparison.md`)와 [LinkedIn 헤드투헤드 실험](https://www.linkedin.com/pulse/superpowers-vs-agent-skills-faster-shipping-safer-reasoning-om-mishra-dzakf/) 기준 요약 (스타 수는 2026-08-12 GitHub API 실측):

| 차원 | agent-skills | Superpowers (obra) | Matt Pocock's skills |
|------|--------------|-------------|---------------------|
| **핵심 초점** | SDLC 전체 생명주기 + 품질 게이트 + 인리포 3단계 eval 프레임워크 | 자율적·추론 중심 실행 — 서브에이전트, 엄격한 파이프라인, worktree 격리 | 한 전문가의 일상 워크플로우를 증류한 견고한 Claude Code 툴킷 ("grill me" 인터로게이션 루프) |
| **스킬 수** | 24 (메타 1 + 23 라이프사이클) | ~15 | ~20 |
| **리뷰 페르소나** | 4개 (code-reviewer, test-engineer, security-auditor, web-performance-auditor) | 서브에이전트 방식 | 없음 |
| **플랫폼 지원** | 10+ (Claude, Cursor, Gemini, Codex, OpenCode, Windsurf, Copilot, Kiro, Command Code, Antigravity) | Claude Code 중심 | Claude Code 중심 |
| **스타 수** | 86.1k | 270k+ | 213k+ |

> **언제 agent-skills인가**: 팀/프로젝트 단위로 **프로덕션급 엔지니어링 프로세스 전체를 에이전트에 내재화**하고 싶을 때. 특히 보안·성능·리뷰·배포까지 **엔드투엔드 품질 게이트**가 필요할 때.

## 7. 설치 후 바로 해볼 것 — 5분 체험

```bash
# 1. Claude Code에서 설치
/plugin marketplace add addyosmani/agent-skills

# 2. 새 세션에서 메타 스킬로 시작
/using-agent-skills "로그인 기능 추가해줘"

# 3. 자동으로 적절한 스킬 매핑 확인
# → interview-me → spec-driven-development → planning-and-task-breakdown → ...

# 4. 개별 스킬 직접 호출도 가능
/spec "JWT 기반 인증 API 설계"
/build
/test
/review
/ship
```

`using-agent-skills` 메타 스킬이 **들어온 작업을 분석해 적절한 워크플로우로 라우팅**하므로, 처음에는 이 하나만 기억하면 된다.

## 8. 오픈웨이트 모델(로컬 LLM)에서의 동작 고려사항

agent-skills는 특정 모델·플랫폼에 종속되지 않게(model-agnostic) 설계됐다. 스킬 자체가 마크다운 파일(SKILL.md + references/)로만 구성돼 있어, 시스템 프롬프트/지시 파일을 받는 모든 에이전트/모델에서 동작한다. 단, 오픈웨이트 모델로 로컬 구동 시 다음 포인트를 고려해야 한다.

| 요소 | 영향 | 대응 |
|------|------|------|
| **컨텍스트 길이** | 24개 스킬 + references 전체를 시스템 프롬프트에 넣으면 수만 토큰 소모 | `using-agent-skills` 메타 스킬로 **필요한 것만 동적 로드** 권장 |
| **지시 추적 능력** | 작은 모델(7B~14B)은 다단계 워크플로우(CLAIM→EXTRACT→DOUBT…) 준수율 낮음 | 핵심 스킬만 선별(`/spec`, `/build`, `/test`, `/review` 등) |
| **함수 호출/툴 사용** | 슬래시 커맨드(`/spec` 등)는 플랫폼 쪽에서 파싱 → 모델에 전달 | 모델이 툴 호출을 지원해야 전체 워크플로우 순환 가능 |
| **안티-합리화 테이블** | "이 단계 건너뛰어도 되지 않을까?" 반박 논리 이해 필요 | 프롬프트에 **예시 대화(few-shot)** 추가로 보완 |

**로컬에서 돌려보려면:**

```bash
# 1. 로컬 클론
git clone https://github.com/addyosmani/agent-skills ~/.local/share/agent-skills

# 2. Ollama/vLLM 등으로 모델 서빙 (예: qwen2.5-coder:14b, codellama:13b)
ollama run qwen2.5-coder:14b

# 3. 에이전트 프레임워크 연동 예시 (OpenCode 스타일)
# AGENTS.md에 스킬 경로 등록 후 skill 툴로 호출
```

**제한점:**
- 공식 테스트/벤치마크는 Claude/Cursor/Codex 등 상용 모델 기준 — 오픈웨이트 모델에서의 성공률 데이터는 공개되지 않음
- `doubt-driven-development` 같은 다단계 적대적 리뷰는 추론 능력이 강한 모델(32B+ 또는 추론 특화)에서만 실효적
- 소형 모델(7B 이하)에선 메타 스킬 없이 **개별 스킬만 프롬프트에 주입**해 쓰는 것이 현실적

> **요약**: 아키텍처상 오픈웨이트 모델에서도 **동작은 함**. 단, 컨텍스트 예산·추론 깊이·툴 호출 지원 여부에 따라 **전체 24스킬 풀세트 대신 핵심 4~6개만 선별 주입**하는 방식이 실용적이다.

## 마무리

agent-skills는 **"에이전트가 지름길을 택하지 않게 하는"** 구체적 메커니즘을 보여준다:

1. **프로세스로 방법론 내재화** — 산문이 아닌 실행 가능한 워크플로우로
2. **안티-합리화 테이블** — 에이전트의 "이 단계 건너뛰어도 되지 않을까?"를 구체적 반박으로 차단
3. **전문가 페르소나 분리** — 한 에이전트가 모든 역할 흉내 대신, 역할별 깊이 있는 리뷰
4. **전 플랫폼 포터빌리티** — 마크다운 기반으로 10+ 플랫폼에서 동일 워크플로우 사용 가능

우리 블로그 기존 글들과의 연결:

- [Hermes 대화 루프 심층]({{site.baseurl}}/dev/2026/07/25/hermes_conversation_loop.html) — 툴 호출 정화→디스패치→실행→관측 4단계가 agent-skills의 Define/Plan/Build/Verify/Review/Ship 6단계 품질 게이트와 대응
- [Hermes 오케스트레이션]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html) — 멀티 에이전트 오케스트레이션에서 페르소나 기반 위임 패턴 참고
- [가드레일 Security]({{site.baseurl}}/tools/2026/07/25/hermes_guardrail_security.html) — `security-and-hardening` 스킬과 `security-auditor` 페르소나로 구현체 확보
- [Prompt Caching 실전 가이드]({{site.baseurl}}/dev/2026/08/05/prompt-caching-guide.html) — `performance-optimization` 스킬의 측정 우선 접근법과 연결

다음 글에서는 **agent-skills의 `doubt-driven-development` 스킬을 실제 에이전트 파이프라인에 적용해 본 회고**를 다뤄볼 예정이다.

## 참고

- [agent-skills GitHub 저장소](https://github.com/addyosmani/agent-skills){:target="_blank"}
- [공식 문서 사이트](https://skills.addy.ie/){:target="_blank"}
- [PyTorchKR 한국어 정리 글](https://discuss.pytorch.kr/t/agent-skills-ai-20/9763){:target="_blank"}
- [비교 문서: agent-skills vs Superpowers vs Matt Pocock's Skills](https://github.com/addyosmani/agent-skills/blob/main/docs/comparison.md){:target="_blank"}
- [헤드투헤드 실험 리포트](https://www.linkedin.com/pulse/superpowers-vs-agent-skills-faster-shipping-safer-reasoning-om-mishra-dzakf/){:target="_blank"}
- [스킬 해부도 명세](https://github.com/addyosmani/agent-skills/blob/main/docs/skill-anatomy.md){:target="_blank"}
- [채택 가이드](https://github.com/addyosmani/agent-skills/blob/main/docs/adoption-guide.md){:target="_blank"}
- [에이전트 페르소나 문서](https://github.com/addyosmani/agent-skills/blob/main/docs/agents.md){:target="_blank"}
