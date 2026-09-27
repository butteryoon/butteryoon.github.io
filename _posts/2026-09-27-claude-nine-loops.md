---
layout: post
comments: true
title: "Claude가 N=4 초양-밀스 9루프 진폭을 계산했다 — 물리학자의 도전 한 달 만의 결과"
description: "이론물리학자 Matt von Hippel이 AI 기업에 던진 도전 과제(N=4 초양-밀스 9루프 산란 진폭)를 Fable 5.1 기반 Claude Science 하네스가 사실상 '계속하라'는 지시만으로 풀었다. 96 CPU로 1주일, 부트스트랩과 폼팩터 두 경로로 계산했고 Lance Dixon이 결과를 검증했다."
img: claude-nine-loops_title.webp
date: 2026-09-27 18:30:00 +0900
last_modified_at: 2026-09-27 20:20:00 +0900
tags: [anthropic, claude-science, ai-science, physics, amplitudeology, autonomous-agent, llm]
related: llm
categories: dev
source_url: https://www.anthropic.com/research/yes-claude-can-do-nine-loops
source_date: 2026-09-25
---
Anthropic 리서치 블로그에 9월 25일 올라온 게스트 포스트 [「Yes, Claude can do Nine Loops」](https://www.anthropic.com/research/yes-claude-can-do-nine-loops){:target="_blank"}를 읽고 정리했다. 이론물리학자 출신 과학 작가 Matt von Hippel이 8월 초 AI 기업들에 "내 옛 분야의 난제를 학계 수준 예산으로 풀어 보라"는 도전장을 냈는데, 한 달도 안 돼 Anthropic의 Claude Science 하네스가 N=4 초양-밀스 이론의 9루프 산란 진폭을 계산해 왔다. 흥미로운 대목은 결과 자체보다 von Hippel이 거기서 무엇을 배웠다고 말하는지다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Fable 5.1 기반 Claude Science가 평면 N=4 초양-밀스의 6입자(육각형) 진폭을 9루프까지 계산했다. 사람이 한 일은 문제 제시와 "계속하라"는 지시 정도였다. 부트스트랩과 폼팩터 두 방식으로 풀었고 방식마다 최종 사용자 비용은 1~2천 달러 수준이다. 그중 SymPy 부트스트랩 계산 자체는 96 CPU로 1주일, 약 100달러였다. Lance Dixon(SLAC·스탠퍼드)이 결과를 검증했다. 다만 새 방법을 만든 것이 아니라 기존 레시피를 사람들이 시도하지 않은 규모로 밀어붙인 결과이고 von Hippel은 "생각보다 따기 쉬운 열매가 많다"는 것을 가장 큰 교훈으로 꼽는다.

## 1. 원문 정보

- **원문**: [Yes, Claude can do Nine Loops](https://www.anthropic.com/research/yes-claude-can-do-nine-loops){:target="_blank"} (Anthropic, 2026-09-25)
- **필자**: Matt von Hippel. 이론물리학자로 일하다 지금은 과학 작가로 활동하며 [4gravitons.com](https://4gravitons.com/){:target="_blank"}에 매주 글을 쓴다. 원문 끝 공시에 따르면 Anthropic이 원고를 청탁하고 대가를 지불했으며 내용과 의견은 필자 본인의 것이다.
- **계산 수행**: Anthropic 소속 물리학자 Liam Fitzpatrick, Siddharth Mishra-Sharma
- **검증**: Lance Dixon(SLAC 국립가속기연구소·스탠퍼드대 교수)이 독립적으로 검증했다. 부록 글도 직접 썼다.
- **동시 결과**: 베이징 중국과학원의 Song He 그룹이 GPT-6의 도움을 일부 받아 9루프 진폭의 심볼(symbol) 부분을 따로 계산했다.

## 2. 배경: 진폭학과 '루프'의 벽

산란 진폭은 아원자 입자의 운동량과 에너지로 특정 반응이 일어날 확률을 계산하는 공식이다. 너무 계산하기 어려워서 물리학자들은 입자 간 상호작용의 복잡도를 나타내는 '루프' 수에서 계산을 끊고 근사한다. 루프를 늘릴수록 실제 답에 가까워지지만 그만큼 계산이 어려워진다. 원문에 따르면 대부분의 산란 진폭은 2루프까지만 계산돼 있고 3루프까지 간 것은 몇 개뿐이다. 입자물리학에서 가장 정밀하다는 예측(전자 이상 자기모멘트)도 5루프를 썼다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"In practice, most scattering amplitude formulas have only been calculated to two loops. A few have three. The most precise prediction in particle physics you might have heard of used five."</blockquote></details>

진폭학자들은 새 기법을 '장난감 모델'에 먼저 시험한다. N=4 초양-밀스 이론에서는 입자마다 초대칭 짝이 넷씩 붙는다. 현실과는 거리가 멀지만 입자들 사이의 미묘한 균형 덕분에 필요한 변수 조합이 줄어들어 오히려 계산이 쉽다. von Hippel은 박사 과정에서 3루프 진폭 계산에 참여했고 Dixon은 2023년 Andy Liu와 함께 폼팩터와 '대척 쌍대성(antipodal duality)'이라는 대칭을 이용해 8루프 진폭을 얻었다([arXiv:2308.08199](https://arxiv.org/abs/2308.08199){:target="_blank"}).

이 계산들은 '부트스트랩'이라는 기법으로 이뤄졌다. 모든 상호작용을 일일이 따지지 않고 답이 대략 어떤 모양일지 정해 둔 뒤 특수한 '알파벳'으로 가능한 후보를 모두 적어 놓는다. 그다음 다른 계산법의 예측, 답이 지켜야 할 규칙, 더 쉬운 관련 문제와의 연결을 하나씩 대조하며 후보를 지워 간다. von Hippel은 이것을 스도쿠에 비유한다. 다만 Dixon조차 9루프 진폭을 직접 푸는 건 너무 어렵다고 보고 8루프 때보다 더 간접적인 경로를 예상하고 있었다.

## 3. Claude Science가 한 일

두 연구자는 Fable 5.1을 Claude Science 위에서 돌렸다. Claude Science는 과학자들이 유료로 쓰는 플랫폼으로, 원문 표현으로는 구조화된 규칙과 프롬프트로 Claude LLM을 감싸 더 견고하고 과학적으로 쓸모 있게 만든 '하네스'다. 이들은 먼저 Claude에게 어느 문제를 풀 가능성이 가장 높은지 물었고 이어 짧은 프롬프트 하나를 줬다.

> "The problem is to compute the Six-particle (hexagon) amplitude in planar N=4 SYM at nine loops."

그 뒤로는 계속하라는 말만 했다.

> "I'm going to sleep and won't be available for another several hours. Keep working on this until I tell you to stop. Give me updates every 4-6 hours."

Claude는 원래의 부트스트랩과 간접적인 폼팩터 방식, 두 가지로 계산을 해냈다. 최종 사용자 기준 비용은 방식마다 1~2천 달러 정도이고 대부분 Claude를 오래 돌린 비용이다. Python과 SymPy로 한 부트스트랩 계산 자체는 약 100달러로, 96 CPU를 1주일 돌린 규모다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Claude ended up doing the calculation two different ways: the original bootstrap, and the indirect form-factor approach. Either approach would have cost an end-user around one or two thousand dollars, mostly due to the expense of running Claude for so long. The bootstrap calculation, done with the Python programming language with package SymPy, took around $100 of the budget, corresponding to running 96 CPUs for a week."</blockquote></details>

Dixon은 9월 1일에 연락을 받았다. 9루프 진폭에서 폼팩터로 되돌아가는 쪽이 비교적 쉬워서 주로 그 경로로 검증했다고 한다. 그가 감탄한 대목은 계산량이 아니었다. 설정 전체가 워낙 깨지기 쉬워서 레시피 어디서든 실수 하나만 나와도 "실패한 수플레처럼" 무너지는데, 논문에 다 적기엔 지루한 구현 세부가 많아 Claude가 코드를 전부 처음부터 짜야 했다는 점이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Not so much because it was a big computational task, but because the whole setup is very fragile: if you make any mistake at all in the computational recipe, it all crashes down like a failed soufflé, and you are left to wonder why (and debug)."</blockquote></details>

사람 쪽 결과도 멀지 않았다. von Hippel이 Anthropic 연락을 받고 며칠 뒤 Song He 그룹이 이미 결과의 대부분을 얻었다고 알려 왔다. 이들도 GPT-6 기반 AI를 썼지만 일부 제약 조건 계산에만 썼고 전체 틀은 사람이 짰다.

## 4. von Hippel이 얻은 교훈

| 항목 | 원문이 말하는 것 |
|------|------|
| **방법** | 새로운 방법이 아니다. 알려진 방법을 사람들이 시도하던 것보다 조금 더 큰 연산량으로 돌렸다. Maple이나 Mathematica 대신 Python을 쓴 점, 더 나은 소프트웨어 공학 관행이 도움이 됐을 수 있지만 "초지능적인 수준은 아니었다". |
| **자율성** | "계속하라" 수준의 감독만 받고 한 번에 끝냈다. 도중에 내부적으로 실수를 얼마나 했는지는 알 수 없지만 외부 협업자의 개입 없이 하네스가 끝까지 끌고 갔다. |
| **예산** | 96 CPU로 1주일. 10년 전에는 큰 규모였지만 지금은 이유만 있으면 감당할 만하다. |
| **결과 형식** | Dixon 그룹이 수년간 개발한 방법을 모두 썼고 기존 루프 차수 결과와 같은 형식으로 답을 내놨다. |

von Hippel이 꼽은 가장 큰 교훈은 생각보다 따기 쉬운 열매가 많다는 것이다. 목표가 단순하고 잘 정의돼 있어도 전문가 눈에는 실제보다 훨씬 어려워 보일 때가 있다. 몇 년 동안 "프로그래머 몇 명만 뽑아도 진폭학이 훨씬 빨리 나아갈 것"이라고 말해 온 컴퓨터과학 쪽 사람들은 이번 일로 옳았음을 증명받은 셈이라고 그는 쓴다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"My biggest takeaway is that there is more low-hanging fruit out there than you'd expect."</blockquote></details>

또 하나는 신뢰성이다. 본인이 96 CPU로 1주일짜리 계산을 했다면 첫 시도에 거의 틀림없이 뭔가를 망쳐 2주가 걸렸을 거라고 한다. 그래서 AI가 쓸모없을 만큼 실수가 잦다고 생각하는 사람에게 전하는 요지는 이렇다. 이제 이런 일은 믿고 맡길 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"I don't know how many mistakes Claude made internally on the way, but the harness got it to the end without an outside collaborator's input. ... It can do this kind of thing reliably now."</blockquote></details>

반면 그가 정말 알고 싶던 질문에는 답을 얻지 못했다. 그는 AI가 계산 장벽을 예상 밖의 방식으로, 새로운 계산법으로 넘는 모습을 보고 싶었다. 초지능 논쟁에서 "지능과 연산력을 혼동한다"는 비판에 답할 단서를 찾고 싶었던 것이다. 그런데 배운 것은 한계가 어디 있는지에 대해 자신이 너무 순진했다는 사실 정도였다. 현실 세계의 진폭 계산은 경쟁이 치열해 쉬운 열매가 적을 수 있다면서도 그쪽에서도 AI 과학 하네스가 프런티어 계산을 한 번에 해낼 수 있는지 확인해 봐야 하고 결과를 검증할 계획도 세워야 한다고 덧붙인다.

Dixon의 부록 글 결론도 비슷하다. 기계에 선수를 빼앗긴 게 괴롭지 않냐는 질문에 그는 아니라고 답한다. 그의 팀은 원래 "기계가 내놓는 어떤 후보 해도 검증할 도구가 있다"는 구호를 내걸고 있었고 Claude가 자기 팀이 쌓아 온 방법을 그대로 따랐으니 "Claude의 결과를 검증하는 동안 Claude는 우리의 이전 작업을 검증하고 있다"는 것이다. 다만 더 깊은 성찰이 필요한 순간은 LLM이 사람보다 먼저 새로운 물리 원리와 통찰을 내놓기 시작할 때라고 선을 긋는다.

## 5. 에이전트 설계 관점의 시사점

- **하네스가 감독을 대신한다.** 사람이 준 것은 문제 정의 한 줄과 "4~6시간마다 보고하라"는 지시뿐이었다. 장시간 자율 작업에서 모델 성능만큼 중요한 것이 규칙·프롬프트·도구를 묶는 하네스라는 점을 보여 준다. 앞서 다룬 [Karpathy의 Autoresearch Loop]({{site.baseurl}}/dev/2026/08/30/karpathy-autoresearch-loop.html)와 같은 방향의 사례다.
- **검증 가능성을 먼저 확보한 문제를 골랐다.** Dixon 그룹은 이미 결과를 검증할 도구와 결과 형식을 갖추고 있었고 Claude는 그 형식에 맞춰 답을 냈다. 자율 에이전트에 일을 맡길 때 결과를 어떻게 확인할지부터 정해 두라는 von Hippel의 조언과도 맞닿는다.
- **새로운 발견이 아니라 규모와 끈기의 문제였다.** 기존 레시피를 실수 없이 끝까지 실행하는 능력만으로도 전문가들이 미뤄 둔 문제가 풀렸다. '따기 쉬운 열매'가 다른 분야에도 남아 있을 가능성을 시사한다.

## 6. 참고 자료

- 원문: [Yes, Claude can do Nine Loops](https://www.anthropic.com/research/yes-claude-can-do-nine-loops){:target="_blank"}
- 9루프 전체 결과(기존 루프 차수와 같은 형식): [smsharma.io/cosmic-nine-loops](https://smsharma.io/cosmic-nine-loops/){:target="_blank"}
- Song He, Jirong Jing, Xiang Li의 동시 9루프 결과: [Zenodo 10.5281/zenodo.22800071](https://doi.org/10.5281/zenodo.22800071){:target="_blank"}
- Matt von Hippel의 도전 글: [It Only Counts When AI Gets to My Field](https://4gravitons.com/2026/08/07/it-only-counts-when-ai-gets-to-my-field/){:target="_blank"}
- Dixon·Liu 8루프 논문: [An Eight Loop Amplitude via Antipodal Duality (arXiv:2308.08199)](https://arxiv.org/abs/2308.08199){:target="_blank"}
- 대척 쌍대성 해설: [Quanta Magazine (2022)](https://www.quantamagazine.org/particle-physicists-puzzle-over-a-new-duality-20220801/){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/OPpCbAAKWv8){:target="_blank"}
