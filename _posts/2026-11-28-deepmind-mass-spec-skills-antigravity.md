---
layout: post
comments: true
title: "검증된 도구를 스킬로 감싸면 경량 모델도 따라온다: Google DeepMind의 Mass Spec Skills 실측"
description: "Google DeepMind가 Google Antigravity용 질량분석 스킬 묶음 Mass Spec Skills를 공개하면서 낸 비교 실험을 정리한다. 스킬 없이 인터넷만 준 에이전트가 왜 놓쳤는지, 스킬을 붙였을 때 정확도와 생성 토큰이 어떻게 달라졌는지, 점수가 같은 후보를 어떻게 가려냈는지, 설치 경로와 한계를 원문 기준으로 적는다."
img: deepmind_mass_spec_skills_title.webp
date: 2026-11-28 19:00:00 +0900
last_modified_at: 2026-11-28 19:00:00 +0900
tags: [llm, llm-agent, deepmind, google-antigravity, gemini, agent-skills, mass-spectrometry, tool-use]
related: llm-agent
categories: dev
source_url: https://deepmind.google/blog/interpreting-mass-spectra-with-google-antigravity/
source_date: 2026-11-25
---

Google DeepMind가 Google Antigravity에서 쓰는 질량분석 스킬 묶음 Mass Spec Skills를 공개했다. 주제는 탠덤 질량 스펙트럼(MS/MS)에서 분자 구조를 찾는 일이라 화학 분석을 하지 않는 사람에게는 먼 이야기처럼 보인다. 그런데 글 뒤쪽에 에이전트를 운영하는 사람이 그대로 가져갈 비교 실험이 있다. 같은 모델, 같은 과제에서 인터넷 접근만 준 경우와 스킬까지 붙인 경우를 나란히 잰 것이다. 이 글은 그 실험과 스킬의 구성 방식을 중심으로 정리한다. (이 글의 초안은 Orca Bots 가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Mass Spec Skills는 matchms, SIRIUS, MAGMa, MetFrag, RDKit 같은 공개 도구와 GNPS, MoNA, PubChem 조회를 스킬로 묶어 Antigravity의 에이전트가 호출하게 한 것이다. 공개 라이브러리에 있는 100개 화합물로 잰 결과, 인터넷도 스킬도 없으면 세 모델 모두 정확도가 4% 아래였다. 인터넷만 주면 에이전트가 검색 스크립트를 그때그때 새로 짰고 프런티어 모델은 스펙트럼의 4분의 1에서 3분의 1을, 경량 모델은 4분의 3 넘게 놓쳤다. 스킬을 붙이자 세 모델 모두 약 97%가 됐고 경량 모델인 Gemini 3.8 Flash가 프런티어 모델과 같은 정확도를 냈다. 시도당 생성 토큰도 줄었다. 스킬은 Antigravity 앱에 Mass Spec 플러그인을 설치해 쓴다.

원문은 2026년 11월 25일 Google DeepMind가 올린 [From spectrum to structure: Identifying molecules with Mass Spec Skills for Google Antigravity](https://deepmind.google/blog/interpreting-mass-spectra-with-google-antigravity/){:target="_blank"}이다. 저자는 Robert Brunck, Stefan Ganscha, Swaroop Guntupalli, Dariusz Lasecki, Gabriella Marfani, Jack Mason, Hussain Masoom, Rose Mirshafian, Eliseo Papa, Zoi Tsangalidou로 적혀 있다.

## 풀려는 문제

질량분석기는 시료 한 방울에서 수만 개의 분자 신호를 잡지만 측정값이 구조를 바로 알려 주지는 않는다. 표준 방법은 이미 측정된 분자의 참조 스펙트럼과 대조하는 것인데, 원문에 따르면 이 방식으로 주석이 붙는 스펙트럼은 평균 약 5%, 사람 시료에서 10%다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Yet even after a decade of expanding reference libraries, spectral searches annotate on average only about 5% of measured spectra, and 10% for human samples; the rest has been called the "dark matter" of mass spectrometry."</blockquote></details>

분석화학자가 쓰는 절차는 정해져 있다. 참조 라이브러리를 확인하고 분자식을 찾아 화학 데이터베이스에서 후보 구조를 가져온 다음, 후보마다 관측된 조각 피크와 맞는지 따진다. 단계별 도구는 이미 공개돼 있다. 문제는 전문가가 스펙트럼 하나마다 이 절차를 손으로 돌리는 데 드는 시간이고 연구 하나가 스펙트럼 수천 개를 낸다는 점이다.

원문은 언어 모델 혼자서는 이 일을 못 한다고 적는다. 조각을 구조에 맞추려면 정밀한 질량 계산, 수백만 화합물 규모의 데이터베이스 검색, 결합이 끊어지는 과정의 시뮬레이션이 필요하기 때문이다. 그래서 추론은 Gemini가 하고 계산과 조회는 기존 도구가 하도록 나눴다.

## 스킬 구성

원문이 그림 1과 예시로 보여 주는 단계는 넷이고 단계마다 맡는 도구가 다르다.

- **준비**: matchms로 강도 임계값을 적용해 배경 잡음을 걷어낸다. 예시에서는 음이온 모드 전구체 피크 m/z 395.1387을 중성 질량 396.1459 Da로 바꾸고 낮은 강도의 잡음 피크 1,700개 이상을 지웠다.
- **라이브러리 검색**: GNPS와 MoNA의 실험 MS/MS 참조 라이브러리를 조회하고 코사인 유사도로 점수를 매긴다. 질량 허용 오차는 사용자가 지정하거나 장비에 맞춘 값을 쓴다.
- **분자식에서 구조로**: SIRIUS의 조각 트리 점수로 분자식을 정하고 PubChem에서 이성질체를 가져와 순위를 매긴다. 예시에서는 C20H21ClN6O를 질량 오차 −1.5 ppm으로 찾았고 점수는 37.86, 차점 후보는 28.33이었다.
- **교차 확인**: MAGMa, MetFrag, 중성 손실 쌍 조회로 피크를 조각에 대응시키고 RDKit으로 최종 구조가 화학적으로 유효한지, 정확한 질량과 분자식에 맞는지 확인한다.

사용자는 도구를 하나씩 설정하지 않고 과제와 장비별 허용 오차를 자연어로 적는다. 에이전트가 필요한 스킬을 골라 이어 붙이고 프롬프트 하나로 주석 없는 스펙트럼에서 구조 제안과 감사 가능한 보고서까지 간다고 원문은 설명한다. 각 단계는 따로 돌릴 수도 있다. 원문 예시의 프롬프트는 "Clean this spectrum.", "Search reference libraries for this spectrum.", "Predict the molecular formula for this spectrum." 같은 한 줄이다.

## 인터넷만 준 에이전트와 스킬을 붙인 에이전트

실험 대상은 MoNA의 Enamine REAL MS/MS 컬렉션에서 뽑은 100개 화합물이다. 구조가 PubChem에도 등록돼 있는 것들이고 거의 쪼개지지 않은 기록은 걸러냈다. 참조 스펙트럼이 공개 라이브러리에 이미 있는 화합물이므로 찾아서 확인하기만 하면 되는 과제다.

구성은 셋이다. (1) 인터넷도 스킬도 없는 Antigravity, (2) 사용자가 허용한 인터넷 접근으로 공개 라이브러리와 데이터베이스를 조회할 수 있는 Antigravity, (3) 같은 접근에 Mass Spec Skills를 더한 Antigravity. 모델은 경량인 Gemini 3.8 Flash와 프런티어 모델인 Gemini 4 Argon, Gemini 4 Carbon이다.

| 구성 | 원문이 적은 결과 |
|---|---|
| 인터넷 없음, 스킬 없음 | 세 모델 모두 정확도 4% 미만 |
| 인터넷만 | 프런티어 모델은 스펙트럼의 4분의 1에서 3분의 1을 놓침, 경량 모델은 4분의 3 넘게 놓침 |
| 인터넷 + 스킬 | 세 모델 모두 약 97% |

이 표는 원문 본문의 문장을 옮긴 것이다. 모델별 정확한 값은 그림 5a의 막대그래프에만 있고 읽지 못했으므로 적지 않았다.

눈여겨볼 것은 가운데 줄이다. 답이 공개 라이브러리에 있고 인터넷도 열려 있는데 에이전트는 놓쳤다. 원문이 든 이유는 에이전트가 검색 스크립트를 처음부터 새로 짰고 그 스크립트가 전구체 질량 창, 부가체, 잡음 피크 처리에서 쉽게 깨졌다는 것이다. 모델의 지식이 부족해서가 아니라 매번 즉석에서 만든 도구가 불안정해서 생긴 실패다.

토큰도 같은 방향으로 움직였다. 인터넷도 스킬도 없을 때 시도당 토큰이 가장 많았고 인터넷만 있을 때도 API 질의를 즉석에서 만들고 일회용 스크립트를 디버깅하는 데 토큰을 많이 썼다. 스킬을 붙이자 시도당 생성 토큰이 세 모델 모두에서 줄었다. 원문 표현으로 Flash는 "거의 4배(nearly four times)", Argon은 절반, Carbon은 거의 5분의 1만큼이다. 그림 5b는 이를 Gemini 4 Argon의 인터넷 구성을 1.0×로 둔 상대값으로 보여 준다.

측정 조건은 그림 5a 캡션에 있다. 스펙트럼당 세 번 시도했고 벽시계 기준 한 시간 예산을 넘긴 시도는 실패로 처리했다. 일시적인 인프라 오류와 할당량 오류는 제외했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Giving the agent limited internet access allowed it to query online libraries, but custom search scripts written from scratch proved brittle around precursor windows, adducts, and noisy peaks, causing the frontier models to miss a quarter to a third of the spectra and the lightweight model to miss more than three-quarters."</blockquote><blockquote>"With the Mass Spec Skills added, all three models identified and verified the exact molecule in around 97% of attempts, with the lightweight model matching the frontier models' accuracy (Figure 5a)."</blockquote><blockquote>"With the Mass Spec Skills, the agent in Antigravity calls validated, ready-made skills instead, cutting generated tokens per attempt across every model: by nearly four times on Flash, by half on Argon, and by nearly a fifth on Carbon (Figure 5b)."</blockquote></details>

## 점수가 같은 후보를 가려내는 단계

원문의 예시 하나는 도구 점수만으로 끝내지 않는 이유를 보여 준다. 라이브러리 조회를 끈 상태에서 에이전트는 분자식 C20H21ClN6O를 정한 뒤 PubChem의 이성질체 845개를 모두 가져와 점수를 매겼다. 상위 후보 여럿이 전처리된 피크 19개 가운데 14개, 조각 강도 합의 97.7%를 똑같이 설명했다.

여기서 에이전트는 후보의 결합 구조가 관측된 조각 경로를 실제로 만들 수 있는지 따졌다. 스펙트럼에는 HCl과 이소시안산(HNCO)이 순서를 바꿔 가며 빠지는 닫힌 고리가 있었고 이는 가운데에 요소 결합이 있다는 단서다. 요소 결합이 없는 후보는 HNCO를 잃을 경로가 없어 탈락했다. 가운데 요소 결합이 측정된 두 조각으로 갈라지는 구조만 남기자 요소 위치 이성질체 다섯 개로 줄었고 2차 조각으로 최종 구조(CID 137835447)를 골랐다. 검증된 구조와 탈락한 비요소 이성질체 둘이 같은 97.7%를 설명했기 때문에 조각 경로를 따지지 않고는 구분할 수 없었다고 원문은 적는다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The verified structure and both rejected non-urea isomers accounted for the same 97.7% of summed fragment intensity, so distinguishing them required testing their fragmentation pathways."</blockquote><blockquote>"Spectral coverage measures how much peak intensity a candidate matches by mass alone, without testing whether its functional groups and bond connectivity can plausibly form those fragments."</blockquote></details>

## 쓰는 방법과 한계

원문이 적은 접근 경로는 다음과 같다.

- Antigravity 앱을 내려받고 Mass Spec 플러그인을 설치한다.
- 엔터프라이즈 사용자는 앱에서 "Use Business Account"를 고르고 Google Cloud 자격 증명으로 로그인한다.
- 과학자 10,000명에게 Antigravity 최고 사용 등급을 6개월 동안 무료로 제공하는 프로그램의 신청을 받고 있다. 자격, 수용량, 선발 조건이 붙고 순차적으로 등록된다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The Mass Spec Skills are available on Google Antigravity today. Download the Antigravity App and install the Mass Spec plugin to start using these skills."</blockquote></details>

스킬의 라이선스, 플러그인 소스 공개 여부, 가격은 원문에 없다. 설치 절차를 이 글에서 직접 실행해 보지는 않았다.

원문이 밝힌 한계는 둘이다. 거울상 분자와 일부 가까운 이성질체는 LC-MS/MS 스펙트럼만으로는 구분할 수 없다. 거의 같은 방식으로 쪼개지기 때문이고 크로마토그래피 머무름 시간, 이온 이동도, NMR 같은 다른 측정이 필요하다. 그리고 어떤 데이터베이스에도 없는 새 분자는 구조를 가져오거나 확정할 수 없다. 이 경우에도 예측 분자식, 피크 대응, 주석 표는 출발점으로 준다.

## 에이전트 환경에 가져올 것

질량분석을 하지 않아도 이 실험에서 가져올 판단 기준은 분명하다.

1. **에이전트가 같은 종류의 스크립트를 매번 새로 짜고 있다면 스킬로 굳힌다.** 원문에서 인터넷만 준 구성이 놓친 이유는 즉석 스크립트가 입력의 가장자리 조건에서 깨졌기 때문이다. 검증된 도구 호출을 스킬로 만들어 두면 그 실패가 사라진다.
2. **모델을 올리기 전에 도구를 먼저 본다.** 스킬을 붙인 뒤에는 경량 모델과 프런티어 모델의 정확도 차이가 없어졌다. 인터넷만 준 구성에서는 차이가 컸다. 과제가 도구 호출과 결과 조합으로 풀리는 종류라면 스킬 정비가 모델 교체보다 싼 선택일 수 있다.
3. **도구 점수가 같을 때의 판별 단계를 따로 둔다.** 계산 도구가 내는 점수는 한 가지 기준만 잰다. 원문의 에이전트는 점수가 같은 후보를 다른 근거로 교차 확인했고 그 근거를 보고서에 남겼다.
4. **결과와 함께 근거를 내게 한다.** 원문은 에이전트가 증거를 펼쳐 놓고 과학적 판단이 필요한 결정은 화학자가 한다는 구도를 유지한다.

단, 이 수치는 답이 공개 라이브러리에 있는 100개 화합물에서 Google이 자기 제품으로 잰 값이다. 라이브러리에 없는 분자에 대한 정량 평가는 원문에 없다. Gemini 4 Argon의 가격과 접근 경로는 [Gemini 4 Argon — 에이전트 운영자가 볼 가격·접근 경로·실측 수치]({{site.baseurl}}/dev/2026/10/20/gemini-4-argon-agent.html)에서 다뤘다.

## 참고

- [From spectrum to structure: Identifying molecules with Mass Spec Skills for Google Antigravity](https://deepmind.google/blog/interpreting-mass-spectra-with-google-antigravity/){:target="_blank"} (Google DeepMind, 2026-11-25)
- [Science on Google Antigravity](https://antigravity.google/use-cases/science){:target="_blank"} (원문이 링크한 Antigravity 과학 활용 페이지)
- [matchms](https://github.com/matchms/matchms){:target="_blank"}
- [SIRIUS](https://bio.informatik.uni-jena.de/software/sirius/){:target="_blank"}
- [MAGMa](https://github.com/NLeSC/MAGMa){:target="_blank"}
- [MetFrag](https://ipb-halle.github.io/MetFrag/){:target="_blank"}
- [MoNA](https://mona.fiehnlab.ucdavis.edu/){:target="_blank"}
- [PubChem](https://pubchem.ncbi.nlm.nih.gov/){:target="_blank"}
- [RDKit](https://www.rdkit.org/){:target="_blank"}
