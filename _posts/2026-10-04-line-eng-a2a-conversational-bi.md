---
layout: post
comments: true
title: "LINE Engineering 분석: A2A 기반 대화형 BI — 의미·안전·구조 세 축으로 운영 신뢰 확보"
description: "LY Corporation Game Platform실이 만든 대화형 BI 'ChatGamesight'의 A2A 프로토콜 기반 에이전트 분리, 온톨로지 RAG, Query Guardrail 3층 설계와 운영 성과"
img: line_eng_a2a_bi_title.webp
date: 2026-10-04 18:00:00 +0900
last_modified_at: 2026-10-04 18:00:00 +0900
tags: [line-eng, a2a, conversational-bi, ontology-rag, query-guardrail, llm-agent, llm-rag, llm-serving]
related: llm-agent
categories: [line-eng-analysis, llm-agent, llm-rag, llm-serving]
source_url: https://techblog.lycorp.co.jp/ko/a2a-conversational-bi-app
source_date: 2026-10-02
---

LY Corporation Game Platform실은 자연어 질문을 SQL로 바꿔 실행하고 시각화까지 이어 주는 대화형 BI 애플리케이션 'ChatGamesight'를 운영한다. 이 글은 LY Corporation 기술 블로그의 개발기를 읽고, 모델이 아니라 설계로 신뢰를 만든 방식을 정리한 것이다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Game Platform실은 **의미(Meaning)·안전(Safety)·구조(Structure)** 세 축으로 대화형 BI의 운영 신뢰를 확보했다. A2A 프로토콜로 에이전트 역할을 Domain/Retrieval/SQL/Guardrail/Report로 나누고, 온톨로지 기반 RAG로 지표 정의와 관계를 검색 가능한 지식으로 정규화했으며, 실행 전에 SQL을 RLS/CLS로 검증하는 Query Guardrail을 독립 서비스로 만들었다. 그 결과 반복 문의와 수작업 리포팅 부담이 줄었고, 분석 결과와 함께 쿼리를 보여 줘 사용자 신뢰도 높아졌다.

## 1. 원문 정보

- **원문 링크:** [LLM이 만든 SQL을 믿고 실행하기까지: A2A 기반의 대화형 BI 애플리케이션 개발기](https://techblog.lycorp.co.jp/ko/a2a-conversational-bi-app){:target="_blank"} (LY Corporation Tech Blog, 2026-10-02)
- **작성자:** 이형중, 김민희, 정소영 (Game Platform실)
- **핵심 키워드:** A2A(Agent-to-Agent), 온톨로지 RAG, Query Guardrail, 대화형 BI, 운영 신뢰

대화형 BI를 '모델이 SQL을 잘 만들게 하는 문제'로 보지 않고, **에이전트 역할 분리(A2A)·의미 지식 정규화(온톨로지 RAG)·실행 전 안전 검증(Query Guardrail)** 세 층으로 설계한 사례다.

## 2. 기술적 내용 분석

### 2.1 문제 배경: '그럴듯한 오답'이 신뢰를 무너뜨린다

원문은 두 가지 과제를 짚는다.

<details class="evidence"><summary>원문 근거 — 문제 1: LLM은 사내 데이터의 의미를 모른다</summary><blockquote>"스키마(DDL)와 컬럼 설명만으로는, 비즈니스에서 쓰는 지표의 의미나 데이터 간 관계를 온전히 전달하기 어렵습니다"</blockquote></details>

<details class="evidence"><summary>원문 근거 — 그럴듯한 오답</summary><blockquote>"운영에서 신뢰를 무너뜨리는 건 보통 이런 ‘그럴듯한 오답’입니다."</blockquote></details>

<details class="evidence"><summary>원문 근거 — 문제 2: 생성된 SQL을 그대로 실행할 수 없다</summary><blockquote>"이런 리스크는 프롬프트만으로 안정적으로 통제되기 어렵습니다."</blockquote></details>

- **의미 문제:** 지표 정의(환불 반영 여부, 기준 시점, 집계 단위)와 테이블 JOIN의 암묵적 규칙, 대시보드의 필터·계산식이 사실상 표준 정의 노릇을 한다.
- **안전 문제:** 허용되지 않은 리소스 조회(권한), `SELECT *` 같은 개인정보 노출, 넓은 기간이나 큰 테이블 JOIN으로 인한 과도한 리소스 사용이 위험 요인이다.

### 2.2 A2A 프로토콜 기반 에이전트 역할 분리

분리 기준은 실패 유형의 전문성 경계다.

| 에이전트 | 책임 | 실패 유형 |
|----------|------|-----------|
| **Domain Agent** | 지표·세그먼트 정의, 해석 규칙, 기본 조건 제안 | 정의 및 의미 해석 실패 |
| **Retrieval Agent** | 온톨로지·메타데이터 인덱스에서 근거 구성, 근거 부족 시 명시적 반환 | 근거 부족 |
| **SQL Agent** | 근거를 반영한 실행 가능 SQL 후보 생성, 중복·필터·기간 규칙 일관 적용 | SQL 실패 |
| **Guardrail/Policy Agent** | 실행 전 정책(RLS, CLS, 리소스) 강제, 위반 시 수정 힌트를 구조화해 반환 | 정책 실패 |
| **Report Agent** | 결과를 재사용 가능한 형태(요약·리포트)로 정리, 커뮤니케이션 비용 절감 | 커뮤니케이션 실패 |

A2A 계약으로 표준화한 요소는 다음과 같다.

- 입출력 스키마: 질문 요약, 근거 묶음, SQL 후보, 검증 결과, 사용자 메시지
- 실패 표현: 근거 부족, 정의 충돌, 권한 위반, 리소스 위험을 서로 다른 유형으로 구분
- 근거 전달: 최종 답변뿐 아니라 해석 근거도 함께 전달
- 개입 지점: SQL 실행 전 승인처럼 사람이 검토하고 수정할 수 있는 지점

<details class="evidence"><summary>원문 근거 — A2A 역할 분리의 의도</summary><blockquote>"각 에이전트가 어떤 입력을 받고 어떤 출력에 책임지는지를 계약처럼 명확히 만드는 것입니다"</blockquote></details>

### 2.3 온톨로지 기반 RAG: 정의·관계·규칙을 검색 가능한 형태로

RAG는 두 역할을 동시에 맡는다.

1. **쿼리 구체화:** 질문에 없는 조건(이벤트 기간, 대상 게임·서비스, 평가 KPI)을 메타데이터로 자동으로 채운다.
2. **지식 검색:** 수치 해석에 필요한 용어 정의, 프로모션·이벤트 문서, 대시보드 설명, 필요하면 외부 시장 자료까지 함께 제공한다.

온톨로지는 Tableau, OpenMetadata 등 여러 소스에서 데이터를 모아 공통 스키마로 정규화해 중앙화하는 방식으로 만들었다. 단순 문서 검색이 아니라 지표·대시보드·이벤트·게임 사이의 의미와 관계를 구조적으로 정의하는 데 초점을 뒀다. 저장소는 Vector DB로 시작했고, Graph DB는 초기 투자 비용과 검증 리스크 때문에 뒤로 미뤘다.

<details class="evidence"><summary>원문 근거 — Vector DB부터 시작한 이유</summary><blockquote>"GraphDB는 스키마 설계와 관계 정의에 필요한 초기 투자 비용이 크고, 어떤 관계가 실제 질의 품질에 영향을 미치는지 충분히 검증되지 않은 시점에서 그래프 모델을 먼저 확정하는 것은 리스크가 있었습니다."</blockquote></details>

Retrieval은 세 단계로 동작한다.

1. **Ontology Routing:** Intent Parser가 의도와 핵심 용어를 뽑으면, Ontology Router가 Entity-Relation Mapping을 참조해 Retrieval Plan을 세운다.
2. **Dual-track Retrieval:** OpenSearch 하이브리드 검색(키워드+의미)으로 관련 문서를 모으고, 질의가 지표 계산을 요구하면 집계 조건(기간, 게임, 지표 정의)을 구체화해 SQL 생성에 넘긴다.
3. **Grounded Answer:** 검색한 맥락과 집계 결과를 합쳐 답하고, 답변에 쓴 데이터 소스도 함께 보여 준다.

### 2.4 Query Guardrail: 실행 전 SQL 검증을 독립 서비스로

Guardrail은 REST와 MCP 도구 두 형태로 제공한다. REST는 애플리케이션 서버가 실행 직전에 호출하고, MCP 도구는 에이전트가 자기 수정 루프에서 호출한다. 같은 검증 로직을 시스템 경로(강제)와 에이전트 경로(자기 수정)가 함께 쓰는 셈이다.

- **RLS(Row-Level Security):** 사용자에게 허용된 리소스만 조회하는지 본다. SQL을 파싱해 '접근 리소스 목록을 반환하는 쿼리'로 재작성하고, 실행 결과를 사용자 권한과 대조한다.
- **CLS(Column-Level Security):** `SELECT *` 확장, 별칭, 서브쿼리까지 고려해 최종 노출 컬럼 집합을 계산하고, 민감 컬럼이 포함됐는지 판단한다.

차단 메시지도 단순한 '권한 없음'에서 멈추지 않는다. 위반한 정책(RLS/CLS/리소스), 문제가 된 리소스·컬럼 요약, 수정 방향 힌트(필수 필터 키, 금지 컬럼 계열)를 함께 돌려준다.

<details class="evidence"><summary>원문 근거 — Guardrail 설계 철학</summary><blockquote>"Guardrail이 단순히 ‘권한 없음’만 반환하면, 사용자는 무엇을 고쳐야 할지 알기 어렵습니다. 에이전트도 마찬가지입니다."</blockquote></details>

### 2.5 운영 성과

반복 문의와 수작업 리포팅 부담이 줄었다. 분석 결과에 쿼리를 숨기지 않고 함께 보여 줘서, 사용자가 직접 확인하고 고칠 수 있게 한 점이 신뢰를 높였다고 한다. 원문이 꼽은 핵심 깨달음은 이 한 문장이다.

<details class="evidence"><summary>원문 근거 — 핵심 깨달음</summary><blockquote>"대화형 BI의 신뢰는 모델이 아니라 설계에서 온다"</blockquote></details>

## 3. 심화 분석

### 3.1 에이전트 오케스트레이션의 실전 패턴

단일 거대 프롬프트나 단일 에이전트에 의미·안전·개선 단위를 몰아 넣으면 운영에서 무너지기 쉽다. 이 사례는 A2A로 에이전트 간 계약을 명시해, 도메인 지식·검색·SQL·정책·리포트를 각각 독립적으로 개발·배포·개선할 수 있게 쪼갰다는 점이 핵심이다.

개발·배포 흐름도 눈여겨볼 만하다. Langflow와 AI Admin으로 플로우(프롬프트·도구·출력 스키마)를 시각적으로 구성하고, 샘플 질문 세트로 회귀를 확인한 뒤, AI Admin의 flow ID로 버전을 관리해 런타임에서 호출한다. 코드 배포와 플로우 배포의 리듬이 분리되니, 지표 정의나 예외처럼 자주 바뀌는 도메인 규칙을 작은 단위로 자주 반영할 수 있다.

### 3.2 온톨로지 RAG의 단계적 접근: Vector DB에서 Graph DB로

관계를 명시적으로 정의하지 않은 Vector DB 방식에서는 2~3홉 떨어진 간접 관계를 LLM 추론에 맡겨야 했고, 답변 정확도가 불안정했다. 원문은 이 경험이 온톨로지에서 어떤 관계를 먼저 정의해야 하는지 알려 주는 기준이 됐고, 이후 Graph DB를 도입할 때 스키마 설계의 토대가 될 예정이라고 쓴다. 실제 질의 품질로 관계의 가치를 확인한 뒤 그래프로 넘어가는 순서는 다른 환경에도 옮겨 볼 만하다.

### 3.3 Guardrail을 차단기가 아닌 피드백 루프로

에이전트가 MCP 도구로 검증 결과를 읽고 구조화된 수정 힌트(필수 필터, 금지 컬럼)를 따라 스스로 쿼리를 고친다. 한편 REST 경로에서는 같은 로직이 실행 전에 강제된다. 에이전트의 자율성과 시스템의 강제력을 한 검증 로직으로 양립시킨 설계로, 에이전트 기반 애플리케이션의 안전장치를 고민할 때 참고할 만하다.

### 3.4 우리 프로젝트와의 관련성

- **에이전트 오케스트레이션:** Hermes Agent의 스킬 기반 워크플로와 A2A의 '역할별 계약' 개념이 맞닿아 있다. 스킬 단위로 책임을 나누고 입출력 스키마를 명시하는 방향이 유효해 보인다.
- **RAG 구축:** 온톨로지 정규화 → Vector DB 임베딩 → Graph DB 확장이라는 순서는 문서 청킹·임베딩·리랭킹 파이프라인과 결합해 볼 수 있다.
- **안전장치:** RLS/CLS를 위한 SQL 파싱 접근은 로컬 LLM 서빙 환경에서 SQL 실행 전 검증 레이어로 검토할 만하다.

## 4. 참고 자료

- 원문: [LY Corporation Tech Blog — LLM이 만든 SQL을 믿고 실행하기까지](https://techblog.lycorp.co.jp/ko/a2a-conversational-bi-app){:target="_blank"} (2026-10-02)
- 관련 글(원문 하단 Related Post):
  - [LLM에게 어디까지 맡길 것인가: AI 에이전트 기반 광고 분석 리포트 자동화](https://techblog.lycorp.co.jp/ko/ai-agent-ad-report-automation){:target="_blank"}
  - [AI를 전제로 다시 설계하다, Tech-Verse 2026 참관기](https://techblog.lycorp.co.jp/ko/tech-verse-2026-ai-driven-development-review){:target="_blank"}
  - [장애 Alert의 원인을 스스로 찾다: SRE Observer 개발기](https://techblog.lycorp.co.jp/ko/building-sre-observer-for-alert-root-cause-analysis){:target="_blank"}
- A2A 프로토콜: [A2A 오픈 프로토콜 저장소](https://github.com/google/A2A){:target="_blank"}
- MCP(Model Context Protocol): [MCP 사양](https://modelcontextprotocol.io/){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/qwtCeJ5cLYs){:target="_blank"}
