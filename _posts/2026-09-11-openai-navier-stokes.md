---
layout: post
comments: true
title: "OpenAI, 밀레니엄 난제 나비에-스토크스 문제를 1만 에이전트로 해결"
description: "OpenAI가 내부 모델 기반 1만 동시 에이전트 시스템으로 나비에-스토크스 존재·매끄러움 문제를 유한 시간 특이점 증명으로 해결하고 Lean으로 검증한 발표를 정리한다."
img: command-title.webp
date: 2026-09-11 19:40:00 +0900
last_modified_at: 2026-09-11 19:40:00 +0900
tags: [openai, navier-stokes, millennium-prize, ai-agents, multi-agent, lean, mathematics, llm] # add tag
related: llm
categories: dev
---
OpenAI가 9월 8일 공개한 [「On the Navier–Stokes Millennium Prize Problem」](https://openai.com/index/navier-stokes-solution/){:target="_blank"}을 정리했다. 90년 묵은 밀레니엄 난제를 AI 멀티에이전트 시스템이 풀었다는, 문장 그대로 믿기 어려운 발표라 원문·논문 PDF·Lean 저장소의 실재부터 확인하고 썼다. [GPT-6 Astra 발표]({{site.baseurl}}/dev/2026/09/09/openai-gpt6-astra.html) 이틀 전에 나온 글이기도 하다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 전문 대조 후 발행했다.)

<!--more-->

> **TL;DR:** OpenAI가 **GPT-6 Astra보다 훨씬 강력한 내부 모델**로 구동되는 **약 1만 개 동시 에이전트** 시스템으로, 나비에-스토크스 존재·매끄러움 문제를 **유한 시간 특이점(blow-up) 증명** 쪽으로 해결하고 **Lean으로 정형 검증**했다. 첫 에이전트 투입부터 해결까지 88시간, Lean 검증에 추가 17시간(이건 GPT-6 Astra가 수행). 전체 시도에 490만 메시지·약 3,000억 출력 토큰이 들었다. OpenAI는 밀레니엄 상금을 청구하지 않겠다고 밝혔다.

## 1. 무엇을 푼 것인가

나비에-스토크스 방정식은 유체를 연속체로 근사해 운동을 기술한다(항공기 설계, 기상 예보, 혈류 해석에 쓰인다). 여태 답이 없던 질문은 이거다. **매끄럽게 시작한 3차원 비압축성 유체가 유한 시간 안에 속도가 무한대로 발산하는 특이점을 만들 수 있는가?** 점성이 운동을 계속 눌러주는데도 말이다.

OpenAI 시스템의 답은 "만들 수 있다"였다. 매끄러운 힘만 가해진, 에너지가 끝까지 유한한 유체가 **안쪽으로 나선을 그리며 스파게티처럼 늘어나는 소용돌이(vortex)** 형태로 특이점에 도달하는 해를 찾아 증명했다. 클레이 수학연구소의 공식 문제 정식화에서 반증 쪽에 해당하는 **statement C(그리고 D)를 성립**시켜 문제를 풀었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Our system produced an analytical proof and a Lean formalization that an initially smooth fluid at rest can develop a singularity in a finite time. ... This resolves the Navier–Stokes Millennium Prize problem by establishing statement "C" (and also "D") in the official Millennium Prize formulation."</blockquote></details>

## 2. 어떻게 풀었나 — 1만 에이전트 오케스트레이션

- **모델**: 8월 28일부터 훈련 중인, **GPT-6 Astra를 크게 능가하는 내부 모델**(대규모 RL로 구축, 훈련은 진행 중). 9월 1일 "밀레니엄 난제 두 개가 풀렸다"는 소문을 듣고 모든 미해결 밀레니엄 문제에 이 모델을 투입하는 프로젝트를 시작했다.
- **구조**: 에이전트를 그룹으로 나눠 그룹마다 문제의 다른 변형(증명 방향 A/B, 반증 방향 C/D)을 맡기고, 그룹 안에서는 서로 이야기할 수 있게 했다. 나비에-스토크스를 푼 그룹은 **동시 실행 약 1만 에이전트** 규모였다.
- **도구**: 인터넷 캐시 읽기, 코드 실행.
- **교차 수분(cross-pollination)**: 일정 시간이 지나면 **Codex로 각 그룹의 쓸 만한 인사이트를 모아** 후속 프롬프트에 넣었다. 해답을 찾은 그룹도 이렇게 길이 열렸다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The groups varied in size, and the group that produced the Navier–Stokes resolution involved on the order of 10,000 concurrent agents. ... After some time, we cross-pollinated the agent groups by using Codex to consolidate the most useful insights from each agent group."</blockquote></details>

중간 단계가 재미있다. 본 문제보다 "쉬운" 문제로 함께 던져둔 **오일러 방정식 정칙성 문제(점성 항을 뺀 극한, 무강제 버전)를 에이전트 약 100개가 50시간 만에 먼저 풀었다**. 이 결과를 보고 자원을 나비에-스토크스로 몰아 **88시간 만에** 해답에 닿았다(9월 5일). **Lean 정형화·검증은 GPT-6 Astra가 17시간에** 마쳤다.

## 3. 물량으로 보는 규모

| 항목 | 수치 |
|------|------|
| 전체 시도 (모든 문제) | 메시지 490만 건, 출력 토큰 약 3,000억 개 |
| 나비에-스토크스만 | 메시지 270만 건, 출력 토큰 약 1,300억 개 |
| 첫 투입 → 해결 | 약 88시간 |
| Lean 검증 | 추가 17시간 (GPT-6 Astra) |

## 4. 동시 연구 — Anthropic 쪽 이야기

발표문의 "Concurrent work" 절이 이례적으로 상세하다. 9월 1일의 그 소문은 Anthropic 직원 Levent Alpöge와 NYU 수학 교수 Tristan Buckmaster의 작업 이야기였다. 두 사람은 이미 Anthropic 내부 모델로 **강제(forced) 오일러 문제**를 풀어둔 상태였다. OpenAI는 이 오일러 결과의 우선권(priority)을 인정한다고 못박았다. 여기에 자기네 시스템이 두 사람의 작업(Codex 프롬프트 포함)에서 아무 영향도 받지 않았다고, 조사까지 해서 확인했다는 업데이트(9월 10일)를 덧붙였다. 두 결과는 다르다. Anthropic 쪽은 강제 오일러, OpenAI 쪽은 무강제 오일러 + 나비에-스토크스 본 문제.

## 5. 읽으면서 든 생각

- 단일 에이전트 루프가 아니라 **수천~1만 에이전트의 병렬 탐색 + Codex 기반 인사이트 통합**, 이 분산 탐색 구조가 고난도 문제를 열었다는 대목이 실무적으로 제일 눈에 들어온다. 에이전트 그룹 간 통신·통합 프로토콜을 설계할 때 두고 볼 기준점이 하나 생긴 셈이다.
- 분석적 증명 → Lean 정형화 → 자동 검증. 이 파이프라인은 LLM 추론의 신뢰 문제를 "절대 검증기"로 받치는 구도다. 다만 환각을 일반적으로 해결한 건 아니다. 정형 검증이 가능한 수학이라는 특수 영역이라 성립하는 이야기다.
- OpenAI가 상금을 청구하지 않겠다고 밝힌 점, "이건 정점이 아니라 진행 중인 스냅숏"이라며 능력 발전의 속도 조절을 언급한 점은 GPT-6 Astra 발표의 안전성 서사와 결이 같다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"We do not intend to claim the Millennium Prize for this result. ... this is not a culmination, but rather a snapshot in time, of progress on AI development."</blockquote></details>

## 참고

- [원문: On the Navier–Stokes Millennium Prize Problem](https://openai.com/index/navier-stokes-solution/){:target="_blank"}
- [증명 논문 PDF](https://cdn.openai.com/pdf/32d9f210-8b73-45e0-91bc-82a30aef8a9a/navier-stokes.pdf){:target="_blank"}
- [Lean 증명 저장소 (openai/NavierStokesAndEuler)](https://github.com/openai/NavierStokesAndEuler){:target="_blank"}
- [관련 글: OpenAI GPT-6 Astra 발표 정리]({{site.baseurl}}/dev/2026/09/09/openai-gpt6-astra.html)
