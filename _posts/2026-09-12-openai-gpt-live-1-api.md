---
layout: post
comments: true
title: "OpenAI GPT-Live-1 API 공개 — 듣고 말하기를 한 모델이 맡는 전이중 음성 에이전트"
description: "GPT-Live-1이 API로 공개됐다. STT→LLM→TTS 파이프라인 대신 단일 모델이 듣기와 말하기를 동시에 처리하고, 복잡한 추론은 GPT-6 Astra 같은 백엔드 모델에 위임한다. 6대 특징, 벤치마크 수치, 분당 $0.05 요금과 적용 시 따져볼 점을 정리한다."
img: gpt_live_1_title.webp
date: 2026-09-12 20:20:00 +0900
last_modified_at: 2026-09-12 20:20:00 +0900
tags: [openai, gpt-live, voice-ai, full-duplex, api, llm]
related: llm
categories: dev
---
OpenAI가 9월 10일 [GPT-Live-1을 API로 공개했다](https://openai.com/index/introducing-gpt-live-1-in-the-api/){:target="_blank"}. ChatGPT에 먼저 들어갔던 전이중(full-duplex) 음성 모델을 개발자가 직접 쓸 수 있게 된 것이다. 기존 음성 에이전트는 음성 인식, 추론 모델, 음성 합성을 이어 붙인 구조라 단계마다 지연이 쌓이고 말이 끊기면 흐름이 무너지곤 했다. GPT-Live-1은 듣기와 말하기를 한 모델이 동시에 맡고 깊은 추론과 도구 호출은 GPT-6 Astra 같은 백엔드 모델에 넘기는 구조를 택했다. 이 글에서는 원문 발표를 기준으로 특징, 벤치마크, 요금을 정리하고 실제 도입 시 따져볼 점을 짚는다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** GPT-Live-1은 음성 계층만 담당하는 프론트엔드 모델이다. 인터럽션 처리, 톤·속도 제어, 배경소음 대응, 장시간 세션 안정성, 전화 통화 지원을 기본으로 갖췄고 API에서 분당 $0.05에 바로 쓸 수 있다. Full Duplex Bench에서 GPT-Realtime-2.1보다 30%p 높은 점수를 냈고, GPT-6 Astra(medium)와 묶으면 Tau3 1위다. 백엔드 모델은 개발자가 고르므로 작업 난이도에 따라 비용과 지연을 조절할 수 있다.

## 1. 무엇이 달라졌나

핵심은 **듣기와 말하기를 단일 모델이 동시에 처리**한다는 점이다. 기존 실시간 음성 API(GPT-Realtime-2.1)를 포함한 체인형 구조에서는 사용자가 말을 끊었을 때 각 단계의 핸드오프가 지연과 오동작의 원인이 됐다. GPT-Live-1은 들어오는 오디오와 나가는 오디오를 함께 추론하므로, 사용자가 중간에 끼어들거나 "응", "그래" 같은 짧은 맞장구를 쳐도 그 자리에서 반응한다. 그 사이 복잡한 추론은 백엔드에서 계속 돌아간다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"GPT‑Live‑1 handles listening and speaking in a single model, simplifying the voice layer. It can respond to interruptions and acknowledgements as they happen, while delegating deeper reasoning to the back end. This lets the conversation continue while work happens in the background."</blockquote></details>

원문이 꼽은 여섯 가지 강점은 다음과 같다.

- **인터럽션 처리**: 입력·출력 오디오를 한 모델이 함께 추론해 체인형 STT–LLM–TTS 구조의 지연과 불안정한 핸드오프를 피한다.
- **추론·도구 호출 위임**: GPT-6 Astra나 서드파티 텍스트 모델에 추론과 도구 호출을 넘긴다.
- **톤·속도·스타일 제어**: 시스템 프롬프트로 에이전트의 말투, 속도, 대화 스타일을 지정한다.
- **무음·배경소음 처리**: 배경소음이나 무음 구간에 대화를 끊거나 모든 단계를 소리 내어 설명하지 않는다.
- **장시간 세션 안정성**: 긴 대화에서 컨텍스트 유지와 대화 품질을 개선했다.
- **전화 통화 지원**: 식당 예약부터 고객지원까지 전화용 전이중 음성 에이전트를 배포할 수 있다.

이 밖에 ASR 전사와 응답 텍스트를 기본으로 제공하고, 영숫자 인식과 키워드 바이어싱을 지원한다. 턴 기반 모델은 아니지만 턴 감지도 지원하므로, 명시적인 턴 경계를 전제로 짠 기존 애플리케이션도 옮겨 올 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"GPT‑Live‑1 natively provides ASR transcripts and response text. It also offers strong alphanumeric understanding and supports keyword biasing. Although GPT‑Live‑1 is not a turn-based model, it natively supports turn detection, so developers can continue to build around explicit turn boundaries."</blockquote></details>

## 2. 프론트엔드와 백엔드를 나눈 구조

GPT-Live-1은 **음성 프론트엔드** 역할만 맡는다. 뒤에 어떤 모델과 도구, 에이전트 하네스를 붙일지는 개발자가 정한다. 원문은 일정 조정이나 주문 상태 확인처럼 물량이 많은 작업에는 Luna 같은 모델을, 추론이 필요한 복잡한 고객 문제에는 Astra 같은 모델을 짝지우는 예를 든다. 작업마다 추론 깊이, 속도, 비용을 따로 맞출 수 있다는 뜻이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Developers choose the models, tools, and agent harness behind the conversation. For example, they might pair GPT‑Live‑1 with a model like Luna for high-volume tasks like scheduling or order updates, and use a model like Astra for complex customer issues that require reasoning. That flexibility lets developers match reasoning depth, speed, and cost to each task."</blockquote></details>

원문에는 Codex SDK와 연결하는 코드 발췌도 실려 있다. 애플리케이션이 대화 맥락을 Codex 스레드에 넘겨 답을 받은 뒤 `session.commentary.append` 이벤트로 GPT-Live-1에 돌려주는 흐름이다. 음성 모델이 직접 저장소를 뒤지는 게 아니라 답을 만들 주체와 말할 주체가 분리돼 있음을 보여주는 예시다.

이 구조가 실제 코드 부담을 얼마나 줄이는지는 고객 인용에서 엿볼 수 있다. 한 헬스케어 스타트업 CTO는 체인형 구조와 비교해 코드베이스가 80% 줄고 2만 3천 줄이 사라졌다고 했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Compared to our cascaded build, GPT‑Live‑1 simplified our code base by 80% and removed 23K lines of code. This enabled natural, real-time patient conversations & freed our team to improve the experience from booking an appointment to navigating care."</blockquote></details>

## 3. 벤치마크와 고객 사례

원문이 공개한 수치는 세 가지다.

- **Full Duplex Bench**: GPT-Realtime-2.1 대비 30%p 향상. 턴 교대 지연과 상호작용 행동에서 큰 폭으로 개선됐다.
- **Tau3**: GPT-6 Astra(medium reasoning effort)와 조합했을 때 1위. 엔드투엔드 작업에서 음성 에이전트 지능을 재는 벤치마크다.
- **Speak 초기 평가**: 이전 턴 기반 시스템 대비 인터럽션이 거의 80% 줄었다. 학습자가 생각하는 동안 튜터가 끼어들지 않게 됐다는 뜻이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Across our evaluations, GPT‑Live‑1 improves Full Duplex Bench performance by 30 percentage points over GPT‑Realtime‑2.1, with large gains in turn-taking latency and interactive behavior. Paired with GPT‑6 Astra at medium reasoning effort, it also ranks #1 on Tau3, which measures frontier voice-agent intelligence on end-to-end tasks."</blockquote></details>

<details class="evidence"><summary>원문 근거</summary><blockquote>"in early evaluations, Speak found that GPT‑Live‑1 gave learners more time to think before the language tutor responded, cutting interruptions by almost 80% versus previous turn-based systems."</blockquote></details>

벤치마크 차트 각주를 보면 항목마다 백엔드가 다르다. 고객지원 과제(항공·소매·통신)와 은행 지원 과제는 Astra(medium)를, 도구 호출 과제는 Terra(low)를 백엔드로 썼다. 수치를 비교할 때 이 조건을 함께 봐야 한다.

고객 사례로는 Yelp(전화 예약·주문 응대의 처리율 개선), Speak(언어 튜터), Fin(고객지원), Cognition(Devin과 음성으로 협업)이 실렸다. 원문 페이지에는 Yelp Host가 배경소음과 옆 사람 대화, 인터럽션을 처리하며 예약을 완료하는 오디오 데모도 있다.

## 4. 음성 옵션 확장

실시간 API가 제공하던 소수의 음성에서 벗어나 억양·방언·언어를 아우르는 음성 선택지를 넓혔다. 발표 시점에 Quartz, Ripple, Vesper, Willow 등 12종의 새 음성이 공개됐고 앞으로 몇 달에 걸쳐 음성과 지원 언어를 계속 늘리겠다고 했다. 커스텀 음성은 별도로 영업팀에 자격과 신청 절차를 문의해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"With GPT‑Live‑1, we're expanding from a small set of real-time voices to a broader selection across accents, dialects, and languages giving developers more choice in how their assistants sound."</blockquote></details>

## 5. 요금과 제공 경로

- **GPT-Live-1(프론트엔드 음성 계층)**: 분당 $0.05. API에서 바로 쓸 수 있다.
- **백엔드 모델**: 별도 과금. GPT-6 Astra를 붙이면 [앞서 정리한 대로]({{site.baseurl}}/dev/2026/09/09/openai-gpt6-astra.html) 입력 $10 / 출력 $50 per 1M 토큰이 추가된다.
- **OpenAI Presence**: GPT-Live-1을 실시간 음성에 쓰는 기업용 에이전트 플랫폼이다. 질문 응대, 문제 해결, 사내 시스템 사용, 승인된 조치 실행, 사람에게 에스컬레이션까지 다루며 OpenAI 계정 담당자를 통해 문의한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"GPT‑Live‑1 is available in the API today at $0.05 per minute for the front-end voice layer. Pair it with the backend model and agent harness that fit your product, then build a voice experience that can scale with the work it needs to do."</blockquote></details>

## 6. 도입 전에 따져볼 점

**구조 전환의 의미.** 음성 이해·생성과 사고·도구 사용을 분리한 설계라, 백엔드에 자체 파인튜닝 모델이나 온프레미스 LLM을 붙이는 구성도 원리상 가능하다. 원문은 "서드파티 모델"에 위임할 수 있다고만 밝혔으므로, 어떤 연결 방식이 지원되는지는 [Live 가이드 문서](https://developers.openai.com/api/docs/guides/live){:target="_blank"}에서 확인해야 한다.

**비용은 분 단위로 선형 증가한다.** 분당 과금이라 통화량이 많은 서비스일수록 세션 길이 관리와 백엔드 모델 선택이 비용을 좌우한다. 단순 문의는 가벼운 모델로, 복잡한 상담만 Astra로 보내는 식의 분기가 필요하다.

**한국어 품질은 아직 미지수다.** 원문은 언어 지원을 "앞으로 몇 달에 걸쳐 확장"한다고만 밝혔다. 한국어 음성 품질과 억양 지원 범위는 직접 세션을 열어 확인하는 편이 안전하다.

**기존 턴 기반 코드와의 호환.** 턴 감지를 기본 지원하므로 기존 실시간 API 위에 짠 로직을 당장 버릴 필요는 없다. 다만 전이중의 장점은 턴 경계에 기대지 않을 때 나오므로, 인터럽션과 맞장구를 어떻게 다룰지 대화 설계를 다시 보는 게 좋다.

## 참고

- [Build more natural voice experiences with GPT‑Live‑1 in the API](https://openai.com/index/introducing-gpt-live-1-in-the-api/){:target="_blank"} — OpenAI, 2026-09-10
- [Live 가이드 문서](https://developers.openai.com/api/docs/guides/live){:target="_blank"}
- [OpenAI Presence 소개](https://openai.com/index/introducing-openai-presence/){:target="_blank"}
- [OpenAI GPT-6 Astra 발표 정리]({{site.baseurl}}/dev/2026/09/09/openai-gpt6-astra.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/BeVGrXEktIk){:target="_blank"}
