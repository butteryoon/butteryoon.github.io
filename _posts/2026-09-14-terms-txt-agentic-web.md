---
layout: post
comments: true
title: "terms.txt — robots.txt로는 못 막는 AI 에이전트에게 값을 매기다"
description: "arXiv 논문 terms.txt 해설. AI 크롤러가 방문자 하나당 수천 페이지를 가져가며 깨진 웹의 암묵적 거래를, 경로·목적별 접근 조건과 Web Bot Auth 서명·HTTP 402 협상·서명 영수증으로 다시 쓰자는 제안."
img: terms_txt_title.webp
date: 2026-09-14 22:00:00 +0900
last_modified_at: 2026-09-14 22:00:00 +0900
tags: [terms-txt, robots-txt, ai-crawler, web-protocol, http-402, web-bot-auth, agents, llm] # add tag
related: llm
categories: dev
---
DAIR.AI가 소개한 논문 [「terms.txt: A Consent and Compensation Protocol for Agentic Web Access」](https://arxiv.org/abs/2609.11152){:target="_blank"}(Rajarshi Chowdhury, 2026-09-10)를 읽고 정리했다. `robots.txt` 하나로 버텨온 웹의 접근 규약을 AI 에이전트 시대에 맞게 다시 쓰자는 제안인데, 무엇보다 **왜 지금 필요한가**를 보여주는 수치가 인상적이다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 대조 후 발행했다.)

<!--more-->

> **TL;DR:** 웹은 "사이트는 크롤러를 들이고, 검색엔진은 방문자를 돌려보낸다"는 **암묵적 거래**로 돌아갔는데 AI 크롤러가 그 거래를 깼다. `robots.txt`는 신원·목적·조건·가격을 표현하지 못하고 우회도 쉽다. 논문이 제안하는 **terms.txt**는 경로별·목적별로 기계 접근 조건을 적는 파일이고, 여기에 **Web Bot Auth 서명 + 서명된 의도 + 위임 토큰 + HTTP 402 협상 + 서명 영수증**을 붙여 오리진이 직접 강제한다.

## 1. 깨진 거래 — 방문자 한 명에 수천 페이지

논문이 근거로 드는 크롤 대 방문자 비율을 보면 문제의 크기가 바로 잡힌다. 구글 검색이 방문자 한 명당 약 5페이지를 가져가는데, Perplexity는 190~194페이지, OpenAI는 수백 페이지다. Anthropic은 2025년 6월 약 70,900페이지까지 갔다가 2026년 7월에는 2,000 이하로 내려왔다. 숫자가 오르내리는 것과 별개로, 트래픽을 돌려주는 대가로 콘텐츠를 내주던 균형이 더는 성립하지 않는다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The open web ran on an unwritten bargain: sites admitted crawlers, and search engines sent visitors back. Public measurements show that bargain breaking under AI crawlers and agents. Automated clients now make up most requests, training dominates Cloudflare-classified crawling, and the largest AI platforms fetch thousands of pages for each visitor they return."</blockquote></details>

## 2. robots.txt가 못 하는 네 가지

논문은 기존 규약의 한계를 네 가지로 못 박는다 — **신원(identity)·목적(purpose)·조건(terms)·가격(price)** 을 표현할 수 없다는 것. 게다가 지킬지 말지는 크롤러 마음이라 우회가 쉽고, 그렇다고 대안이 없느냐 하면 대부분 **특정 CDN의 독점 기능**이라 표준이 되기 어렵다. "크롤을 허용할 것인가 말 것인가"라는 이분법으로는 "학습용이면 유료, 검색 색인이면 무료" 같은 현실의 요구를 담을 수 없다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The web's common control, robots.txt, cannot express identity, purpose, terms, or price, can be circumvented, and newer alternatives are largely proprietary CDN features."</blockquote></details>

## 3. terms.txt — 경로별·목적별로 값을 매기는 파일

형식은 `robots.txt`를 닮았지만 표현력이 다르다. 논문 5절이 제시하는 필드를 훑어보면 성격이 드러난다.

- `Terms-Id` — 조건 문서의 버전
- `Receipt-Keys` — 영수증 서명 검증에 쓸 키 디렉토리
- `Payment` — 결제 수단
- `Path` — 경로별 규칙
- `Purpose` — **목적별 규칙**: `search`, `agent`, `train-ai`, `archive`, `research`
- `Unsigned` — 서명 없이 들어온 요청을 어떻게 다룰지

각 목적에는 `allow` / `charge` / `deny`와 사용 범위·위임 조건·가격을 붙인다. 즉 "검색 색인은 무료 허용, 학습용 수집은 페이지당 과금, 아카이브는 거부" 같은 정책을 파일 하나로 선언할 수 있다.

## 4. 선언에 그치지 않게 — 오리진이 강제하는 교환

텍스트 파일만으로는 예의에 기대는 신사협정이다. 논문의 핵심은 여기에 암호학적 교환을 붙인 부분이다.

| 요소 | 역할 |
|------|------|
| **Web Bot Auth 서명** | 에이전트가 자기 신원을 서명으로 증명 |
| **서명된 의도(signed intent)** | "무엇을 위해 가져가는가"를 요청에 명시 |
| **위임 토큰(delegation token)** | 사람이나 상위 주체가 위임한 권한 범위를 증명 |
| **HTTP 402 협상** | `402 Payment Required`로 유료 접근을 협상 |
| **서명 영수증(signed receipt)** | 무엇을 어떤 조건으로 가져갔는지 사후 감사 가능하게 기록 |

HTTP 402는 표준에 오래 예약만 돼 있던 상태 코드인데, 에이전트가 스스로 결제하는 시대에 와서야 쓸모를 찾은 셈이다.

## 5. 강제·감사·계약의 경계

현실적인 대목은 논문이 **무엇을 기술로 강제할 수 있고 무엇은 못 하는지** 를 나눠둔 점이다. 전달 이전 단계는 오리진이 강제할 수 있다(서명·토큰·결제 확인). 전달 이후 선언한 목적을 실제로 지켰는지는 행동과 대조하는 **감사**의 영역이다. 그리고 적법하게 받아간 콘텐츠로 모델이 무엇을 하는지는 결국 **계약**의 문제로 남는다. 기술이 다 해결한다고 말하지 않아서 오히려 설득력이 있다.

## 6. 읽으면서 든 생각

- 이 제안의 진짜 관문은 문법이 아니라 **채택**이다. 사이트가 파일을 놓아도 크롤러가 서명을 안 붙이면 그만이고, 결국 오리진이 서명 없는 요청을 정말 막을 의지가 있느냐에 달렸다. `Unsigned` 필드가 존재하는 이유이기도 하다.
- 반대로 에이전트를 만드는 쪽에서는 **지불 의사가 곧 접근권**이 되는 세계가 열린다. 며칠 전 정리한 [OpenAI GPT-Live-1]({{site.baseurl}}/dev/2026/09/12/openai-gpt-live-1-api.html)이나 자율 에이전트들이 웹을 도구로 쓰는 흐름을 보면, 에이전트가 스스로 402를 받고 결제해 데이터를 사는 그림이 그리 멀어 보이지 않는다.
- 개인 블로그 입장에서도 남의 일이 아니다. 이 블로그 역시 `robots.txt`에 AI 크롤러를 명시적으로 허용해 두고 있는데, 그건 "노출을 얻는 대가"라는 옛 거래를 전제한 선택이다. 거래의 전제가 바뀌면 선택도 다시 검토할 일이다.

## 참고

- [논문: terms.txt — A Consent and Compensation Protocol for Agentic Web Access (arXiv:2609.11152)](https://arxiv.org/abs/2609.11152){:target="_blank"}
- [DAIR.AI Academy 논문 페이지](https://academy.dair.ai/papers/terms-txt-a-consent-and-compensation-protocol-for-agentic-web-access-2609.11152){:target="_blank"}
- [Model Context Protocol](https://modelcontextprotocol.io){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/2lFZtgB85JM){:target="_blank"}
