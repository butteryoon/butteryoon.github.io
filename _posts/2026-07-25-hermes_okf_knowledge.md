---
layout: post
comments: true
title: "Hermes 파이프라인에 OKF 지식 번들 올리기"
description: "구글 Open Knowledge Format(OKF)을 Hermes 에이전트 파이프라인의 결정적 컨텍스트 소스로 올리는 법 — 마크다운 파일 디렉토리를 스킬로 감싸고, 벡터 RAG와 하이브리드로 조합"
img: api_code_title.jpg
date: 2026-07-25 18:00:00 +0900
last_modified_at: 2026-07-25 18:00:00 +0900
tags: [hermes, okf, knowledge, rag, llm, ai agent, markdown, nous research] # add tag
related: llm
categories: tools
---

[가드레일 Security 글]({{site.baseurl}}/tools/2026/07/25/hermes_guardrail_security.html)에서 무인 파이프라인에 보안 계층을 얹었다. 보안 다음으로 필요한 건 **에이전트가 "무엇을 아고" 행동하느냐**다. [OKF 리뷰 글]({{site.baseurl}}/dev/2026/07/22/okf_review.html)에서 정리한 구글 Open Knowledge Format — "파일 경로가 곧 개념의 식별자, 에이전트가 파일을 직접 읽는 결정적 지식" — 을 Hermes 파이프라인에 실제로 올리는 구성을 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** OKF는 `type` 하나만 필수인 마크다운 파일 디렉토리 규약이다. 이걸 Hermes에서는 **스킬 디렉토리(`SKILL.md` + 자료 파일)로 그대로 감싸면** 에이전트가 관련 작업 시 자동 로드해 결정적으로 읽는다. 큐레이션된 정형 지식(데이터 카탈로그·지표 정의·런북)은 OKF로, 대량 비정형은 기존 벡터 RAG로 — 하이브리드. 재미있는 점: Hermes 스킬 자체가 이미 "디렉토리 + 명세 = 지식" 구조라 OKF 철학과 태생적으로 맞물린다.

## OKF가 뭔지 (리뷰 글 요약)

[OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/tree/main/okf)은 벤더 중립 스펙으로, "LLM 위키" 패턴을 마크다운 파일 디렉토리로 표준화한 **지식 포맷**이다. 클라우드도 SDK도 아닌 그냥 파일 규약.

- 마크다운 파일 디렉토리 = 지식 번들. 개념 하나당 파일 하나, **파일 경로가 곧 개념의 식별자**
- YAML frontmatter: 필수 필드는 `type` 하나뿐. `title`/`description`/`resource`/`tags`/`timestamp`는 선택
- 일반 마크다운 링크로 개념 간 관계 표현 — 디렉토리 계층 위에 그래프가 얹힘
- 예약 파일: `index.md`(내비게이션·점진적 공개), `log.md`(변경 이력)
- 청킹·임베딩 없이 에이전트가 glob·grep·파일 읽기·링크 추적만으로 소비

자세한 RAG 대체 여부 논의는 [리뷰 글]({{site.baseurl}}/dev/2026/07/22/okf_review.html)에 있다. 이 글은 "그걸 Hermes에서 어떻게 쓰느냐"다.

## 증거 — Hermes 스킬은 이미 OKF다

이 글을 쓰는 환경의 실제 스킬 디렉토리:

```text
$ ls ~/.hermes/skills/orchestration/
SKILL.md

$ head -3 ~/.hermes/skills/orchestration/SKILL.md
---
name: orchestration
description: >-
  Use Orca orchestration for structured multi-agent coordination...
```

`orchestration/SKILL.md` — 디렉토리 하나에 명세 파일 하나. 파일 경로가 곧 그 지식의 식별자고, 사람도 읽고 에이전트도 로드한다. **이것이 OKF가 표준화하려는 "파일이 곧 지식"과 동일한 구조**다. 즉 OKF 지식 번들을 Hermes에 올릴 때 새로운 관행을 만드는 게 아니라, 이미 쓰는 스킬 구조를 지식 번들 규약에 맞춰 정돈만 하면 된다.

## Hermes에 OKF 번들을 올리는 구조

지식 번들을 스킬로 감싸다. 카탈로그 성격의 지식은 `type` 필수 frontmatter를 OKF 규약대로 채운다.

```
~/.hermes/skills/sales-knowledge/
├── SKILL.md                 # 언제 이 번들을 읽을지(description)
└── knowledge/               # OKF 번들
    ├── index.md             # 내비게이션
    ├── log.md               # 변경 이력
    ├── tables/orders.md     # frontmatter: type: table
    ├── tables/customers.md  # frontmatter: type: table
    └── metrics/weekly_active_users.md  # frontmatter: type: metric
```

`SKILL.md`는 로더 역할만 한다 — 번들 위치를 알리고, 언제 읽을지 서술한다.

```yaml
---
name: sales-knowledge
description: "Use when answering questions about sales DB schema, join keys, or WAU metric definitions."
---
# sales 지식 번들
1. 질문과 관련된 파일을 knowledge/ 에서 glob·grep 으로 찾는다
2. 링크([customers](tables/customers.md))를 따라 관련 개념을 읽는다
3. 정확한 조인 키·스키마는 파일 원문을 그대로 인용한다 (추론 금지)
```

에이전트가 "orders 테이블 조인 키가 뭐야?"를 받으면, 벡터 검색이 아니라 `knowledge/tables/orders.md`를 **정확히** 읽는다. 리뷰 글에서 지적한 OKF의 결정성 강점이 그대로 살아난다.

## 벡터 RAG와 하이브리드

OKF는 벡터 DB를 대체하는 게 아니다(리뷰 글 결론). 실무 구도는 하이브리드:

| 지식 종류 | 제공 방식 | 이유 |
| --- | --- | --- |
| 데이터 카탈로그, 지표 정의, 런북 | **OKF (결정적)** | 정확성·출처 추적·git 버전 관리가 핵심 |
| 수백만 건 민원·상담 이력·논문 | **벡터 RAG (확률적)** | 의미 검색·대규모 처리가 필요 |

큰 번들에서는 OKF 파일을 벡터 스토어에 올려 시맨틱 검색을 얹는 구성(공식 스펙 언급)도 가능하다. Hermes 파이프라인에서는 결정적 지식은 스킬(OKF)로, 비정형 대량은 기존 RAG 툴 호출로 분기하면 된다.

## 오케스트레이션 파이프라인에 끼우기

보안 글에서 만든 파이프라인에 OKF 번들을 추가:

```
cronjob(매주 토 "arxiv 수집+요약")
   ├─ security audit + doctor
   ├─ egress iron-proxy ON
   └─ guarded-pipeline 스킬
        └─ delegate_task(배치) 병렬 요약
cronjob(매주 일 "보고서 → 블로그 글 발행")
   └─ secrets(1Password) 로 키 로드
        └─ 출력 레일 통과 후 push
kanban(장기 테마 조사)
   └─ sales-knowledge 스킬로 카탈로그 결정적 참조  ← OKF 번들
```

에이전트가 무인으로 돌 때, "무엇을 아는가"가 코드가 아니라 **파일 번들**로 주어진다. 갱신은 파일 수정 + git commit이라 버전 관리·리뷰·롤백이 코드와 동일한 흐름이다 — 가드레일 글의 "무인 환경 위생"과 같은 철학.

## 마무리

OKF를 Hermes에 올리는 건 낯선 짓이 아니다. 스킬 디렉토리(`SKILL.md` + 자료)가 이미 OKF와 같은 "파일=지식" 구조이므로, 지식 번들을 규약(`type` 필수 frontmatter, `index.md`/`log.md`)에 맞춰 정돈하고 스킬로 감싸면 끝이다. 큐레이션된 정형 지식은 OKF로 결정적으로, 대량 비정형은 벡터 RAG로 — 이 하이브리드가 [AI Factory 글]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)의 "파일시스템 기반 워크스페이스 + 스킬" 컴포넌트를 Hermes에서 채우는 그림이다. 시리즈는 설치기 → 오케스트레이션 → 가드레일 Security → OKF 지식 번들로 이어진다.

## 참고

- [OKF 스펙 (GitHub: GoogleCloudPlatform/knowledge-catalog)](https://github.com/GoogleCloudPlatform/knowledge-catalog/tree/main/okf){:target="_blank"}
- [OKF는 RAG를 대체하나? (관련글)]({{site.baseurl}}/dev/2026/07/22/okf_review.html)
- [스킬 오케스트레이션 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html)
- [가드레일 Security 계층 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_guardrail_security.html)
- [NVIDIA AI Factory — 파일시스템/스킬 컴포넌트 (관련글)]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)
