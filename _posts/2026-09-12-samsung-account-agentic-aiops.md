---
layout: post
comments: true
title: "삼성계정의 Agentic AIOps — 멀티 에이전트로 장애 원인 분석을 5분 안에"
description: "AWS 기술 블로그의 삼성계정 Agentic AIOps 사례 해설. 오픈소스 MCP 서버의 한계에서 FastMCP 커스텀 서버, Strands Agents as Tools 패턴까지의 진화 과정과 us-east-1 장애를 3분 47초에 분석한 실전 기록."
img: samsung_aiops_title.webp
date: 2026-09-12 21:10:00 +0900
last_modified_at: 2026-09-12 21:10:00 +0900
tags: [aiops, multi-agent, strands-agents, mcp, fastmcp, bedrock, sre, rca, llm] # add tag
related: llm
categories: dev
---
AWS 기술 블로그에 올라온 [「Part2: 삼성계정 서비스의 Agentic AIOps — 운영환경에서 Multi-Agent 시스템으로 RCA 자동화 하기」](https://aws.amazon.com/ko/blogs/tech/part2-agentic-aiops-samsung-account-service){:target="_blank"}(2026-03-27)를 정리했다. 대규모 운영 환경에 멀티 에이전트를 실전 투입한 국내 사례 중 손에 꼽게 구체적인 글이다 — 무엇보다 "처음부터 잘 설계했다"가 아니라 **실패하고 방향을 튼 과정**을 주차별로 공개한 점이 좋다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 전문 대조 후 전면 재작성해 발행했다.)

<!--more-->

> **TL;DR:** 삼성계정 SRE 환경에서 이상 탐지 → 근본 원인 분석(RCA) → 조치 제안까지를 **5분 이내**로 자동화한 사례다. 오픈소스 Datadog MCP 서버로 시작했다가 "숫자만으로는 맥락이 전달되지 않는" 한계에 부딪혀 **FastMCP 커스텀 MCP 서버**를 직접 만들었고, 단일 에이전트의 검증 불가 문제를 겪은 뒤 **Strands Agents SDK의 Agents as Tools 패턴**으로 수집·분석·조치 제안의 책임을 분리했다. 실전에서 AWS us-east-1 장애를 **3분 47초** 만에 "외부 인프라 원인, 신뢰도 high"로 분석해냈다. 실행 권한은 끝까지 사람에게 남겼다.

## 1. 문제 — 데이터는 많은데 분석은 사람 몫

삼성계정은 Datadog, CloudWatch, EKS 로그 등 관측 데이터가 이미 풍부했다. 문제는 장애가 나면 이 데이터들을 **하나의 맥락으로 엮어 RCA와 조치 가이드로 바꾸는 일**이 자동화돼 있지 않았다는 것 — 분석 품질과 속도가 담당자 숙련도에 좌우되고, MTTR/MTTD를 구조적으로 줄이기 어려웠다. 목표는 분명했다. 이상 탐지 후 **5분 이내에 근본 원인 후보와 근거 제시**, 결과는 Slack 워크플로우 안에서 공유.

## 2. 4주간의 진화 — 실패가 설계를 만들었다

이 글의 백미는 아키텍처가 아니라 **여정**이다.

- **1~2주차, 오픈소스 MCP 서버의 한계**: Datadog용 오픈소스 MCP 서버를 붙여봤지만, 대시보드 위젯의 수치를 에이전트가 제대로 해석하지 못했다. 운영자는 "평소보다 에러율이 급증했다"고 판단하는데 API가 주는 건 "15%"라는 숫자뿐 — **데이터 양이 아니라 표현 방식의 문제**였다.
- **3주차, 커스텀 MCP 서버 + 단일 에이전트의 위험**: FastMCP로 커스텀 Datadog MCP 서버를 직접 구축했다. 설계 원칙은 "숫자가 아닌 **상태 변화**를 전달한다" — 기준선 대비 변화율, 이상 여부, 최근 배포 이력 같은 맥락을 구조화해 함께 준다. 그런데 입력 품질이 좋아지니 다른 문제가 튀어나왔다. 수집부터 조치 제안까지 다 맡은 **단일 에이전트는 그럴듯한 결론을 빨리 내지만 검증할 구조가 없다**.
- **4주차, Agents as Tools 패턴**: "더 똑똑한 에이전트"가 아니라 "**분석 책임을 분리하는 구조**"로 방향을 틀었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"이 지점에서 본 프로젝트는 '더 똑똑한 Agent'를 만드는 방향이 아니라 '분석 책임을 분리하는 구조'를 만드는 방향으로 전환했습니다."</blockquote></details>

## 3. 구조 — Orchestrator와 세 명의 전문가

최종 구조는 Strands Agents SDK의 **Agents as Tools** 패턴이다. 전문 에이전트를 `@tool` 데코레이터로 함수처럼 감싸고, Orchestrator가 상황을 보고 동적으로 호출한다.

- **DataCollector**: 커스텀 MCP 도구로 메트릭·EKS 상태·로그 패턴을 수집 (실측 데이터만, 추측 금지)
- **Analyzer**: 수집된 데이터만 근거로 Chain-of-Thought RCA — 외부 도구 없이 순수 추론. 신뢰도 기준까지 명시(독립 증거 3개 이상 = high)
- **SolutionProvider**: 분석 결과와 과거 유사 사례를 참조해 즉시/단기/장기 조치 제안

Graph 패턴(실행 순서가 그래프로 고정)과 달리, 어떤 전문가를 어떤 순서로 몇 번 부를지를 **Orchestrator의 LLM이 런타임에 결정**한다. 배포 직후 장애면 변경 이력부터, 특정 리전 장애면 인프라 상태부터 — 장애 분석처럼 매번 양상이 다른 일에 맞는 선택이다.

## 4. 실전 — us-east-1 장애를 3분 47초에

2025년 10월 19일 AWS us-east-1 리전 대규모 장애(LSE) 때 이 시스템이 실전을 치렀다. 16:05 Datadog 알림이 뜨자 자동 분석이 돌았다.

1. **수집**: 에러율 기준선 대비 +850%(critical), EKS 노드 리소스는 정상, us-east-1 API 지연 확인, **분석 기간 내 내부 배포 이력 없음**
2. **분석**: 시간 축 정렬(16:02 API 지연 → 16:03 에러율 상승 → 16:05 알림), 내부 원인 배제, 영향 서비스가 모두 us-east-1 의존 → **"AWS us-east-1 인프라 이슈, 신뢰도 high(독립 증거 3개)"**
3. **제안**: 즉시(Health Dashboard 확인·failover 준비), 단기(타임아웃 상향·에러 메시지 개선), 장기(멀티 리전 Active-Active)

<details class="evidence"><summary>원문 근거</summary><blockquote>"전체 분석 과정은 3분 47초 만에 완료되었습니다. 운영자가 Slack 알림을 확인하는 시점에 이미 1차 RCA 결과와 조치 가이드가 준비되어 있었습니다."</blockquote></details>

운영자가 알림을 열었을 때 이미 1차 RCA가 도착해 있었다는 것 — 내부 코드 리뷰나 롤백 검토에 쓸 뻔한 시간을 통째로 아낀 셈이다.

## 5. 배울 점

- **환각은 프롬프트가 아니라 구조로 막는다.** "가짜 데이터 생성 금지"를 프롬프트에 적는 것보다, 수집자는 실측만·분석자는 수집된 것만·제안자는 검증된 분석만 보게 **역할을 물리적으로 제한**하는 쪽이 훨씬 잘 먹혔다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"환각을 줄이는 가장 효과적인 방법은 더 많은 규칙을 프롬프트에 추가하는 것이 아니라, 잘못된 행동이 불가능한 구조를 만드는 것입니다."</blockquote></details>

- **범용 MCP 서버는 도메인 판단 단위를 모른다.** 커스텀 MCP의 가치는 API 래핑이 아니라 "운영자가 판단하는 단위(기준선 대비, 추세, 이벤트 연관)"로 데이터를 재구성하는 데 있다.
- **실행 권한은 사람에게.** 시스템은 분석·제안까지만, 서비스 재시작·롤백 같은 권한은 의도적으로 배제했다. "분석을 자동화했지만 책임을 자동화하지는 않았다"는 문장이 이 사례의 요약이다.
- 스택: Bedrock(Claude Sonnet 4.0) + Strands Agents SDK + FastMCP + EKS/CloudWatch + Datadog + Slack + OpenSearch(과거 RCA 검색).

## 6. 읽으면서 든 생각

우리 블로그의 [오픈소스 에이전트 스택]({{site.baseurl}}/tools/2026/09/11/opensource-agent-stack.html) 실험과 규모는 다르지만 결론이 겹친다 — 모델 성능보다 **구조**(책임 분리, 검증 가능한 단계, 사람의 승인 게이트)가 운영 신뢰성을 만든다. 특히 "숫자 대신 상태 변화를 전달하는 커스텀 MCP"는 며칠 전 만든 지오코딩 MCP 서버 같은 소규모 도구에도 그대로 적용할 만한 원칙이다. 시리즈 [Part 1(AI SecOps — 멀티 에이전트 보안 위협 탐지)](https://aws.amazon.com/ko/blogs/tech/part1-samsung-account-ai-secops){:target="_blank"}도 함께 읽을 만하다.

## 참고

- [원문: Part2 — 삼성계정 Agentic AIOps (AWS 기술 블로그)](https://aws.amazon.com/ko/blogs/tech/part2-agentic-aiops-samsung-account-service){:target="_blank"}
- [Part1 — 삼성계정 AI SecOps](https://aws.amazon.com/ko/blogs/tech/part1-samsung-account-ai-secops){:target="_blank"}
- [Strands Agents SDK](https://strandsagents.com/){:target="_blank"} · [FastMCP](https://github.com/jlowin/fastmcp){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/qwtCeJ5cLYs){:target="_blank"}
