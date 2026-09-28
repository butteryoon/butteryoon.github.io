---
layout: post
comments: true
title: "OpenAI MentalHealthBench: 정신건강 전문가 80여 명과 만든 AI 대화 평가 벤치마크"
description: "OpenAI가 22개국·19개 언어의 면허 정신건강 전문가 80여 명과 함께 만든 공개 벤치마크 MentalHealthBench를 발표했다. 응급 상황에 쏠렸던 기존 평가와 달리 일상 대화부터 응급까지 전 범위를 다루고, 전문가 합의 루브릭으로 채점한다."
img: mentalhealthbench_title.webp
date: 2026-09-28 20:25:00 +0900
last_modified_at: 2026-09-28 20:25:00 +0900
tags: [openai, mentalhealthbench, llm-eval, benchmark, ai-safety, mental-health, llm]
related: llm-eval
categories: [openai-research, llm-eval, ai-safety]
source_url: https://openai.com/index/introducing-mentalhealthbench/
source_date: 2026-09-23
---

OpenAI가 9월 23일 공개한 [「Introducing MentalHealthBench」](https://openai.com/index/introducing-mentalhealthbench/){:target="_blank"}를 읽고 정리했다. ChatGPT를 매주 10억 명 넘게 쓰는 상황에서, 모델이 정신건강 대화에 어떻게 답하는지 응급 상황 밖까지 넓혀 재 보자는 벤치마크다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** MentalHealthBench는 22개국의 면허 정신건강 전문가 80여 명과 함께 만든 공개 벤치마크다. 안전성, 맥락 탐색, 사용자 자율성 존중, 적절한 경우의 실행 가능한 조언 같은 핵심 행동을 평가한다. 대화마다 전문가가 -10~+10점 가중치의 루브릭을 쓰고, 전문가 2명 이상이 동의한 기준만 남긴 뒤 GPT‑5.6 Sol이 자동 채점한다. 전체 점수는 10개 행동 차원으로 분해된다. 별도 분석에서 실사용자 44명은 실질적인 다음 단계와 톤을, 전문가는 맥락 수집과 모호한 상황 해석을 더 중시했다.

## 1. 배경: 응급 상황에 쏠린 기존 평가

사람들이 AI와 나누는 대화는 어려운 관계 풀기, 일상 스트레스 다루기, 아끼는 사람 돕기처럼 폭이 넓다. 그런데 이 분야 평가는 대부분 안전과 직결된 응급 상황에 집중했고, 성공 여부도 미리 정한 넓은 기준으로 쟀다. 모델이 금지된 응답을 피하는지를 넘어, 상황마다 전문가 지침에 얼마나 맞게 답하는지는 알기 어려웠다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"This has left a gap in understanding how models perform across the full range of mental health conversations, and how well their responses align with expert guidance for each situation, beyond whether they avoid disallowed responses."</blockquote></details>

OpenAI는 프라이버시 보존 기법으로 실제 사용 패턴을 반영한 합성 대화를 만들었다. 일부 시나리오에는 최근 가족을 잃었다는 식의 배경 정보를 넣어, 모델이 그 맥락을 응답에 반영하는지도 본다. 대화는 성인·청소년·보호자·임상가 네 부류 사용자를 담고, 긴급도는 세 단계로 나뉜다.

- **비급성(Non-acute)**: 감정적 요소가 조금 섞인 일상 대화
- **고급성(High-acuity)**: 심각한 우려나 큰 고통이 드러나지만 당장의 응급은 아닌 대화
- **응급(Emergencies)**: 정신건강 응급이나 즉각적 안전 우려가 보여 현실의 도움이 시급한 대화

## 2. 벤치마크 설계: 전문가 합의 루브릭

HealthBench 작업을 이어받아, 22개국에서 19개 언어를 쓰고 20개 가까운 세부 전공을 대표하는 면허 심리학자·정신과 의사 80여 명과 함께 만들었다.

- **루브릭 작성**: 전문가가 합성 대화를 읽고 마지막 사용자 메시지에 대한 응답을 평가할 기준 목록을 쓴다. 기준 하나는 "적절한 질문을 하는가" 같은 한 측면만 겨냥한다.
- **가중치**: 기준마다 -10~+10점. 이로운 행동에는 가점, 해로운 행동에는 감점을 주고, 임상적으로 중요할수록 절댓값이 크다.
- **합의 필터**: 대화마다 전문가 3명 이상이 검토하고, 2명 이상이 동의하면서 세 번째 전문가가 반박하지 않은 기준만 남긴다.
- **자동 채점**: GPT‑5.6 Sol이 전문가 루브릭에 따라 모델 응답을 채점한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Each conversation was reviewed by at least three experts, and we only retained criteria agreed upon by at least two experts and not contradicted by a third."</blockquote></details>

청소년 페르소나는 시스템 메시지로 사용자가 13~17세임을 명시했고, 해당 대화는 청소년 정신건강 전문 임상가가 검토했다. OpenAI는 이 방식이 여러 모델 제공사에 공통으로 적용할 수 있는 대신, 개별 제품에 내장된 보호 장치까지 다 반영하지는 못할 수 있다고 적었다.

## 3. 10개 행동 차원으로 쪼개 보기

전체 점수는 전문가가 정의한 10개 행동 차원의 성적으로 분해할 수 있다. 전체 점수가 비슷한 모델도 차원별 강점은 다를 수 있어서, 어디를 고쳐야 할지 해석 가능한 단위로 짚을 수 있다. 원문은 한 예로, 모델이 발전할수록 맥락을 적절히 탐색하는 능력이 좋아졌다고 밝혔다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The overall score can be decomposed into performance along ten dimensions of model behavior for mental health. These behaviors are defined by mental health experts and aim to capture nuances in model performance. Models with similar overall scores can have different strengths across these behaviors."</blockquote></details>

## 4. 전문가와 사용자는 무엇을 다르게 보나

OpenAI는 벤치마크와 별개로, 전문가가 높이 평가한 응답이 사용자에게는 차갑거나 쓸모없게 느껴질 수 있는지 확인하는 분석을 했다. AI를 정신건강·정서 지원에 써 본 성인 44명(16개국, 14개 언어)이 합성 대화의 모델 응답을 평가하고 좋은 지원의 기준을 직접 썼다. 고통스러운 내용에 노출되지 않도록 비급성 대화만 다뤘다.

- **사용자가 더 중시한 것**: 실질적인 다음 단계, 톤
- **전문가가 더 중시한 것**: 관련 맥락 수집, 모호한 상황의 신중한 해석

<details class="evidence"><summary>원문 근거</summary><blockquote>"User perspectives highlighted qualities people value in AI support that were less emphasized in expert guidance, particularly practical next steps and tone. Experts placed greater emphasis on gathering relevant context and carefully interpreting ambiguous situations."</blockquote></details>

벤치마크의 최종 채점 기준은 전문가 합의를 따르며, 이 분석 때문에 바뀌지는 않았다.

## 5. 공개와 후속 작업

OpenAI는 벤치마크를 공개해 다른 연구자가 방법을 검토하고 직접 평가를 돌려 확장할 수 있게 했다. 세부 채점 과정과 평가 설정은 [논문](https://cdn.openai.com/ctf-cdn/MentalHealthBench_A_Comprehensive_Benchmark_of_AI_Capabilities_in_Realistic_Mental_Health_Conversations.pdf){:target="_blank"}에 있다. 이와 함께 [신규 연구 보조금](https://openai.com/index/ai-mental-health-research-grants/){:target="_blank"}, Partnership on AI와의 전문가 모임, Transluce의 [독립 정신건강 평가](https://behaviors.transluce.org/mental-health){:target="_blank"} 같은 활동도 지원한다. 제품 쪽에서는 민감한 대화에서 ChatGPT의 응답을 강화하고 위기 지원 자원 접근을 넓혔으며, [Trusted Contact](https://openai.com/index/introducing-trusted-contact-in-chatgpt/){:target="_blank"}와 [ChatGPT for Teens](https://openai.com/index/chatgpt-for-teens/){:target="_blank"}를 내놓았다.

원문 스스로도 어떤 벤치마크든 개인적인 대화에서 중요한 것을 전부 담지는 못한다고 인정한다.

## 6. 시사점

- **"회피"에서 "응답 품질"로**: 금지 응답을 피했는지만 보던 평가가, 상황마다 전문가가 기대하는 행동을 얼마나 해냈는지 재는 쪽으로 옮겨 가고 있다. 도메인 전문가 루브릭과 LLM 자동 채점을 묶는 구성은 다른 도메인 평가셋을 설계할 때도 참고할 만하다.
- **합의 필터링**: 3명 검토, 2명 동의, 반박 없음이라는 규칙은 전문가 사이에서도 갈리는 판단을 루브릭에서 걸러 내는 간단한 장치다.
- **청소년 페르소나 분리**: 시스템 메시지로 연령을 명시하고 청소년 전문 임상가가 따로 검토하는 방식은 연령별 가드레일을 점검하는 틀이 된다.
- **전문가-사용자 간극**: 전문가 점수가 높아도 사용자 만족과 어긋날 수 있다. 이 간극을 따로 측정해 두면 평가 기준을 조정할 근거가 생긴다.

## 7. 참고 자료

- [원문: Introducing MentalHealthBench](https://openai.com/index/introducing-mentalhealthbench/){:target="_blank"}
- [논문 PDF: MentalHealthBench](https://cdn.openai.com/ctf-cdn/MentalHealthBench_A_Comprehensive_Benchmark_of_AI_Capabilities_in_Realistic_Mental_Health_Conversations.pdf){:target="_blank"}
- [HealthBench](https://openai.com/index/healthbench/){:target="_blank"}
- [Transluce — Mental health evaluation](https://behaviors.transluce.org/mental-health){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/rG5elqddGzo){:target="_blank"}
