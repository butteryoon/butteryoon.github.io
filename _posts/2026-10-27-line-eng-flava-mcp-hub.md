---
layout: post
comments: true
title: "MCP 서버 650개를 한 관문으로 — LINE Flava MCP Hub의 4층 배관 설계"
description: "LY Corporation의 전사 MCP 플랫폼 Flava MCP Hub 설계기 정리. DCR을 지원하지 않는 사내 IAM과 MCP 표준 사이의 Protocol Gap을 stdio 프록시(Connector) 한 겹으로 메운 방법, Rust 오픈소스 Gateway에서 자체 Go Gateway로 옮긴 이유(JWKS ES512 파싱 실패·CORS 미들웨어 순서·OOM), 다음 과제인 OpenAPI to MCP와 ID-JAG까지."
img: flava_mcp_hub_title.webp
date: 2026-10-27 18:50:00 +0900
last_modified_at: 2026-10-27 18:50:00 +0900
tags: [line-eng, mcp, mcp-gateway, oauth, oidc, llm-agent, agent-case]
related: agent-case
categories: [line-eng-analysis, llm-agent]
source_url: https://techblog.lycorp.co.jp/ko/flava-mcp-hub-ai-infrastructure
source_date: 2026-10-26
---
MCP 서버 하나 만드는 건 어렵지 않다. 막히는 곳은 그 서버를 사내 인증 뒤에 붙일 때다. 브라우저 콜백을 받고 토큰을 캐시하고 만료되면 갱신하는 코드를 팀마다 따로 짠다. LY Corporation 기술 블로그에 10월 26일 올라온 [「Flava MCP Hub가 걸어온 길: AI 시대를 위한 배관 공사」](https://techblog.lycorp.co.jp/ko/flava-mcp-hub-ai-infrastructure){:target="_blank"}는 이 반복 작업을 플랫폼이 떠맡게 만든 기록이다. 지금은 MCP 서버 650개 이상이 이 플랫폼 위에서 돈다. 사내에 MCP를 퍼뜨리려는 팀이라면 인증을 어디서 처리할지, Gateway를 직접 만들지를 정할 때 참고할 대목이 많다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Flava MCP Hub는 사내 프라이빗 클라우드 Flava 위에 올린 전사 MCP 플랫폼으로, 카탈로그·런타임·Gateway를 한곳에서 제공한다. 요청 경로를 Connection(Connector)·Proxy(Gateway)·Authorization·Monitoring 네 층으로 나눴다. 사내 IAM이 DCR·CIMD를 지원하지 않아 생긴 Protocol Gap은 클라이언트 쪽 stdio 프록시인 Connector가 OIDC·토큰 캐시·갱신·Bearer 주입을 대신 맡는 방식으로 풀었다. 인증 준비 시간은 약 1주일(내부 추정)에서 5분으로 줄었다. 처음 쓰던 Rust 오픈소스 Gateway는 ES512 키가 섞인 JWKS 로딩 실패, CORS보다 앞서 도는 인증 미들웨어, 잦은 OOM 때문에 Go로 새로 만든 자체 Gateway로 바꿨다. 다음 과제는 OpenAPI to MCP, Internal Auth, MCP 최신 스펙 대응, CLI·skill 확장, ID-JAG 다섯 가지다.

## 왜 플랫폼이 필요했나

사내에서 MCP를 쓰는 팀이 늘자 원문이 꼽는 문제 네 가지가 드러났다.

1. **서버가 통제 없이 퍼졌다.** 어떤 MCP 서버가 어디 있는지 아무도 몰라서 이미 있는 걸 다시 만드는 일이 생겼다.
2. **인증이 클라이언트·서버마다 제각각이었다.** 권한 관리가 빠진 MCP는 위험 요인인데, 팀마다 인증을 직접 구현하기엔 비용이 컸다.
3. **사내 인증 체계가 MCP 표준을 따라가지 못했다.** 원문은 이를 'Protocol Gap'이라고 부른다.
4. **사용 현황을 알 수 없었다.** 누가 무엇을 얼마나 쓰는지 데이터가 없으니 무엇부터 지원할지도 정할 수 없었다.

Flava MCP Hub는 이 문제들을 카탈로그(사내 MCP 서버 목록), 런타임(생성·배포·사용), Gateway(권한·접근 제어·인증 흐름) 세 역할로 받는다. 카탈로그에서 서버를 골라 1분 안에 연결하고 새 서버는 인증을 갖춘 채로 5분 안에 배포한다는 것이 원문이 밝힌 숫자다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"현재 플랫폼 위에서 동작하는 MCP 서버는 650개 이상, 사용하는 팀은 200팀 이상이며, 누적 도구 호출 수는 1,150만 건을 넘었습니다."</blockquote></details>

## 4층 배관 모델

요청은 클라이언트에서 출발해 Connector와 Gateway를 지나 MCP 서버에 닿는다. 원문은 이 경로를 네 층으로 나누고 층마다 책임을 따로 뒀다.

| 층 | 하는 일 | 핵심 컴포넌트 |
|---|---|---|
| 1층 Connection | 클라이언트 요청을 처음 받고 사내 인증의 복잡함을 감춘다 | MCP Connector (stdio 연결, OIDC 흐름·토큰 캐시·자동 갱신·Bearer 헤더 주입) |
| 2층 Proxy | 어떤 정책으로 어느 서버에 보낼지 정하고, CORS·헤더 변환·전송 프로토콜을 공통 규약으로 맞춘다 | MCP Gateway |
| 3층 Authorization | 토큰을 먼저 검사하고 사내 여러 인증 방식을 한곳에서 검증한다 | Gateway의 OAuth/OIDC 미들웨어 |
| 4층 Monitoring | 전 구간의 도구 호출·인증 결과·지연 시간을 모은다 | 모니터링 층 |

이렇게 나누면 MCP 서버를 만드는 팀은 프로토콜 처리도 인증 코드도 쓰지 않는다. Gateway가 검증된 요청만 통과시키기 때문이다. 클라이언트 쪽은 설정 한 줄로 Connector에 붙는다.

## Protocol Gap: 표준은 '처음 보는 클라이언트'를 전제한다

MCP 표준은 클라이언트가 처음 만나는 서버에도 바로 연결할 수 있어야 한다는 전제에서 출발한다. 그래서 DCR(dynamic client registration)이나 CIMD(client ID metadata document) 같은 방식을 쓴다. LY의 사내 인증 시스템은 IAM 기반이라 클라이언트가 미리 등록돼 있어야 하고 토큰도 사내 시스템에서 받아야 한다. 표준을 따르면 사내 인증을 못 쓰고 사내 인증에 맞추면 표준에서 벗어난다. 이 틈을 각 팀이 메우려면 결국 같은 인증 로직을 팀마다 다시 짜야 했다.

해법은 끊어진 두 지점 사이에 Connector를 끼우는 것이었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"클라이언트 입장에서 Connector는 표준 stdio MCP 서버처럼 보입니다. 클라이언트는 기존 stdio MCP 표준에 맞춰 연결하기만 하면 됩니다."</blockquote></details>

사내 OIDC, 브라우저 콜백, 토큰 캐시, 자동 갱신, Bearer 헤더 주입은 전부 Connector 안에서 끝난다. 요청은 Connector를 지날 때 이미 인증된 상태가 되고 클라이언트는 사내 인증이 있는지조차 몰라도 된다. 효과는 준비 시간에서 바로 보였다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"내부 추정으로는 이 과정에 대략 1주일 정도가 필요했습니다. 지금은 Connector가 인증 처리를 맡기 때문에 MCP 서버 개발팀은 MCP 서버 로직에만 집중하면 됩니다. 나머지는 Flava MCP Hub가 담당하므로 같은 작업을 5분 안에 마칠 수 있습니다."</blockquote></details>

## 오픈소스 Gateway에서 자체 Go Gateway로

처음에는 개발 기간이 약 2개월로 빠듯해 검증된 오픈소스 Gateway를 들였다. 공통 문제를 다시 풀 필요가 없었으니 초기에는 맞는 선택이었다. 운영이 길어지면서 한계가 셋 보였다. 팀에는 Go에 익숙한 사람이 많았는데 그 오픈소스는 Rust 기반이라 장애가 나면 분석과 디버깅이 어려웠다. 프로젝트 로드맵이 사내 요구를 운영에 필요한 속도로 반영해 주지 않았다. 사내 인증용 변경을 upstream PR로 올리거나 fork에서 따로 관리해야 하는 부담도 있었다.

당시 쓰던 버전과 환경에서 실제로 부딪힌 문제는 이랬다.

- **JWKS 호환성**: 사내 JWKS에 ES512 키가 들어 있었는데, 검증에 그 키를 쓰지 않는데도 로딩 단계에서 파싱이 실패해 Gateway 자체가 뜨지 않았다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"사내 인증 시스템이 제공하는 JWKS(JSON Web Key Set)에는 ES512 알고리즘을 사용하는 키가 포함돼 있었습니다. 당시 사용하던 버전은 이 값을 파싱하지 못했고, 실제 토큰 검증에 해당 키를 사용하지 않더라도 JWKS를 로딩하는 단계에서 실패했습니다."</blockquote></details>
- **미들웨어 실행 순서**: 인증 미들웨어가 CORS보다 먼저 돌아서 인증을 켜면 브라우저 preflight 요청이 CORS 헤더를 받기 전에 거절됐다.
- **메모리 안정성**: MCP 트래픽을 처리하다 Gateway가 자주 OOM에 걸렸고 그 위의 MCP 서버 안정성까지 예측하기 어려워졌다.

Go로 다시 만든 자체 Gateway는 구조를 사내 요구에 맞춰 새로 설계했고 핵심 동작을 직접 제어한다. 팀에 익숙한 언어와 도구로 장애를 분석할 수 있게 됐다. upstream이나 fork에 기대지 않으니 패치 주기도 짧아졌다. 원문은 판단 기준도 함께 남겼다. 오픈소스는 빨리 시작하기에 좋지만 프로젝트 방향과 운영 요구가 계속 벌어지면 fork와 패치로 따라가는 방식이 오히려 부담을 키운다는 것이다. LLM으로 코드를 분석하고 고칠 수 있게 되면서 직접 개발하거나 이관하는 부담이 예전보다 줄었다는 점도 Go 재작성 과정에서 확인했다고 적었다.

## 뜻밖의 수확: 전사 AI 도구 사용량이 보이기 시작했다

처음 기대한 건 중복 제거와 표준화였다. 운영해 보니 다른 가치가 하나 더 있었다. 모든 도구 호출이 Hub를 거치니 어떤 도구가 실제로 많이 쓰이는지, 어느 팀의 MCP 서버가 다른 팀에서도 재사용되는지가 데이터로 쌓였다. 지금은 어떤 기능을 먼저 공식 지원할지, 어떤 MCP 템플릿을 만들지를 이 트래픽 데이터로 정한다. 연결을 쉽게 하려고 만든 구조가 전사 AI 도구 사용을 들여다보는 관측 지점이 된 셈이다.

## 다음 과제

원문이 밝힌 준비 중인 방향은 다섯 가지다.

- **OpenAPI to MCP**: 팀이 가진 REST API의 OpenAPI 스펙에서 경로와 오퍼레이션을 MCP 도구로 바꿔 MCP 엔드포인트로 노출한다. API를 다시 구현하지 않고도 에이전트가 쓸 수 있게 하려는 것이다.
- **Internal Auth**: 팀마다 이미 구현해 둔 API 인증을 그대로 쓸 수 있게 한다.
- **MCP 최신 스펙 대응**: MCP는 변경이 잦다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"2026-07-28 버전에서는 요청마다 프로토콜 버전을 선언하는 방식과 server/discover를 통한 서버 정보 조회가 도입됐습니다."</blockquote></details>
- **CLI와 skill**: 개발자가 실제로 일하는 CLI 환경과 skill 흐름 안에서도 Hub를 쓰게 한다.
- **ID-JAG**(identity assertion JWT authorization grant): 지금은 MCP 서버마다 동의 화면을 거쳐 토큰을 따로 받아야 한다. ID-JAG 토큰을 한 번 받아 서버별 토큰으로 교환하면 서버마다 OAuth 클라이언트를 등록하고 동의받는 과정이 빠진다. 아직 도입 전이고 원문도 사용성이 좋아질 것으로 "기대"한다고만 적었다.

## 내 환경에 옮겨 볼 것

- **사내 IdP가 DCR을 지원하지 않는다면** 원격 MCP 서버마다 OAuth를 붙이기 전에, 클라이언트 쪽에 stdio 프록시를 하나 두고 OIDC 로그인·토큰 캐시·갱신·헤더 주입을 몰아주는 구조부터 검토해 볼 만하다. 클라이언트는 평범한 stdio MCP 서버에 붙는 것과 다를 게 없다.
- **MCP Gateway를 브라우저 클라이언트에 열 계획이라면** CORS 처리가 인증 미들웨어보다 앞에 오는지 확인한다. 순서가 바뀌면 preflight가 막힌다.
- **JWKS를 읽는 컴포넌트는** IdP가 내놓는 키 알고리즘을 전부 파싱하는지 확인한다. 쓰지 않는 키 하나 때문에 로딩 전체가 실패할 수 있다.
- **도구 호출을 한 관문으로 모으면** 사용량 데이터는 덤으로 따라온다. 어떤 MCP 서버를 먼저 다듬을지 정할 근거가 생긴다.
- **오픈소스 Gateway를 쓰고 있다면** 운영팀 주력 언어와 맞는지, 사내 요구가 fork 패치로 계속 쌓이고 있지 않은지가 자체 개발로 넘어갈지 가늠하는 신호다.

## 참고

- 원문: [Flava MCP Hub가 걸어온 길: AI 시대를 위한 배관 공사](https://techblog.lycorp.co.jp/ko/flava-mcp-hub-ai-infrastructure){:target="_blank"} (LY Corporation Tech Blog, 2026-10-26, 박민혁·강인배)
- Flava 배경: [Flava 소개 글](https://techblog.lycorp.co.jp/ko/ly-corporation-next-generation-cloud-platform-flava-introduction){:target="_blank"}
- 같은 블로그 글 정리: [AI가 만든 테스트를 믿는 법 — LINE Plus 회귀 테스트 자동화]({{site.baseurl}}/line-eng-analysis/llm-agent/testing/2026/10/18/line-eng-ai-e2e-test-automation.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/Xlg2KbYFUoM){:target="_blank"}

