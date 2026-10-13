---
layout: post
comments: true
title: "Karpathy의 LLM 능력 퍼넬 — 75%는 접점이 없고 0.2%만 수직 상승을 체감한다"
description: "Andrej Karpathy가 4월 트윗을 다시 꺼내며 LLM 능력에 대한 이해 격차가 '두 그룹의 엇갈림'에서 4단계 퍼넬로 벌어졌다고 정리했다. 약 60억 명(75%)은 거의 접점이 없고, 2,000만 명(0.2%)이 프론티어급 에이전트를 실무에 쓰며, 약 5,000명만 내부 시스템에 접근한다."
img: karpathy-llm-capability-funnel_title.webp
date: 2026-10-13 20:20:00 +0900
last_modified_at: 2026-10-13 20:20:00 +0900
tags: [karpathy, llm, llm-agent, jagged-intelligence, rlvr, ai-adoption, agent-research]
related: agent-research
categories: dev
source_url: https://x.com/karpathy/status/2109361546505966046
source_date: 2026-10-12
---

Andrej Karpathy가 한국 시간 10월 12일 새벽에 올린 [긴 트윗](https://x.com/karpathy/status/2109361546505966046){:target="_blank"}을 정리했다. 4월에 썼던 [트윗](https://x.com/karpathy/status/2042334451611693415){:target="_blank"}을 인용하면서 그때 "서로 딴소리를 하는 두 그룹"이라고 불렀던 LLM 능력 인식 격차가 반년 사이 더 벌어져 이제는 날카로운 퍼넬(funnel) 모양이 됐다고 말한다. 인구를 네 층으로 나눈 대략적인 계산과, 그 퍼넬을 만드는 요인 세 가지가 핵심이다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Karpathy는 LLM을 겪어 본 정도에 따라 세계 인구를 네 층으로 나눈다. 약 60억 명(~75%)은 LLM을 거의 접해 보지 못했고, 10~20억 명(~20%)은 무료 챗봇을 가끔 쓴다. 수학·코드 분야에서 프론티어급 에이전트를 실무에 쓰는 사람은 약 2,000만 명(0.2%), 내부 프론티어 시스템에 접근하는 사람은 약 5,000명(~0.00006%)이다. 이 퍼넬을 만드는 것은 ① 문제의 야심·규모·지평, ② 검증 가능한 도메인에 능력이 몰리는 '들쭉날쭉함(jaggedness)', ③ 접근 등급이다.

## 4월 트윗: 두 그룹이 서로 딴소리를 한다

4월 9일(UTC) 트윗에서 Karpathy는 AI 능력 인식이 두 그룹으로 갈린다고 봤다. 한쪽은 작년쯤 무료 ChatGPT를 써 보고 그 인상으로 AI를 판단하는 사람들이다. 이들은 환각이나 "세차장에 차를 몰고 갈까, 걸어갈까" 같은 질문을 OpenAI Advanced Voice Mode가 엉뚱하게 받아넘기는 영상을 보며 웃는다. 다른 쪽은 OpenAI Codex나 Claude Code 같은 최신 에이전트 모델에 돈을 내고 프로그래밍·수학·연구에 실무로 쓰는 사람들이다. Karpathy는 이 그룹이 'AI Psychosis'에 가장 심하게 노출돼 있다고 표현했다. 며칠~몇 주 걸릴 프로그래밍 문제를 모델이 녹여 버리는 장면을 직접 보기 때문이다.

같은 회사의 무료 음성 모드는 가장 단순한 질문에서 헛발질하는데, 최상위 유료 Codex 모델은 한 시간 동안 혼자 코드베이스 전체를 일관되게 재구성하거나 시스템 취약점을 찾아 공격한다. 4월 트윗은 이 두 장면이 동시에 참이라는 데서 끝난다.

## 10월 트윗: 네 층짜리 퍼넬

반년 뒤 Karpathy는 격차가 *넓어지고 있다*며 이제는 두 그룹의 엇갈림보다 퍼넬에 가깝다고 했다. 본인이 "냅킨 계산(napkin math)"이라고 밝힌 추정치다.

1. **약 60억 명(~75%)**: LLM을 거의 접해 보지 못했다.
   <details class="evidence"><summary>원문 근거</summary><blockquote>"Napkin math somewhere around 6B people (~75% of the population) have barely come in contact with LLMs at all."</blockquote></details>
2. **10~20억 명(~20%)**: 무료 ChatGPT류 제품을 가끔 쓴다. 이들 눈에 LLM은 좀 어설픈 챗봇이고 조금 나은 구글 검색이나 글쓰기 도구 정도다. 수학·코드 밖의 전문직도 상당수가 여기 속한다. 예컨대 경영진은 일과 대부분을 사람과 이야기하며 보내기 때문에 지금 벌어지는 일을 머리로는 이해해도 직접 겪지는 못한다.
3. **약 2,000만 명(0.2%)**: 수학·코드 분야에서 프론티어급 LLM을 실무에 쓴다. 몇 주~몇 달 걸리던 크고 복잡한 프로젝트를 에이전트가 프롬프트 하나로 끝내는 장면을 직접 본다. 앱을 만들고 베끼고 다른 언어로 옮기고 바이너리에서 디컴파일한다. Karpathy 자신도 1년이 채 안 된 시점까지는 코드를 손으로 한 글자씩 치면서 가끔 Tab으로 자동완성을 받는 정도였다고 한다.
   <details class="evidence"><summary>원문 근거</summary><blockquote>"Somewhere around 20M people (0.2%) see first-hand that large, complex projects that used to take them weeks/months can now be completed by agents with a prompt."</blockquote></details>
4. **약 5,000명(~0.00006%)**: 프론티어급 시스템을 내부에서 쓴다. 바깥세상은 그 예고편만 봤다. 수천 개 에이전트가 몇 주씩 협업해 거대한 소프트웨어 프로젝트를 굴리고, 제로데이를 찾아내고, 기계 속도로 사이버 공격과 방어를 주고받고, 새로운 과학을 발견하고, 수학의 최전선을 넓힌다. 업계 최고 전문가가 몇 년 걸려 할 일이다.
   <details class="evidence"><summary>원문 근거</summary><blockquote>"And finally we get to the ~5,000 people (~0.00006%) with access to frontier-grade systems internally. The external world has seen the preview."</blockquote></details>

마지막 층에서는 사람의 검토와 이해가 결과물을 따라가지 못하기 시작했다고 Karpathy는 덧붙인다. 몇 달 전 일어난 OpenAI-HF 사건을 사람들이 아직도 '고고학'처럼 파헤치고 있고 수학자들은 프론티어 수학 원고 722편을 한동안 붙들고 있어야 할 것이라는 예를 든다.

## 퍼넬을 만드는 세 가지 요인

1. **야심·문제 크기·지평**: 영향력은 얼마나 큰 문제를 얼마나 긴 호흡으로 맡기느냐에 비례한다. 한 문단으로 답이 끝나는 질문은 시스템에 거의 부담을 주지 않는다. 정말 신경 쓰는 크고 어려운 문제를 쌓아 두고 있어야 한다. 이 요인이 퍼넬의 '일반 소비자 ↔ 전문가' 축을 만든다.
2. **시스템의 들쭉날쭉함(jaggedness)**: Karpathy가 따로 여러 번 다뤘다고 밝힌 주제다. 능력은 디지털이고 검증 가능하며 경제적 가치가 높은 도메인에서 가장 높이 솟는다. LLM 능력이 수익 가능성을 기준으로 고른 환경 묶음 위에서 검증 가능한 보상으로 강화학습(RLVR)을 돌려 얻어지기 때문이다. 이 요인이 '수학·코드 ↔ 그 밖의 분야' 축을 만든다.
   <details class="evidence"><summary>원문 근거</summary><blockquote>"Capability peaks in domains that are digital, verifiable and economically valuable. This is because LLM capability emerges from reinforcement learning on verifiable rewards on a curated environment mixture driven by revenue potential."</blockquote></details>
3. **접근 등급**: 무료, 유료, 내부.

두 번째 요인은 4월 트윗에서 더 구체적으로 나온다. 코드 분야에는 "단위 테스트를 통과했는가, 아닌가"처럼 명시적으로 검증할 수 있는 보상 함수가 있어 강화학습에 잘 맞는다. 글쓰기는 그렇게 판정하기가 훨씬 어렵다. 게다가 B2B 시장에서 가치가 크니 팀의 가장 큰 몫이 그쪽 개선에 투입된다. 검증기 품질이 RL 성능을 좌우한다는 사례는 [Salesforce RIVER 논문 분석]({{site.baseurl}}/dev/2026/09/24/river-verifier-quality.html)에서 다룬 적이 있다.

## 지금 이 순간의 이상함

Karpathy는 이 상황을 "지금 이 순간의 이상함"이라고 부른다. 일반 대중 대부분은 이런 시스템을 써 본 적이 없고 써 봤다 해도 어설픈 챗봇으로 보였을 것이다. 전문가 대다수는 아직 소소한 개선만 느낀다. 그리고 아주 얇은 층의 전문가들은 곡선이 수직으로 꺾여 올라가는 현기증을 겪고 있다. 이 모든 일이 동시에 벌어진다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"And a small sliver of professionals are experiencing the vertigo of the curve going vertical. And it is all happening at the same time."</blockquote></details>

같은 기술을 놓고 누가 무엇을 말하는지 읽으려면 화자가 이 퍼넬의 어느 층에 서 있는지부터 봐야 한다는 이야기로 읽힌다. "LLM은 아직 쓸 데가 별로 없다"는 말과 "몇 주짜리 일을 프롬프트 하나로 끝냈다"는 말이 둘 다 당사자 경험으로는 정확할 수 있다. 다만 숫자는 Karpathy 스스로 냅킨 계산이라고 밝힌 대략적 추정이다. 정밀한 조사 결과로 인용하기보다는 규모의 감을 잡는 데 쓰는 편이 맞다.

## 참고

- [Andrej Karpathy, 2026-10-11(UTC) 트윗 — LLM 능력 퍼넬](https://x.com/karpathy/status/2109361546505966046){:target="_blank"}
- [Andrej Karpathy, 2026-04-09(UTC) 트윗 — 두 그룹의 엇갈림](https://x.com/karpathy/status/2042334451611693415){:target="_blank"}
- [LLM 출력을 어떻게 읽을까 — Karpathy의 형식 사다리 네 단계]({{site.baseurl}}/dev/2026/10/03/llm-output-formats-karpathy.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/JB3mBjGt94Y){:target="_blank"}
