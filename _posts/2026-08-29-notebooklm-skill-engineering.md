---
layout: post
comments: true
title: "NotebookLM으로 Claude 스킬 대량 생산하기 — 프롬프트 엔지니어링에서 스킬 엔지니어링으로"
description: "David Marco(@David_TornAI)의 X 스레드 분석. NotebookLM에 신뢰할 수 있는 소스를 업로드해 skill.md 파일을 자동 생성하고, Claude Code/Hermes Agent에 영구 AI 워커로 등록하는 6단계 워크플로우를 정리한다."
img: notebooklm-skill-engineering.webp
date: 2026-08-29 17:49:00 +0900
last_modified_at: 2026-08-29 17:49:00 +0900
tags: [notebooklm, skill-engineering, claude-code, hermes-agent, prompt-engineering, ai-workers, knowledge-management] # add tag
related: llm
categories: dev
---

David Marco(@David_TornAI)가 2026년 8월 28일 X에 올린 스레드 ["PEOPLE ARE USING NOTEBOOKLM TO MASS-PRODUCE SPECIALIZED CLAUDE SKILLS IN MINUTES"](https://x.com/david_tornai/status/2093337464215932962)는 **프롬프트 엔지니어링에서 스킬 엔지니어링으로 넘어가는 패러다임 전환**을 보여준다. 매 채팅마다 프롬프트를 다시 쓰는 대신, **NotebookLM을 지식 엔진으로, Claude/Hermes를 실행 엔진으로** 삼아 재사용 가능한 AI 워커를 구축하는 6단계 워크플로우다. 좋아요 1천여 회를 받으며 실무자들 사이에서 화제가 됐다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 대조 후 발행했다.)

<!--more-->

> **TL;DR:** NotebookLM에 PDF·문서·논문·내부 가이드 등 신뢰할 수 있는 소스를 업로드하고, **"이 소스만 바탕으로 완전한 skill.md 파일 생성"** 프롬프트를 던지면 구조화된 스킬 파일이 나온다. 이를 Claude Code/Hermes Agent 스킬 폴더에 드롭하면 **영구 AI 워커**가 된다. 한 스킬 = 한 작업(단일 책임) 원칙으로 카피라이팅, 코드 리뷰, 리서치 등 수십 개 스킬을 라이브러리화하면 "매번 프롬프트 재작성"에서 해방된다.

## 1. 배경: 왜 지금 '스킬 엔지니어링'인가?

기존 LLM 활용 패턴은 **채팅마다 프롬프트 작성 → 컨텍스트 설명 → 결과 검토 → 수정 → 다시 프롬프트 작성**의 반복이었다. 이 방식의 문제는 이렇다:

| 문제 | 영향 |
|------|------|
| **컨텍스트 재설명 비용** | 매 세션마다 도메인 지식, 규칙, 출력 형식 재전달 |
| **일관성 부족** | 프롬프트 미세 차이로 출력 품질·형식 편차 발생 |
| **환각·추측** | 검증되지 않은 지식에 의존해 부정확한 답변 생성 |
| **지식 자산화 불가** | 프롬프트가 개인/휘발성에 머물러 조직 자산으로 축적 안 됨 |

스레드는 이 문제를 **NotebookLM(지식 저장·구조화) + skill.md(실행 지시서) + Claude/Hermes(실행)** 3단 조합으로 해결한다. 이 블로그에서 다뤄온 [Hermes Agent 스킬 시스템]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html)(YAML 프론트매터 + 마크다운 바디)과 정확히 일치하는 접근이다.

## 2. 6단계 워크플로우 상세

### Step 1. 신뢰할 수 있는 소스 수집 (Gather Trusted Sources)

NotebookLM에 업로드할 소스들:

| 소스 유형 | 예시 | 우리 팀 해당 자산 |
|-----------|------|-------------------|
| **PDF/문서** | 기술 스펙, 아키텍처 문서, API 레퍼런스 | 인프라 설정 가이드, 서빙 스택 문서 |
| **연구 논문** | RAGAS, GraphRAG, 에이전트 벤치마크 논문 | 주간 논문 서베이로 쌓인 아카이브 |
| **아티클/블로그** | 기술 블로그, 업계 에세이 | 일간 블로그 분석 파이프라인 산출물 |
| **유튜브 트랜스크립트** | 컨퍼런스 발표, 튜토리얼 | 발표 세션 자막 |
| **내부 가이드** | 코딩 컨벤션, 배포 절차, 온보딩 문서 | `CLAUDE.md`, 블로그 글 템플릿 |

> **핵심 원칙**: "소스 품질 = 스킬 품질". 검증되지 않은 웹 스크랩보다 **공식 문서·승인된 내부 문서·피어 리뷰 논문**을 우선 업로드한다.

### Step 2. 지식을 skill.md로 변환 (Turn Knowledge into skill.md)

NotebookLM에 던질 프롬프트 템플릿:

```
이 소스들만 바탕으로 Hermes Agent용 skill.md 파일을 생성하세요.
다음 섹션을 모두 포함하세요:
- role: 스킬의 역할 한 줄 정의
- objectives: 달성하려는 목표 리스트
- workflow: 단계별 실행 워크플로우 (번호 매겨진 리스트)
- rules: 반드시 준수해야 할 규칙 (제약 조건 포함)
- constraints: 금지 사항, 하지 말 것들
- best_practices: 모범 사례·팁
- output_format: 출력 파일/형식 스펙
- examples: 입력-출력 예시 최소 2개

절대 외부 지식 사용 금지. 소스에 없는 내용은 "정보 없음"으로 표기.
출력은 YAML frontmatter + 마크다운 바디 형태의 완전한 skill.md 파일로.
```

**생성 예시** (블로그 포스트 작성 스킬 일부):

```yaml
---
name: blog-post-authoring
description: "Jekyll 블로그 포스트 초안 자동 작성: 프론트매터 규격 준수, 내부 링크 {{site.baseurl}} 형식, 태그 소문자 하이픈, 날짜 KST 검증"
version: 1.0.0
author: Hermes Agent
---
```

```markdown
## Workflow
1. 현재 KST 날짜 확인 (`date '+%Y-%m-%d %H:%M:%S %z'`)
2. `_drafts/0000-00-00-templete.md` 프론트매터 복사
3. 제목/설명/태그/카테고리/이미지 채우기
4. 본문 작성: 인트로 1문단 → `<!--more-->` → `> **TL;DR:**` → `## 1.` 섹션들
5. 코드블록 언어 태그 필수, 내부 링크 `{{site.baseurl}}/dev/...` 형식
6. 마무리에 고정 멘트 `*이 글은 Hermes Agent가 초안 작성 후 Claude가 검토·발행합니다.*`
7. 파일명: `_drafts/YYYY-MM-DD-slug.md` 저장
```

### Step 3. 스킬 설치 (Install the Skill)

생성된 `skill.md`를 스킬 디렉토리에 배치:

```bash
# Hermes Agent 기준
cp blog-post-authoring.md ~/.hermes/skills/writing/blog-post-authoring/SKILL.md

# 또는 skill_manage로 정식 등록
hermes skills install ~/.hermes/skills/writing/blog-post-authoring
```

확인:
```bash
hermes skills list | grep blog-post
hermes -s blog-post-authoring chat -q "RAGAS 평가 방법론 블로그 초안 써줘"
```

### Step 4. 한 스킬 = 한 작업 (One Skill, One Job)

거대 만능 프롬프트 대신 **단일 책임 스킬**로 분해:

| 스킬명 | 책임 범위 | 입력 | 출력 |
|--------|-----------|------|------|
| `blog-post-authoring` | Jekyll 포스트 초안 작성 | 주제, 핵심 포인트, 참고 링크 | `_drafts/YYYY-MM-DD-slug.md` |
| `ragas-evaluation` | RAGAS 4메트릭 평가 실행 | 골든셋 경로, 임계값 설정 | 평가 리포트 JSON + 마크다운 |
| `blog-analysis` | 기술 블로그 수집·분석·중복 검사 | 없음(배치 실행) | `_drafts/YYYY-MM-DD-slug.md` |
| `weekly-paper-survey` | arXiv 논문 수집·분류·리포트 생성 | 없음(주간 크론) | `_reports/YYYYMMDD_report.md` |
| `code-review-security` | 보안 취약점/공급망/설정 스캔 | PR diff, 리포지토리 경로 | 리뷰 코멘트 마크다운 |
| `team-onboarding` | 신규 멤버 아키텍처/컨벤션 가이드 | 질문, 역할 | 맞춤형 온보딩 문서 |

**원칙**: 스킬 하나당 **하나의 명확한 작업**만 담당. 복합 작업은 여러 스킬을 체이닝(`hermes -s skill1,skill2`)하거나 오케스트레이션 스킬로 조합.

### Step 5. 스킬 접지 — 검증된 정보만 사용 (Keep Skills Grounded)

| 비접지 프롬프트 | 접지된 스킬 |
|----------------|-------------|
| "베스트 프랙티스로 코드 리뷰해줘" | "저장소 `CLAUDE.md`의 보안 규칙(하드코딩 시크릿 금지 등)만 적용해 리뷰" |
| "RAGAS 평가 돌려줘" | "지정된 골든셋을 세그먼트별로 나눠 합의된 임계값으로 평가" |
| "블로그 글 써줘" | "`_drafts/0000-00-00-templete.md` 프론트매터 순서·필수 필드·태그 규칙·마무리 멘트 강제 적용" |

**효과**:
- ✅ **정확도 ↑** — 소스에 없는 내용 생성 안 함
- ✅ **일관성 ↑** — 같은 입력에 같은 출력 포맷 보장
- ✅ **환각 ↓** — "모름"이라고 솔직히 답함
- ✅ **편집 ↓** — 초안 품질이 바로 발행 가능 수준

### Step 6. 라이브러리 구축 — 영구 AI 워커 축적 (Build a Library)

```
~/.hermes/skills/
├── writing/
│   ├── blog-post-authoring/
│   ├── technical-doc-writing/
│   └── changelog-generation/
├── evaluation/
│   ├── ragas-evaluation/
│   ├── ablation-study/
│   └── benchmark-comparison/
├── analysis/
│   ├── nvidia-blog-analysis/
│   ├── paper-survey-weekly/
│   └── competitor-monitoring/
├── devops/
│   ├── k8s-manifest-review/
│   ├── helm-chart-linting/
│   └── cost-optimization-check/
└── onboarding/
    ├── team-onboarding/
    └── architecture-walkthrough/
```

**운영 루틴**:
1. **주간**: 새 논문/블로그/문서 → NotebookLM 업로드 → 관련 스킬 업데이트
2. **월간**: 스킬 사용 통계 확인 → 미사용 스킬 아카이브, 고빈도 스킬 프롬프트 최적화
3. **분기**: 스킬 카탈로그 문서화 → 온보딩 스킬에 반영

## 3. 적용 예시 — 어떤 스킬부터 만들까

| 우선순위 | 스킬 | 소스 자료 | 예상 효과 |
|----------|------|-----------|-----------|
| **P0** | `blog-post-authoring` | `CLAUDE.md`, 글 템플릿, 기존 포스트 표본 | 포스트 규격 위반 감소, 작성 시간 단축 |
| **P0** | `ragas-evaluation` | RAGAS 논문·공식 문서, 골든셋 스키마 | 평가 파이프라인 표준화, CI/CD 게이트 자동화 |
| **P1** | `blog-analysis` | 기존 배치 프롬프트, 중복 방지 규칙, 포스트 예시 | 배치 프롬프트 하드코딩 제거, 스킬로 버전 관리 |
| **P1** | `weekly-paper-survey` | 서베이 배치 프롬프트, 카테고리 정의, 출력 템플릿 | 동일 |
| **P2** | `code-review-security` | 보안 컨벤션 문서, OWASP Top 10 | PR 리뷰 자동화, 보안 게이트 강화 |
| **P2** | `architecture-decision-log` | ADR 템플릿, 기존 ADR 표본, 기술 스택 문서 | 결정 기록 자동화, 온보딩 가속 |

## 4. 주의점과 한계 대응

| 이슈 | 원인 | 대응 방안 |
|------|------|-----------|
| **NotebookLM 컨텍스트 윈도우 초과** | 대용량 문서셋(수백 MB) 업로드 불가 | 소스를 **주제별로 분할 업로드** → 별도 스킬로 생성 → 메타 스킬로 조합 |
| **skill.md 스키마 불일치** | NotebookLM이 YAML 프론트매터 형식을 어김 | 생성 후 `hermes skills check <skill명>` 필수 실행, 실패 시 재생성 프롬프트에 "스키마 엄수" 추가 |
| **소스 업데이트 시 스킬 동기화 누락** | 문서 변경 → 스킬 미반영 → 구버전 로직 실행 | **CI 파이프라인에 소스 변경 감지 → NotebookLM 재생성 → 스킬 배포** 자동화 검토 |
| **스킬 간 의존성/충돌** | 여러 스킬 동시 로드 시 규칙 충돌 | `hermes skills config`로 플랫폼별 활성화 제어, 네임스페이스(`writing/`, `eval/`)로 격리 |

## 5. Andrew Ng 포스트와의 연결: "트레이드오프 판단은 스킬에 인코딩"

이틀 전 분석한 [Andrew Ng 포스트]({{site.baseurl}}/dev/2026/08/29/andrew-ng-agentic-coding-fundamentals.html)는 **"에이전트가 코드를 짜줘도 트레이드오프 판단은 사람 몫"**이라 강조했다. 이 스레드는 그 **판단을 스킬에 명시적으로 인코딩**하는 실천법이다:

| Andrew Ng 주장 | 스킬 엔지니어링 구현 |
|----------------|---------------------|
| "나쁜 트레이드오프를 인지 못 하면 에이전트에게 맡김" | 스킬의 `rules`, `constraints`에 **트레이드오프 가드레일** 명시 (예: "지연시간 > 2초면 캐시 폴백 강제") |
| "데이터 아키텍처가 AI 품질 결정" | Step 1에서 **검증된 소스만 업로드** → 스킬이 참조하는 지식 기반 품질 보장 |
| "풀스택/아키텍처/운영 역량 필수" | **단일 책임 스킬**로 각 영역 전문성 캡슐화 → 필요 시 조합 실행 |

## 6. 마무리

> **"Instead of writing the same prompts repeatedly, they use curated knowledge to create reusable SKILL.md files that help Claude deliver consistent results for specific tasks."**
>
> — @David_TornAI, 스레드 첫 트윗

프롬프트 엔지니어링은 **일회성 대화 기술**이지만, 스킬 엔지니어링은 **영구 자산 구축 기술**이다. 이미 스킬 인프라(Claude Code, Hermes 등)를 쓰고 있다면, 남은 건 흩어진 문서·프롬프트·배치 잡을 skill.md로 이관·표준화하는 작업이다.

시작은 작게 잡는 편이 좋다: **NotebookLM 노트북 하나 → 문서 몇 개 업로드 → 스킬 1개 완전 생성·검증·배포**. 이 하나가 "지식 엔진 + 실행 엔진" 조합의 첫 실증이 된다.

## 참고

- [@David_TornAI 원문 스레드](https://x.com/david_tornai/status/2093337464215932962){:target="_blank"}
- [NotebookLM](https://notebooklm.google/){:target="_blank"}
- [Andrew Ng의 에이전트 시대 소프트웨어 기초 (관련글)]({{site.baseurl}}/dev/2026/08/29/andrew-ng-agentic-coding-fundamentals.html)
- [Hermes Agent 스킬 저장과 오케스트레이션 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html)
