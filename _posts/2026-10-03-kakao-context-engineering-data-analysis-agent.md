---
layout: post
comments: true
title: "카카오 테크 — 데이터 분석 에이전트를 만들며 배운 컨텍스트 설계"
description: "카카오 기술 블로그 해설. 데이터 분석 에이전트의 컨텍스트를 시스템 프롬프트·도구 명세서·메시지 배열로 나눠 설계하고, 테이블 목록 인덱스로 점진적 발견을, SQL 서브에이전트로 중간 기록 격리를 구현한 과정을 정리한다."
img: kakao_context_engineering_title.webp
date: 2026-10-03 18:00:00 +0900
last_modified_at: 2026-10-03 18:00:00 +0900
tags: [kakao-tech, llm, llm-agent, context-engineering, agent, subagent]
related: llm-agent
categories: [kakao-tech, llm-agent, ai-research]
source_url: https://tech.kakao.com/posts/838
source_date: 2026-10-02
---

카카오 기술 블로그에 10월 2일 올라온 [「데이터 분석 에이전트를 만들며 배운 컨텍스트 설계」](https://tech.kakao.com/posts/838){:target="_blank"}를 읽고 정리했다. 사용자의 비즈니스 질문을 받아 데이터를 찾고 SQL을 작성·실행한 뒤 결과를 설명하는 에이전트를 만들면서, 시스템 프롬프트와 도구 명세서와 메시지 배열에 각각 무엇을 담았는지 공개한 글이다. 테이블 목록을 인덱스로 주는 점진적 발견, SQL 작업을 서브에이전트로 떼어낸 구조도 함께 다룬다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 시스템 프롬프트에는 역할·안전 규칙·여러 도구에 공통인 사용 원칙만 남기고, 도구별 호출 조건·입력값 의미·결과 표시 방식은 도구 명세서로 옮겼다. 테이블 탐색은 별도 탐색 에이전트 대신 서비스별 테이블 목록을 시스템 프롬프트에 인덱스로 넣고, 메인 에이전트가 후보를 고른 뒤 스키마 조회 도구로 상세 정보를 가져오게 바꿨다. SQL 작성·실행·오류 수정은 도구로 호출하는 서브에이전트가 맡아, 실패한 SQL과 오류 메시지가 메인 메시지 배열에 쌓이지 않게 했다.

## 1. 컨텍스트의 세 요소

원문은 컨텍스트를 모델이 다음 행동을 정하거나 답변을 만들 때 입력으로 받는 정보로 정의하고, 이를 세 요소로 나눠 설명한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"이 글에서는 컨텍스트를 시스템 프롬프트, 도구 명세서, 메시지 배열의 세 요소를 중심으로 살펴보겠습니다."</blockquote></details>

| 요소 | 담는 내용 | 비고 |
|------|-----------|------|
| **시스템 프롬프트** | 역할 정의, 안전 규칙, 판단 기준과 기본값, 공통 도구 사용 가이드, 답변 가이드, 사전 지식(서비스 설명·테이블 목록) | 개별 도구의 역할·호출 조건·입력값 의미는 넣지 않는다 |
| **도구 명세서** | 도구의 역할, 호출 조건, 입력값의 의미와 선택 기준, 결과 표시 방식 | 원문이 든 Claude Code 사례에서 도구 명세서는 약 15k 토큰, 시스템 프롬프트는 3.7k 토큰이다 |
| **메시지 배열** | 사용자와 LLM의 대화, 도구 호출과 실행 결과를 순서대로 쌓은 목록 | 다음 행동을 정하는 입력이자, "기간을 한 달로 늘려줘" 같은 후속 요청을 처리할 때 참고하는 기록 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"아래 Claude Code 사례에서 도구 명세서의 분량은 약 15k 토큰으로, 시스템 프롬프트(3.7k 토큰)의 약 4배입니다."</blockquote></details>

### 1-1. 도구 사용 조건을 시스템 프롬프트에서 명세서로 옮긴 이유

GPT-4로 에이전트를 처음 만들던 때에는 모델이 상황에 맞는 도구와 작업 순서를 스스로 고르는 능력이 지금보다 약했다. 그래서 도구 명세서에는 도구가 하는 일만 짧게 적고, 시스템 프롬프트에 'Workflow' 항목을 따로 두어 테이블 탐색 → SQL 생성·실행 순서와 "이미 알고 있는 데이터라면 검색을 생략한다" 같은 상황별 지침을 모아 두었다.

모델 성능이 올라간 뒤에는 순서 판단을 모델에 맡기고, 각 도구의 사용 조건을 해당 도구 명세서에 적는 구조로 바꿨다. Anthropic 도구 정의 가이드의 "Provide extremely detailed descriptions" 권고와 같은 방향이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"이후 모델의 성능이 높아지면서, 모델이 사용자의 요청과 현재 상황을 바탕으로 필요한 도구와 작업 순서를 판단하도록 구성을 바꾸었습니다. 이를 위해 각 도구의 사용 조건을 해당 도구의 명세서에 작성했습니다."</blockquote></details>

### 1-2. 차트 생성 도구 명세서

원문이 공개한 차트 생성 도구 명세서다. description 안에 호출 조건·입력 원칙·결과 표시를 절로 나눠 적고, `chart_type`은 enum으로 값을 제한하면서 분석 목적별 선택 기준을 함께 달았다.

```json
{
  "name": "generate_chart",
  "description": "실행 결과를 기반으로 Vega-Lite Spec을 생성한다.\n\n## 호출 조건\n- 사용자가 명시적으로 차트 생성을 요청한 경우에만 호출한다.\n- 앞서 실행한 SQL의 결과가 있어야 한다.\n\n## 입력 원칙\n- 차트 생성기는 대화를 보지 못하므로, 차트로 확인하려는 내용과 강조할 부분을 instruction으로 전달한다.\n\n## 결과 표시\n- 차트는 화면에 표시된다. 차트 데이터를 글로 옮기지 않고 분석에서 얻은 인사이트를 한두 문장으로 덧붙인다.",
  "parameters": {
    "type": "object",
    "properties": {
      "chart_type": {
        "type": "string",
        "enum": ["bar","line","area","point","pie","donut","heatmap"],
        "description": "차트 타입. 시계열 추세는 line, 시간에 따른 누적량이나 구성 변화는 area, 카테고리 비교는 bar, 구성비는 pie/donut, 두 변수 관계는 point, 2차원 분포는 heatmap을 사용한다."
      },
      "instruction": {
        "type": "string",
        "description": "이 차트로 확인하려는 내용과 강조할 부분을 한두 문장으로 설명합니다 (예: '9월 일별 DAU 추세와 주말 하락을 보여줘'). 생성기는 이 설명을 참고해 축 제목과 정렬 방식 등을 정하고, 질문과 강조점이 드러나도록 차트를 구성합니다."
      }
    },
    "required": ["chart_type","instruction"]
  }
}
```

눈여겨볼 곳은 `instruction` 필드다. 이 도구는 내부에서 별도 LLM을 불러 Vega-Lite Spec을 만드는데, 그 LLM은 사용자와의 대화를 보지 못한다. 그래서 메인 에이전트가 차트로 보여줄 내용과 강조점을 입력값에 직접 적어 넘긴다.

## 2. 관련 기능은 하나의 도구로 묶는다

기능마다 도구를 따로 만들면 에이전트가 읽을 명세서가 늘고, 고를 후보도 많아지고, 공통 설명과 입력 규칙이 여러 명세서에 반복된다. 원문은 John Ousterhout의 『A Philosophy of Software Design』이 말하는 '깊은 모듈'(단순한 인터페이스로 풍부한 기능을 제공하는 모듈)을 끌어와, 관련 기능을 한 도구로 묶고 세부 처리는 도구 안에서 하도록 했다.

### 2-1. 노트 저장과 수정을 한 도구로

자주 쓰는 SQL을 노트로 저장해 재사용하는 기능에서는 생성과 수정을 `upsert_note` 하나로 처리한다.

```json
{
  "name": "upsert_note",
  "description": "SQL을 포함한 노트를 새로 저장하거나 기존 노트를 수정한다. 사용자가 저장 또는 수정을 명시적으로 요청했을 때 사용한다.",
  "parameters": {
    "type": "object",
    "properties": {
      "note_key": { "type": ["string","null"], "description": "수정할 기존 노트의 key. 새 노트를 생성할 때는 null을 전달한다." },
      "body": { "type": "string", "description": "저장할 노트 본문. 기존 노트를 수정할 때는 수정이 반영된 전체 본문을 전달한다." }
    },
    "required": ["note_key","body"]
  }
}
```

에이전트는 같은 도구를 부르면서 `note_key` 값으로 생성과 수정을 가른다.

### 2-2. 여러 단계 작업도 한 도구로

SQL 작성 → 실행 → 오류 시 수정 → 재실행으로 이어지는 과정도 하나의 도구로 묶었다. 이 도구의 실체가 SQL 작성·실행을 전담하는 서브에이전트다(4절).

## 3. 점진적 발견: 테이블 목록을 인덱스로

Anthropic은 「Effective context engineering for AI agents」에서 에이전트가 스스로 탐색하며 관련 컨텍스트를 조금씩 찾아 나가는 방식을 '점진적 발견(progressive disclosure)'이라 불렀다. 원문은 Claude Code가 파일명 검색 → 코드 내용 검색 → 관련 코드 읽기 → 정의와 사용처 추적 순으로 로그인 코드를 찾아가는 과정과, 이름·설명을 먼저 보고 필요한 스킬의 상세 지침만 읽는 Skills를 예로 든다.

데이터 분석 에이전트에서는 이 원리를 테이블 탐색에 적용했다.

- **이전**: 별도 탐색 에이전트가 서비스와 테이블을 검색하고 상세 스키마를 가져왔다. 메인 에이전트는 어떤 테이블에 접근할 수 있는지 몰라서, 필요한 테이블이 아예 없을 때도 탐색을 반복한 뒤에야 분석할 수 없다고 판단했다.
- **이후**: 서비스 설명과 접근 가능한 전체 테이블 목록을 서비스별로 정리해 시스템 프롬프트에 넣었다. 메인 에이전트가 목록에서 후보를 고르고, 스키마 조회 도구로 컬럼 이름·타입·샘플값·분석 규칙을 가져온다.

바꾼 뒤에는 필요한 테이블이 목록에 없다는 사실을 더 일찍 알아차렸고, 테이블을 찾는 데 드는 턴 수도 줄었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"서비스별 테이블 목록은 스킬의 이름과 설명처럼, 어떤 정보가 있고 어디서 더 찾을 수 있는지 알려주는 인덱스 역할을 합니다. 에이전트는 테이블 이름과 짧은 설명으로 후보를 좁힌 뒤, 필요한 테이블의 상세 스키마를 조회합니다."</blockquote></details>

## 4. 서브에이전트로 중간 기록 격리

처음에는 메인 에이전트가 SQL 작성·실행과 오류 수정·재실행을 모두 맡았다. 그러다 보니 실패한 SQL과 오류 메시지가 메시지 배열에 계속 쌓였고, 문제가 풀린 뒤에도 이 기록이 다음 분석의 입력에 섞여 들어갔다.

개선한 구조는 이렇다.

- 메인 에이전트가 SQL 담당 서브에이전트를 **도구로 호출**하면서 분석 목적, 사용할 테이블, 조회 기간, 필터 조건, 집계 기준을 넘긴다. 선택한 테이블의 스키마와 분석 규칙도 서브에이전트 컨텍스트에 함께 들어간다.
- 메인 에이전트의 대화 기록은 넘어가지 않는다. 앞서 확인한 조건은 작업 지시에 직접 적어야 한다.
- 서브에이전트는 자기 컨텍스트에서 SQL 작성·실행·수정을 처리하고, **최종 SQL, 실행 상태, 결과 또는 실패 사유**만 돌려준다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"반면 실패한 SQL과 오류 메시지는 문제가 해결되면 다시 참고할 일이 거의 없으므로 서브에이전트의 컨텍스트에서 처리하고, 메인 에이전트에는 최종 결과만 전달했습니다."</blockquote></details>

### 4-1. 테이블 탐색은 메인, SQL은 서브인 이유

테이블 탐색에서는 오히려 별도 에이전트를 없애고 메인 에이전트가 직접 스키마를 조회하게 했다. 두 결정이 반대로 보이지만 기준은 하나다. 작업 중 얻은 정보를 나중에도 쓰는가.

| 작업 | 나중에도 쓰는가 | 컨텍스트 보관 위치 |
|------|----------------|-------------------|
| 테이블 목록·스키마 | 쓴다 (SQL 작업 지시와 결과 해석에 계속 필요) | 메인 에이전트 |
| 실패한 SQL·오류 메시지 | 거의 안 쓴다 (해결 후 참고할 일이 드묾) | 서브에이전트 |

## 5. 정리하며 얻은 시사점

아래는 원문 내용을 바탕으로 한 정리다.

1. **도구 명세서에 분량을 아끼지 않는다** — 호출 조건, 입력값의 의미와 선택 기준, 결과 표시 방식까지 적는다. Claude Code 사례에서도 명세서가 시스템 프롬프트의 약 4배였다.
2. **인덱스와 상세 조회를 나눈다** — 전체 스키마를 프롬프트에 넣지 않고, 무엇이 있는지만 목록으로 준 뒤 필요할 때 도구로 가져온다.
3. **다시 볼 일 없는 중간 기록은 서브에이전트로 보낸다** — 메인 컨텍스트에는 결과만 남긴다.
4. **대화를 보지 못하는 하위 LLM에는 목적을 입력값으로 넘긴다** — 차트 생성 도구의 `instruction`처럼 명시적인 필드를 둔다.
5. **관련 기능은 한 도구로 묶는다** — 생성·수정이나 여러 단계 작업을 하나로 묶어 에이전트가 고를 후보를 줄인다.

원문의 마지막 당부도 4절의 판단 기준과 같다. 각 작업에 어떤 정보가 필요한지, 작업 중 얻은 정보를 이후에도 쓰는지 확인하라는 것이다.

## 6. 참고 자료

- 원문: [카카오 기술 블로그 — 데이터 분석 에이전트를 만들며 배운 컨텍스트 설계](https://tech.kakao.com/posts/838){:target="_blank"} (2026-10-02)
- [Anthropic — Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents){:target="_blank"}
- [Anthropic — Define tools: Best practices for tool definitions](https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools){:target="_blank"}
- [Anthropic — Equipping agents for the real world with Agent Skills](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills){:target="_blank"}
- [John Ousterhout — A Philosophy of Software Design](https://web.stanford.edu/~ouster/cgi-bin/book.php){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/JKUTrJ4vK00){:target="_blank"}
