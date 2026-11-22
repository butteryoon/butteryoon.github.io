---
layout: post
comments: true
title: "Gemini 4 Neon 출시 — 가격·접근 경로·사고 수준과 하네스별 수치"
description: "구글 Gemini 4 Neon 발표를 에이전트 운영자 관점에서 정리했다. 입력 $3 / 캐시 입력 $0.15 / 출력 $15, 1M 토큰 컨텍스트, 네 단계 사고 수준과 기본값 high, 접근 경로, 같은 모델이 하네스에 따라 12.8%와 39.0%로 갈린 SWE-sweep 수치, 보안용 모델의 제한 접근까지."
img: gemini-4-neon_title.webp
date: 2026-11-22 19:20:00 +0900
last_modified_at: 2026-11-22 19:20:00 +0900
tags: [deepmind, gemini, gemini-4-neon, antigravity, pricing, llm-model, llm-agent, llm]
related: llm-model
categories: dev
source_url: https://deepmind.google/blog/introducing-gemini-4-neon/
source_date: 2026-11-18
---
구글이 11월 18일 [Gemini 4 Neon](https://deepmind.google/blog/introducing-gemini-4-neon/){:target="_blank"}을 발표했다. Gemini 4 계열에서 Carbon과 Hydrogen 사이에 놓이는 모델이고 발표 당일부터 Gemini API 유료 등급과 Google Antigravity 등에 순차 제공되기 시작했다. 발표문은 대부분 벤치마크 순위로 채워져 있는데, 에이전트를 직접 돌리는 쪽에서 오늘 확인할 것은 가격, 접근 경로, 사고 수준 설정, 그리고 하네스에 따라 점수가 얼마나 달라지는지다. 앞서 정리한 [Gemini 4 Argon]({{site.baseurl}}/dev/2026/10/20/gemini-4-argon-agent.html), [Gemini 4 Hydrogen과 llm-gemini]({{site.baseurl}}/tools/2026/11/13/llm-gemini-037-gemini-4-hydrogen.html)에 이어 원문 기준으로 적는다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Gemini 4 Neon의 가격은 1M 토큰당 입력 $3, 캐시 입력 $0.15(95% 할인), 출력 $15이고 컨텍스트 창은 1M 토큰이다. 사고 수준은 네 단계이며 기본값은 high, 최상위는 xhigh다. Gemini API 유료 등급(Google AI Studio, Android Studio), Gemini for business, Google Antigravity, Gemini 앱에서 발표 당일부터 순차 제공된다. SWE-sweep 점수는 mini-SWE 하네스에서 12.8%, Antigravity 하네스에서 39.0%로 같은 모델인데 세 배 넘게 벌어진다. 보안 특화 "Neon for cyber"는 Fairwind Program 참여자에게만 열린다. 수치는 모두 구글이 발표문에 적은 값이고 독립 재현 결과가 아니다.

## 1. 가격과 컨텍스트

| 항목 | 값 |
|------|------|
| 입력 | 1M 토큰당 $3 |
| 캐시 입력 | 1M 토큰당 $0.15 (95% 할인) |
| 출력 | 1M 토큰당 $15 |
| 컨텍스트 창 | 1M 토큰 |

캐시 입력 단가가 일반 입력의 95% 할인이라는 점이 가격표에서 가장 눈에 띈다. 시스템 프롬프트와 도구 정의가 길고 턴이 많은 에이전트 워크로드라면 캐시 적중률이 실제 청구액을 좌우한다. 캐시가 어떤 조건에서 적중하는지(최소 길이, 유지 시간 등)는 발표문에 없다. 이 부분은 API 문서에서 따로 확인해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The model is priced at $3 per million input tokens ($0.15 per million cached input tokens, a 95% discount) and $15 per million output tokens, with a 1 million-token context window."</blockquote></details>

## 2. 어디서 쓸 수 있나

발표문이 적은 접근 경로는 대상별로 넷이다.

- **개발자**: Gemini API 유료 등급(Google AI Studio, Android Studio). 유료 Google AI Pro·Ultra 구독자는 Google Antigravity에서, Pro·Ultra 구독자는 AI Studio에서도 써 볼 수 있다.
- **기업**: Gemini for business(API 또는 모델 드롭다운)와 Google Antigravity.
- **일반 사용자**: Google AI Pro·Ultra 구독자가 Gemini 앱의 모델 드롭다운에서 선택.
- **보안 담당자**: Fairwind Program 참여 자격이 있는 곳에 며칠에 걸쳐 열린다. 참여 요청은 Cloud 계정 담당 팀을 통한다.

발표문 본문에는 API 모델 식별자가 적혀 있지 않다. 다만 발표문의 AI Studio 링크가 `model=gemini-4-neon` 파라미터를 달고 있다. API 호출에 쓸 정확한 모델명은 AI Studio나 API 문서에서 확인하고 쓰는 편이 안전하다. "rolling out"이라는 표현대로 순차 제공이므로 계정에 따라 당일에 보이지 않을 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Gemini 4 Neon is rolling out today in the Gemini API in Google AI Studio, Gemini for business (via API and model dropdown), for paid Google AI Pro and Ultra subscribers in Google Antigravity, and for Pro and Ultra subscribers in the Gemini app."</blockquote></details>

## 3. 사고 수준 — 기본값 high, 최상위 xhigh

Neon의 사고 수준(thinking level)은 네 단계다. 발표문에 이름이 나오는 것은 기본값인 high와 그 위의 xhigh 둘뿐이고 나머지 두 단계의 이름은 본문에 없다.

구글이 수준별로 밝힌 비용 관련 수치다.

- **high(기본값)**: Carbon보다 작업당 비용이 32% 낮다. Vals Index와 Agent Arena에서는 상위 경쟁 프론티어 모델보다 작업당 비용이 대략 50%~80% 낮다고 한다.
- **xhigh**: 가장 어려운 엔지니어링·분석·과학 워크로드용이다. Neon의 지능 점수가 가장 높게 나오는 설정이며 경쟁 모델의 최고 설정보다 작업당 비용이 36% 이상 낮다고 한다.

비교 대상이 "작업당 비용"이라는 점에 주의한다. 토큰 단가가 아니라 한 작업을 끝내는 데 든 총비용이므로, 사고 수준을 올리면 단가는 같아도 청구액은 늘어날 수 있다. 벤치마크 점수가 어느 수준에서 측정됐는지도 항목마다 다르다. Artificial Analysis Intelligence Index의 61점은 xhigh에서, FrontierCode 2.0의 46.2%는 기본값 high에서 잰 값이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Neon’s four levels scale steadily, letting teams dial intelligence and token spend to what each workload needs."</blockquote><blockquote>"At its default high thinking level, it pairs a high Artificial Analysis Intelligence Index score with a 32% lower cost per task than Carbon."</blockquote></details>

## 4. 코딩 에이전트 수치 — 하네스가 점수를 가른다

발표문이 본문 텍스트로 적은 코딩 관련 수치다.

| 지표 | 값 | 조건 |
|------|------|------|
| Artificial Analysis Coding Agent Index | 58.6 | 전체 1위라고 발표 |
| Terminal-Bench 4.1 | 65.2% | |
| FrontierCode 2.0 | 46.2% | 기본값 high |
| SWE-sweep | 12.8% | mini-SWE 하네스 |
| SWE-sweep | 39.0% | Google Antigravity 하네스 |

이 표에서 실무적으로 쓸모 있는 줄은 SWE-sweep 두 줄이다. 모델은 같은데 하네스를 바꾸자 점수가 세 배 넘게 올랐다. 구글은 이를 Antigravity 하네스의 효과로 설명한다. 뒤집어 읽으면, Antigravity가 아닌 자체 에이전트 루프나 다른 코딩 에이전트에 Neon을 꽂았을 때는 발표문의 높은 쪽 수치를 기대할 근거가 없다는 뜻이다. 모델을 바꿔 볼 때는 자기 하네스에서 직접 재는 수밖에 없다.

발표문은 에이전트 동작에 대해서도 몇 가지를 적었다. 중복 파일 읽기를 줄이고 작업 중 진행 상황을 공유하며 긴 세션에서 컨텍스트 압축을 안정적으로 처리한다는 것이다. Antigravity에서는 Neon이 오케스트레이터가 되어 Hydrogen 서브에이전트를 병렬로 지휘하는 구성을 내세운다. 이 항목들에는 수치가 붙어 있지 않다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"On SWE-sweep, it already leads on the mini-SWE harness (12.8%) and more than triples to a state-of-the-art 39.0% in Google Antigravity showing what the Antigravity harness unlocks."</blockquote></details>

## 5. 업무·과학 벤치마크

발표문에 나열된 나머지 수치는 참고용으로만 모아 둔다. 전부 구글이 인용한 값이다.

| 지표 | 값 |
|------|------|
| Vals Index | 70.9% (2위) |
| Agent Arena Index | 1336 (2위, 4,200건 이상의 에이전트 세션 기준) |
| Agents' Last Exam v1.1 | 75.8% |
| Chartography | 73.5% |
| Zapier AutomationBench | 56.3% (1위) |
| Andon Labs Vending-Bench 2 | $16,843 (1위) |
| Terminal-Bench Science | 70.5% |
| RiemannBench | 84.0% |
| HLE-Diamond | 71.3% |

Legal Research Bench와 Tax Agent Bench에서는 1위라고만 적었고 점수는 본문에 없다. 조정 가능성(steerability) 항목에서 Agent Arena Index 2위라는 설명도 있는데, 작업 도중 사용자가 방향을 바꿨을 때 깔끔하게 따라오는 정도를 뜻한다.

## 6. 보안 관련 — 쓸 수 있는 것과 아직 못 쓰는 것

일반 Neon에 대해 구글이 밝힌 보안·안전 수치다.

- 보안 코드 생성: AutoBaxBench의 hard 등급에서 76%.
- 침투 테스트: WebExploitBench v1.2에서 51.26%. 경쟁 대형 모델의 절반 비용이라고 한다.
- 간접 프롬프트 인젝션: Gray Swan의 IPI 벤치마크에서, 실시간 보호 장치를 켠 최고 사고 수준 기준 공격 성공률 0.0%.
- 정당한 작업 거부율: Terminal-Bench 4.0에서 1.98%, Terminal-Bench-Science에서 0%.
- 에이전트 행동: 에이전트 안전성 평가에서 오정렬 행동 비율 0.09%, 에이전트 정직성 평가 통과율 99.2%, 코딩 부정행위 평가에서 지름길을 택한 비율 2.45%.

프롬프트 인젝션 수치에는 조건이 둘 붙어 있다. 실시간 보호 장치가 켜져 있고 사고 수준이 최고일 때의 값이다. 기본값 high에서 잰 수치는 발표문에 없으므로, 웹 페이지나 메일을 읽는 에이전트에 기본 설정으로 붙이면서 이 수치를 그대로 기대해서는 안 된다.

취약점 탐지·검증·패치에 맞춘 "Gemini 4 Neon for cyber"는 별도 경로다. SEC-bench Pro 90.5%, Collinear의 CWE-bench v2 70.7% 같은 수치가 붙어 있지만 Fairwind Program의 신뢰된 방어자에게만 순차 제공된다. 일반 API 키로 오늘 쓸 수 있는 모델이 아니다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"We found that with real-time safeguards at its highest thinking level, Neon resisted every attack within this Gray Swan benchmark, matching the 0.0% attack success rate reached only by other Gemini models."</blockquote><blockquote>"Gemini 4 Neon for cyber is rolling out to trusted defenders in the coming days as part of the Fairwind Program."</blockquote></details>

## 7. 읽을 때 걸러야 할 것

- **출처가 한쪽이다.** 이 글의 수치는 전부 구글 발표문 본문 텍스트에서 옮겼다. 외부 지표(Artificial Analysis, Vals, Arena 등)도 구글이 인용한 값이고 각 리더보드에서 다시 대조하지 않았다.
- **차트는 옮기지 않았다.** 발표문에는 사고 수준별 비용·점수 그래프가 있지만 그래프로만 제시된 값은 이 글에 넣지 않았다.
- **모델 카드는 대조하지 않았다.** 발표문이 링크한 모델 카드(PDF)는 이 글에서 읽지 않았다. 지식 기준일, 입출력 제한 같은 항목은 거기서 확인해야 한다.
- **한국어 성능 언급이 없다.** 발표문에 한국어나 다국어 평가는 나오지 않는다.
- **데모는 Antigravity 기준이다.** 본문의 3D·시뮬레이터·스프레드시트 시연은 모두 Google Antigravity에서 돌린 결과다.

## 8. 오늘 해볼 것

1. **접근 확인**: Gemini API 유료 등급 계정이면 AI Studio 모델 목록에 Neon이 보이는지 확인한다. 순차 제공이라 보이지 않을 수 있다.
2. **사고 수준을 명시하고 비교**: 기본값은 high다. 지금 쓰는 모델과 비교할 때는 high와 xhigh를 따로 재고 점수와 함께 작업당 비용을 기록한다.
3. **캐시 적중 점검**: 캐시 입력 할인 폭이 크므로, 프롬프트 앞부분(시스템 프롬프트, 도구 정의)이 턴마다 바뀌지 않는지 먼저 본다.
4. **자기 하네스에서 측정**: SWE-sweep의 12.8%와 39.0% 차이가 보여 주듯 하네스 영향이 크다. 발표문 수치 대신 실제로 돌리는 에이전트 루프에서 평가 세트를 돌린다.
5. **인젝션 방어는 따로 검증**: 외부 문서를 읽는 에이전트라면 기본 설정에서의 인젝션 저항을 자체 테스트로 확인한다. 발표문의 수치는 최고 사고 수준 조건이다.

## 참고

- [Introducing Gemini 4 Neon (Google DeepMind)](https://deepmind.google/blog/introducing-gemini-4-neon/){:target="_blank"}
- [Gemini 4 Neon 모델 카드 (PDF)](https://storage.googleapis.com/deepmind-media/Model-Cards/Gemini-4-Neon-Model-Card.pdf){:target="_blank"}
- [Google AI Studio에서 Gemini 4 Neon 열기](https://aistudio.google.com/prompts/new_chat?model=gemini-4-neon){:target="_blank"}
- [Google Antigravity](https://antigravity.google/){:target="_blank"}
- [Vals Index](https://www.vals.ai/benchmarks/vals_index){:target="_blank"}
- [Artificial Analysis Coding Agent Index](https://artificialanalysis.ai/agents/coding-agents#coding-agents-index){:target="_blank"}
- [Agent Arena 리더보드](https://arena.ai/leaderboard/agent){:target="_blank"}
- [관련 글: Gemini 4 Argon — 에이전트 운영자가 볼 가격·접근 경로·실측 수치]({{site.baseurl}}/dev/2026/10/20/gemini-4-argon-agent.html)
- [관련 글: llm-gemini 0.37과 Gemini 4 Hydrogen]({{site.baseurl}}/tools/2026/11/13/llm-gemini-037-gemini-4-hydrogen.html)
- [관련 글: OpenAI GPT-6.2 Astra·Sol 출시]({{site.baseurl}}/dev/2026/11/19/openai-gpt-6-2-astra-sol.html)
