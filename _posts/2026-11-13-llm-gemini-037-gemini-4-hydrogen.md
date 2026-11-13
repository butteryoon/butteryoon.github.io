---
layout: post
comments: true
title: "llm-gemini 0.37: Gemini 4 Hydrogen을 CLI에서 쓰기 전에 볼 가격표와 실측 비용"
description: "Simon Willison의 llm-gemini 0.37이 Gemini 4 Hydrogen을 지원한다. 설치와 키 설정, thinking_level 옵션을 README 기준으로 정리하고, 12월 1일 Carbon·Argon 가격 인상 전후의 가격표와 같은 프롬프트를 세 모델에 돌린 실측 비용·시간을 함께 본다."
img: llm-gemini-037-gemini-4-hydrogen_title.webp
date: 2026-11-13 19:30:00 +0900
last_modified_at: 2026-11-13 19:30:00 +0900
tags: [llm-serving, llm, gemini, gemini-4-hydrogen, llm-gemini, simon-willison, pricing]
related: llm-serving
categories: tools
source_url: https://simonwillison.net/2026/Nov/12/llm-gemini/
source_date: 2026-11-12
---

Simon Willison이 `llm` CLI용 플러그인 [llm-gemini 0.37](https://github.com/simonw/llm-gemini/releases/tag/0.37){:target="_blank"}을 냈다. 바뀐 것은 하나다. 같은 날 구글이 발표한 Gemini 4 Hydrogen을 `llm -m gemini-4-hydrogen`으로 부를 수 있다. 릴리스 글에서 Simon은 Gemini 4 가격 체계가 헷갈린다며 표로 정리했고 같은 프롬프트를 세 모델에 세 가지 사고 수준으로 돌린 비교 격자도 붙였다. 이 글은 그 가격표와 실측 수치, 그리고 플러그인 README에 적힌 설치·설정 방법을 한데 모은다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** `llm install llm-gemini` 후 `llm keys set gemini`로 키를 넣으면 `llm -m gemini-4-hydrogen`을 쓸 수 있다. Hydrogen은 입력 100만 토큰당 $1.50, 출력 $7.50이다. Carbon과 Argon은 11월 30일까지 $2/$10로 서로 같고 12월 1일부터 $4/$20로 오른다. 그래서 지금은 Hydrogen이 25% 쌀 뿐이지만 12월부터는 격차가 벌어진다. Simon의 펠리컨 SVG 실험에서 Hydrogen은 high 수준 한 번에 3.99센트·19초, Carbon은 20.31센트·4분 11초가 들었다. 다만 그림 품질 차이도 뚜렷했다. Hydrogen은 무료 티어 모델이기도 하다.

## 설치와 키 설정

`llm`이 깔려 있는 환경에 플러그인을 추가한다. 아래 명령은 모두 [llm-gemini README](https://github.com/simonw/llm-gemini){:target="_blank"}에 있는 그대로다.

```bash
llm install llm-gemini
llm keys set gemini
# 프롬프트가 뜨면 Google AI Studio에서 발급한 키를 붙여 넣는다
```

키를 저장하지 않고 환경 변수로 넘기려면 `LLM_GEMINI_KEY`를 쓴다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"You can also set the API key by assigning it to the environment variable `LLM_GEMINI_KEY`."</blockquote></details>

0.37에서 추가된 모델 ID는 `gemini-4-hydrogen`이다. Carbon과 Argon은 `gemini-4-carbon`, `gemini-4-argon`으로 이미 등록돼 있다.

```bash
llm -m gemini-4-hydrogen "A short joke about a pelican and a walrus"
```

<details class="evidence"><summary>원문 근거</summary><blockquote>"Gemini 4 Hydrogen, as `llm -m gemini-4-hydrogen`. #162"</blockquote></details>

### 사고 수준 지정

Gemini 3 이후 모델은 응답 전에 얼마나 추론할지를 `thinking_level` 옵션으로 정한다. 지정하지 않으면 모델 기본값을 쓴다.

```bash
llm -m gemini-4-argon -o thinking_level high \
  'Prove that there are infinitely many primes'
```

README의 표를 보면 `gemini-4-carbon`, `gemini-4-argon`, `gemini-4-hydrogen` 세 모델 모두 `low`, `medium`, `high`를 받는다. 구글 발표문은 Hydrogen에 `xhigh`까지 네 단계가 있고 기본값이 high라고 설명하는데, 0.37 README의 Hydrogen 항목에는 `xhigh`가 없다. `xhigh`를 써야 한다면 플러그인이 받아 주는지 먼저 확인해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Hydrogen’s four thinking levels — low, medium, high (default), and a new xhigh tier — offer a wide range of options across quality, latency, and cost"</blockquote></details>

## Gemini 4 가격표

Simon이 정리한 표를 옮기면 다음과 같다. 단위는 100만 토큰당 달러다.

| 모델 | 기간 | 입력 | 캐시 입력 | 출력 |
|------|------|------|-----------|------|
| Gemini 4 Carbon | 2026년 11월 30일까지 | $2.00 | $0.10 | $10.00 |
| Gemini 4 Carbon | 2026년 12월 1일부터 | $4.00 | $0.20 | $20.00 |
| Gemini 4 Argon | 2026년 11월 30일까지 | $2.00 | $0.10 | $10.00 |
| Gemini 4 Argon | 2026년 12월 1일부터 | $4.00 | $0.20 | $20.00 |
| Gemini 4 Hydrogen | 기간 구분 없음 | $1.50 | $0.075 | $7.50 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"Carbon and Argon are the same price, and both are scheduled to double in price on December 1st."</blockquote></details>

표에서 읽을 것은 세 가지다.

첫째, Carbon과 Argon은 단가가 같다. 가격이 같은데 모델이 둘인 이유를 구글 가격 페이지는 Argon이 턴당 토큰을 덜 쓴다는 점으로 설명한다. 단가는 같아도 작업당 비용은 다를 수 있다는 뜻이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"delivers Gemini 4 intelligence with faster responses and lower token spend per turn compared to Gemini 4 Carbon"</blockquote></details>

둘째, 저가 모델이라는 Hydrogen이 지금은 별로 싸지 않다. 11월 말까지는 상위 모델보다 25% 낮은 값이다. Simon은 12월 1일 이후에는 할인 폭이 75%가 된다고 썼는데, 위 표의 숫자($1.50 대 $4.00)로 직접 계산하면 62.5%다. 어느 쪽이든 12월부터 격차가 커진다는 방향은 같다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Hydrogen, supposedly the cheap model, is currently only a 25% discount on its more capable peers. The Hydrogen price will make more sense after December 1st, when it will represent a 75% discount."</blockquote></details>

셋째, 다른 회사의 소형 모델과 견주면 Hydrogen은 여전히 비싸다. Simon은 Claude Haiku 5.5와 GPT-6 Luna가 둘 다 $0.10/$0.50이라 Hydrogen이 그 15배라고 적었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"It's still considerably more expensive than Claude Haiku 5.5 and GPT-6 Luna - they're both $0.10/$0.50, so Hydrogen is 15x that."</blockquote></details>

Hydrogen은 새 무료 티어 모델이기도 하다. 구글 가격 페이지는 Argon이 무료 티어에서 제공되지 않는다고 밝히고 있으니, 키만 발급받아 돈을 내지 않고 Gemini 4를 시험해 볼 수 있는 모델은 Hydrogen이다.

## 같은 프롬프트로 본 실측 비용과 시간

단가만으로는 실제 청구액을 알 수 없다. 사고 토큰이 출력 토큰에 포함되고 모델마다 쓰는 양이 다르기 때문이다. Simon은 "Generate an SVG of a pelican riding a bicycle" 프롬프트(입력 40토큰)를 세 모델에 세 사고 수준으로 돌려 [비교 격자](https://static.simonwillison.net/static/2026/gemini-4-pelicans.html){:target="_blank"}를 공개했다. 비용은 11월 현재 단가 기준이다.

| 사고 수준 | gemini-4-carbon | gemini-4-argon | gemini-4-hydrogen |
|-----------|-----------------|----------------|-------------------|
| high | 출력 20,305토큰 · 20.31센트 · 4분 11초 | 출력 17,158토큰 · 17.17센트 · 2분 29초 | 출력 5,311토큰 · 3.99센트 · 19초 |
| medium | 출력 11,020토큰 · 11.03센트 · 1분 53초 | 출력 11,090토큰 · 11.10센트 · 1분 32초 | 출력 3,964토큰 · 2.98센트 · 17초 |
| low | 출력 9,153토큰 · 9.16센트 · 1분 34초 | 출력 10,177토큰 · 10.18센트 · 1분 21초 | 출력 2,996토큰 · 2.25센트 · 10초 |

단가 차이는 25%인데 이 실험의 호출당 비용은 Hydrogen이 Carbon의 5분의 1에서 4분의 1 수준이다. 출력 토큰을 훨씬 적게 쓰기 때문이다. 응답 시간 차이는 더 커서 high 기준 19초 대 4분 11초다.

Argon의 "턴당 토큰 절감"은 이 한 번의 실험에서는 high 수준에서만 나타났다. medium과 low에서는 Argon이 Carbon보다 출력 토큰을 조금 더 썼다. 프롬프트 하나, 실행 한 번의 결과이므로 일반화할 수는 없고 자기 작업으로 직접 재 봐야 한다는 근거 정도로 읽으면 된다.

싸고 빠른 대신 결과물 차이는 분명했다고 Simon은 평가한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Considering how close they are in price, the difference in pelican quality for Hydrogen is quite stark."</blockquote></details>

## 어떤 모델을 고를까

구글이 설명하는 Hydrogen의 자리는 실행 담당이다. 설계 판단이나 까다로운 버그 분석은 Carbon·Argon이 맡고 코드 작성과 테스트 실행, 변경 검증은 Hydrogen이 맡는 구성을 발표문이 직접 제시한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Hydrogen is built to stand on its own as a model for everyday work — or to pair with Carbon and Argon as the high-efficiency execution engine in multi-model workflows."</blockquote></details>

위 수치들을 놓고 정리하면 판단 기준은 이렇게 잡을 수 있다.

- 12월 1일부터 Carbon·Argon 단가가 두 배가 된다. 이 두 모델로 돌리는 배치나 에이전트가 있으면 지금 월 비용에 2를 곱해 보고 일부 단계를 Hydrogen으로 내릴 수 있는지 11월 안에 시험해 두는 편이 좋다.
- Carbon과 Argon 사이에서는 단가가 아니라 작업당 토큰 사용량으로 골라야 한다. 위 실험처럼 사고 수준에 따라 우열이 뒤집힐 수 있다.
- Hydrogen으로 내릴 단계는 품질 저하를 감당할 수 있는 곳이어야 한다. 펠리컨 실험은 그 차이가 작지 않다는 것을 보여 준다.
- 비용이 가장 중요하고 Gemini일 필요가 없다면, Simon이 든 $0.10/$0.50급 모델이 Hydrogen보다 15배 싸다.

`llm`에서는 `-m` 값만 바꾸면 모델이 바뀌므로 단계별로 모델을 나눠 쓰기 쉽다. 아래는 원문에 있는 예제가 아니라 그 구성을 `llm`의 표준 입력 파이프로 옮겨 본 예시다.

```bash
# 설계 판단은 상위 모델에
llm -m gemini-4-carbon -o thinking_level high \
  "이 서비스에 캐시 레이어를 넣을 때 고려할 점을 정리해줘" > design.md

# 구현은 Hydrogen에 — 앞 단계 결과를 표준 입력으로 넘긴다
cat design.md | llm -m gemini-4-hydrogen \
  "이 설계를 바탕으로 Redis 캐시 래퍼 클래스를 작성해줘" > cache.py
```

같은 프롬프트를 모델과 `thinking_level`만 바꿔 돌린 뒤 `llm logs -n 1 --short --usage`로 토큰 사용량을 확인하면, 위 표와 같은 실측을 자기 작업으로 만들 수 있다.

## 참고

- 원문: [Simon Willison — Release: llm-gemini 0.37](https://simonwillison.net/2026/Nov/12/llm-gemini/){:target="_blank"}
- 릴리스 노트: [llm-gemini 0.37](https://github.com/simonw/llm-gemini/releases/tag/0.37){:target="_blank"}
- llm-gemini 저장소: [github.com/simonw/llm-gemini](https://github.com/simonw/llm-gemini){:target="_blank"}
- llm CLI 문서: [llm.datasette.io](https://llm.datasette.io/){:target="_blank"}
- 구글 가격 페이지: [aistudio.google.com/docs/pricing](https://aistudio.google.com/docs/pricing){:target="_blank"}
- 구글 발표문: [Introducing Gemini 4 Hydrogen](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-hydrogen/){:target="_blank"}
- 비교 격자: [Gemini 4 SVG thinking-level comparison](https://static.simonwillison.net/static/2026/gemini-4-pelicans.html){:target="_blank"}
- 관련 글: [Gemini 4 Argon — 에이전트 운영자가 볼 가격·접근 경로·실측 수치]({{site.baseurl}}/dev/2026/10/20/gemini-4-argon-agent.html)
