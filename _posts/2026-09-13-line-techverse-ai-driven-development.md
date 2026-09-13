---
layout: post
comments: true
title: "LINE Tech-Verse 2026 — AI를 전제로 다시 설계하는 개발 문화"
description: "LINE 엔지니어링 블로그의 Tech-Verse 2026 참관기 해설. 에이전트 하네스 공통화, 검증의 인프라화, 선언형 명세(Moon), 그리고 인당 생산성 9.24배라는 실측 수치까지."
img: line_techverse_title.webp
date: 2026-09-13 22:30:00 +0900
last_modified_at: 2026-09-13 22:30:00 +0900
tags: [line, tech-verse, ai-driven-development, agent-harness, mcp, declarative-api, developer-productivity, llm] # add tag
related: llm
categories: dev
---
LINE 엔지니어링 블로그에 올라온 [「AI를 전제로 다시 설계하다 — Tech-Verse 2026 참관기」](https://techblog.lycorp.co.jp/ko/tech-verse-2026-ai-driven-development-review){:target="_blank"}(2026-09-11, 한규범·황건구)를 정리했다. 컨퍼런스 요약이라기보다 30년 레거시를 짊어진 조직이 AI 주도 개발을 1년간 몸에 붙인 기록이다. 구호 대신 **실측 수치**(9.24배, 39%, 46배)로 말해서 남겨둘 만했다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 전문 대조 후 발행했다.)

<!--more-->

> **TL;DR:** LINE이 Tech-Verse 2026에서 보여준 AI 주도 개발의 뼈대는 세 가지다. (1) 에이전트 하네스의 전사 공통화 — 개인을 누르는 천장이 아니라 평균을 끌어올리는 **바닥**, (2) 검증을 코드 리뷰가 아닌 **인프라**(하드 게이트, 스펙-하네스 계약)로 이동, (3) 하루 300억 요청의 API 게이트웨이를 **선언형 명세(Moon)**로 재설계해 성능·문서·AI 도구 연계를 한 번에. 백엔드 한 팀의 10개월 실측: 인당·시간당 생산성 **9.24배**, 코드량은 오히려 **39% 감소**.

## 1. 공통화는 천장이 아니라 바닥이다

에이전트 하네스(작업 환경·규칙·검증 장치)를 전사 공통화하면 개인의 자유를 묶는 것 아니냐, 흔한 반론이다. 이 글의 답은 정반대다 — 공통화가 없으면 **아예 경험조차 못 하는 사람**이 생기고, 공통화는 상방을 누르는 게 아니라 하방을 끌어올린다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"AI 에이전트의 종착점은 결국 개인화이지만, 공통화가 없으면 아예 경험조차 하지 못하는 사람이 생깁니다. 그래서 전사 관점에서는 상방을 누르기 위해서가 아니라, 하방을 끌어올려 평균을 높이기 위해 공통화가 필요하다 ... '공통화는 천장이 아니라 바닥을 만드는 일'"</blockquote></details>

키노트에서는 AI 에이전트 신규 브랜드 **Agent i**를 공개했다. 에이전트 빌더로는 출시 2개월 만에 워크플로가 100개 넘게 나왔고, **MCP 허브**는 150개 이상의 MCP 서버를 스키마 검증·레이트 제어·권한 추적까지 묶어 플랫폼 레벨에서 관리한다. 런타임 가드레일 5종(프롬프트 인젝션 탐지, 개인정보 마스킹, 유해 콘텐츠·오프토픽 감지, 정보 분류)도 실행 단계에 들어가 있다.

## 2. 검증은 리뷰가 아니라 인프라로 — "2026년은 하네스의 해"

에이전트가 레거시 영역을 멋대로 고친 사건이 전환점이었다고 한다. 그 뒤로는 검증을 사람 리뷰에 맡기지 않고 인프라에 심었다.

| 패턴 | 하는 일 | 시점 |
|------|------|------|
| **하드 게이트** | 위험한 행동을 커밋/푸시 단계에서 아예 차단 | 사전 예방 |
| **스펙-하네스 계약** | 코드 작성 전에 합격 기준(테스트·문서·관례)부터 정의 | 설계 |
| **승인 기반 스킬 진화** | 사람의 승인을 거쳐 에이전트 스킬을 단계적으로 확장 | 런타임 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"특히 \"기준이 '모델이 끝났다고 말했는가'가 아니라 '그렇게 볼 수 있는 근거가 남았는가'로 바뀐다\"는 문장이 오래 남았습니다."</blockquote></details>

## 3. 9.24배 — 10개월의 실측

참관기에서 가장 눈에 띄는 대목은 LINE Plus 백엔드 한 팀이 10개월간 측정한 결과다. CLAUDE.md 10~15줄로 시작하는 컨텍스트 엔지니어링 → 하드 게이트·스펙-하네스 계약 → 승인 기반 스킬 진화로 이어지는 4주 도입 플랜을 거쳐 **인당·시간당 생산성 9.24배**까지 갔다. 뒤집힌 대목이 하나 있다. 코드량은 **39% 줄었는데** 처리 티켓은 늘었다 — 안 써도 될 코드가 빠진 것이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"LINE Plus 백엔드 팀이 10개월간 측정한 결과, 인당·시간당 생산성 9.24배를 기록한 여정을 담은 발표였습니다. 흥미로운 점은 컨텍스트 엔지니어링을 처음 도입했을 때 코드량은 오히려 39% 줄었는데도 처리한 티켓은 늘어났다는 사실입니다."</blockquote></details>

패널 토론에서 나온 **AI Velocity Paradox**도 같은 얘기다 — 각 단계에 AI를 넣어 부분 최적화를 해도 전체 리드타임이 안 줄면, 병목은 과업(노드)이 아니라 **핸드오프(엣지)** 다. 리뷰 대기와 컨텍스트 손실이 진짜 범인이라는 말.

## 4. 선언형 명세의 복리 — API Gateway 'Moon'

Core Technology 트랙에서는 하루 300억 요청을 받는 LINE API Gateway를 **선언형 명세 기반 프레임워크 Moon**으로 다시 만든 이야기가 나왔다. API를 코드가 아니라 명세로 정의하니 서버당 3배 성능과 무재배포 API 추가가 따라왔고, 명세 하나에서 문서 자동 생성 → 실제 트래픽으로 명세 정확성 검증 → AI 개발 도구 연계까지 줄줄이 뻗었다. 700건 넘던 문서-코드 불일치를 풀려던 접근이 AI 시대의 기반이 된 셈이다.

키노트의 또 다른 대목 — 30년 레거시에 105개 이상 서비스를 굴리는 회사의 AX(AI Transformation)는 신규 서비스와 다르고, 기획·설계의 구조화가 먼저다. 1년 만에 **AI용 구조화 설계서 약 46배 증가**, 레거시 내 **AI 생성 코드 비중 약 20%** 도달. 화려할 것 없는 기반 작업에서 나온 수치라 더 믿음이 간다.

## 5. 읽으면서 든 생각

이 블로그의 [오픈소스 에이전트 스택]({{site.baseurl}}/tools/2026/09/11/opensource-agent-stack.html) 실험, 그리고 어제 정리한 [삼성계정 AIOps 사례]({{site.baseurl}}/dev/2026/09/12/samsung-account-agentic-aiops.html)가 가리키는 곳과 거의 겹친다 — **모델보다 하네스(구조·검증·게이트)가 먼저**라는 것. 개인 규모로도 옮겨볼 만한 게 몇 가지 있다.

- CI/CD에 하드 게이트 한 종류(시크릿 스캔, 미승인 외부 호출 차단)부터
- PR·작업 템플릿에 "코드 작성 전 완료 기준 정의"(스펙-하네스 계약) 넣기
- 측정 지표는 스토리 포인트가 아니라 **인당 완료 티켓 수·리뷰 회전 시간·AI 생성 코드 비중**처럼 셀 수 있는 것으로

이 블로그의 발행 파이프라인도 실은 같은 원리로 돌아간다 — 검수 체크리스트가 스펙-하네스 계약, 중복·날짜 검증이 하드 게이트, WRITING-RULES.md 갱신이 승인 기반 스킬 진화에 해당한다.

## 참고

- [원문: AI를 전제로 다시 설계하다 — Tech-Verse 2026 참관기](https://techblog.lycorp.co.jp/ko/tech-verse-2026-ai-driven-development-review){:target="_blank"}
- [Tech-Verse 2026 키노트 영상](https://www.youtube.com/watch?v=EhPJ9ncV10g){:target="_blank"}
- [세션: 2026년은 하네스의 해](https://tech-verse.lycorp.co.jp/2026/ko/sessions/49.html){:target="_blank"} · [세션: AI 도구로 생산성 10배가 가능할까?](https://tech-verse.lycorp.co.jp/2026/ko/sessions/41.html){:target="_blank"} · [세션: LINE API Gateway의 진화](https://tech-verse.lycorp.co.jp/2026/ko/sessions/72.html){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/AMWQIpdsSHY){:target="_blank"}
