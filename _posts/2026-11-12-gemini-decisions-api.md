---
layout: post
comments: true
title: "Gemini Decisions API: 멀티모달 결정 모델을 API로 바로 쓰기"
description: "Google이 결정 전용 모델 Gemini Decision 1과 Decisions API를 공개했다. yes/no·다중선택·루브릭 채점을 한 번의 포워드 패스로 처리하고, 텍스트·이미지·오디오·비디오·PDF를 한 입력에 섞어 넣을 수 있다. 엔드투엔드 150-250ms, 입력 1M 토큰당 $0.10, 출력 토큰 무료."
img: gemini_decisions_api_title.webp
date: 2026-11-12 19:00:00 +0900
last_modified_at: 2026-11-12 20:30:00 +0900
tags: [llm, llm-serving, google, gemini, decisions-api, decision-model, multimodal, api, serving-stack]
related: serving-stack
categories: [dev]
source_url: https://blog.google/innovation-and-ai/technology/developers-tools/gemini-decisions-api/
source_date: 2026-11-11
---
Google이 **Gemini Decisions API**를 내놨다. 도구 선택, 정책 검사, 모델 라우팅처럼 답이 정해진 작업을 위한 엔드포인트로, 새 결정 전용 모델 Gemini Decision 1(`gemini-decision-1`)이 뒤에서 돈다. 분류기를 따로 학습시킬 필요 없이 질문과 선택지만 정의하면 yes/no·다중선택·루브릭 채점 결과가 확률과 함께 돌아온다. 입력은 텍스트·이미지·오디오·비디오·PDF를 섞어 넣어도 되고 한 요청에 질문을 1,000개까지 붙일 수 있다. 값은 입력 1M 토큰당 $0.10이고 출력 토큰은 받지 않는다. Google AI Studio에서 API 키를 받으면 바로 써 볼 수 있다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:**
> - 결정 전용 모델 **Gemini Decision 1**(`gemini-decision-1`)과 Decisions API 공개. 질문 타입은 `noul`(yes/no), `choice`(다중선택), `score`(순서형 루브릭) 세 가지
> - **멀티모달**: 텍스트·이미지·오디오·비디오·PDF를 한 입력에 섞어 최대 256K 토큰까지 넣는다
> - **단일 패스 채점**: 답을 토큰 단위로 생성하지 않고 후보를 한 번의 포워드 패스로 채점한다. 엔드투엔드 150-250ms, 질문이 1개에서 100개로 늘어도 지연이 거의 그대로다
> - **가격**: 입력 1M 토큰당 $0.10, 출력 $0. 공통 벤치마크 기준 Gemini 3.5 Flash-Lite + Interactions API보다 5배 가까이 싸다
> - **바로 사용**: Google AI Studio에서 API 키 발급 후 `google-genai` SDK나 REST로 호출. OpenRouter·LangChain·LiteLLM 등 생태계 통합은 순차 적용 중

## 무엇이 다른가: 분류기 대신 선택지 정의

에이전트 라우팅, 정책 검사, 컨텍스트 등급 매기기는 모두 답이 몇 개로 닫혀 있는 작업이다. 지금까지는 작은 분류기를 파인튜닝해 붙이거나, 범용 LLM에 긴 프롬프트를 주고 돌아온 JSON을 파싱했다. 앞쪽은 학습 파이프라인을 따로 돌려야 하고 뒤쪽은 생성 지연과 파싱 실패를 감수해야 한다.

Decisions API에서는 질문과 선택지만 정의한다. 모델이 후보를 채점해 확률과 함께 돌려준다. Simon Willison은 이 API가 [Jev](https://simonwillison.net/2026/Sep/21/jev/){:target="_blank"}와 **거의 똑같다**고 짚었다. 지원하는 질문 타입 세 가지(yes/no, choice, rubric 채점)가 같고 yes/no 질문을 부르는 Jev의 용어 'Noul'까지 그대로 가져왔다. Google은 이로써 Jev, [OpenAI](https://simonwillison.net/2026/Oct/6/llm-openai-decisions/){:target="_blank"}에 이어 결정 모델 시장에 세 번째로 들어왔다.

차이는 입력 모달리티다. Jev는 텍스트(와 JSON)만, OpenAI는 텍스트와 이미지를 받는다. Gemini Decision 1은 텍스트·이미지·오디오·비디오·PDF를 모두 받는다. 통화 녹음에 계정 메타데이터를 붙이거나, 스캔한 PDF 청구서에 파손 사진을 함께 넣는 식이다. 이미지·비디오는 해상도를 조절해 시각 정보량과 토큰 비용을 맞출 수 있다.

## 세 가지 질문 타입

한 요청 안에 세 타입을 섞어 보낼 수 있다. 모든 답에는 0~1 사이 신뢰도가 붙어서 애매한 케이스를 폴백이나 사람 검토로 넘기는 분기를 짜기 쉽다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Every answer includes a confidence score (from 0 to 1) so your code can route ambiguous cases to a fallback or human reviewer:"</blockquote></details>

| 타입 | 키 | 용도 | 반환값 |
|------|-----|------|--------|
| Yes/No | `noul` | 조건이 성립할 확률 추정 (참·거짓 설명 기준은 선택) | 확률 |
| 다중선택 | `choice` | 2~255개 선택지 중 하나 고르기 | 승자 + 전 후보의 정규화 확률 |
| 순서형 루브릭 | `score` | 0부터 시작하는 순서 등급으로 채점 | 연속 가중 점수 + 등급별 확률 |

원문 예제는 휴게실 카운터 사진과 메모 한 줄을 세 타입으로 한꺼번에 검사한다.

```python
from google import genai

client = genai.Client(http_options={"api_version": "v1beta"})

result = client.decisions.create(
    model="gemini-decision-1",
    input=[{
            "type": "image",
            "uri": "https://storage.googleapis.com/generativeai-downloads/images/spill.jpg",
            "mime_type": "image/jpeg",
        },
        {
            "type": "text",
            "text": "Breakroom counter check before the 9 AM client meeting.",
        },
    ],
    questions={
        "has_spill": {
            "type": "noul",
            "instructions": "Is there an uncleaned liquid spill on the counter?",
        },
        "next_action": {
            "type": "choice",
            "instructions": "Which cleanup action fits the scene?",
            "criteria": {
                "wipe_with_cloth": "Small counter spill with a cloth or towel already nearby.",
                "dispatch_janitorial": "Large floor hazard, broken glass, or active leak.",
                "ready_for_meeting": "Counter is clean and ready.",
            },
        },
        "severity": {
            "type": "score",
            "instructions": "Rate the severity of the counter mess.",
            "criteria": [
                "Clean and ready.",
                "Minor localized spill.",
                "Major spill or safety hazard.",
            ],
        },
    },
)
```

`input` 필드는 Interactions API와 같은 `Content` 블록 형식이다. 과금 구조상 공통 입력은 질문 수와 상관없이 요청당 한 번만 청구되고 질문마다 붙는 건 그 질문의 instruction·criteria 토큰뿐이다. 같은 입력에 물을 게 여럿이면 요청 하나에 몰아 보내는 편이 싸다.

## 지연과 비용

원문이 밝힌 수치는 이렇다.

- **지연**: 모델 연산 기준 중앙값 50-100ms, 엔드투엔드 150-250ms. 컨텍스트 16K 토큰까지는 질문을 1개에서 100개로 늘려도 지연이 거의 늘지 않는다
- **교정(calibration)**: 에이전트 라우팅·추론·랭킹·멀티모달 이해 벤치마크에서 교정 오차가 가장 낮았다
- **비용**: 출력 토큰을 받지 않아서 공통 벤치마크 묶음 기준 Gemini 3.5 Flash-Lite + Interactions API보다 5배 가까이 싸다

<details class="evidence"><summary>원문 근거</summary><blockquote>"Gemini Decision 1 scores options without token-by-token decoding, keeping median latency 50-100 ms of model compute (150-250 ms end-to-end), across multimodal inputs and contexts up to 16K tokens, remaining nearly flat from 1 to 100 questions per request. With zero output tokens billed, Gemini Decision 1 with the Decisions API is nearly 5x cheaper across the shared benchmark suite than Gemini 3.5 Flash-Lite with the Interactions API."</blockquote></details>

가격은 입력 1M 토큰당 $0.10, 출력 $0이다. 사고(thinking) 토큰이나 오디오 전사·OCR 같은 별도 요금도 없다. Google AI Studio의 무료 티어로 프로토타이핑을 할 수 있다. 다른 결정 모델과 비교하면 OpenAI의 GPT-6 Luna 결정 모델과 같은 값(입력 1M당 10센트, 출력 무료)이고 셋 중 가장 싼 쪽은 여전히 Jev(1M당 4.2센트)다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"It's priced the same as the GPT-6 Luna decision model: 10 cents per million input tokens, and output is free. Jev remains the cheapest of the three, at 4.2 cents per million."</blockquote></details>

## Willison의 비디오 테스트: 확률이 1.0으로 붙지 않는다

Willison은 자기 영상(로봇이 걸어 다니는 영상)을 넣고 yes/no 질문 네 개를 던졌다. 용이 나오는지, 로봇이 나오는지, 배경이 프랑스인지, 미국인지. 결과는 로봇 0.923, 미국 0.923, 용 0.050, 프랑스 0.083이었다. 방향은 다 맞았는데, Jev와 OpenAI가 yes/no에 1.0이나 0.0을 돌려주는 것과 달리 Gemini는 확신을 조금 덜어 낸 값을 준다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Jev and OpenAI both return 1.0 or 0.0 for yes/no questions, but Gemini insists on being a little less confident than that:"</blockquote></details>

다른 결정 모델에서 옮겨 오는 경우라면 이 차이를 챙겨야 한다. `probability == 1.0` 같은 조건이나 다른 모델에 맞춰 둔 임계값은 그대로 쓰면 안 되고 자기 데이터로 다시 잡아야 한다.

## Google 내부 사용 사례

원문은 Google 안에서 이미 Decisions API를 저지연 제어 계층으로 쓰고 있다며 네 가지를 들었다.

1. **Gemini Notebook 모델 라우팅** — 들어온 쿼리의 의도를 밀리초 단위로 분류해 모델 티어와 사고 수준을 정한다. 일상 쿼리는 지연을 최대 40% 줄이고 깊은 추론 모델은 복잡한 작업에만 쓴다.
2. **Stitch rapid canvas** — 제품 흐름의 플레이어블 와이어프레임을 몇 초 만에 만들고 음성이나 탭으로 한 수정을 거의 즉시 반영한다. 흐름이 마음에 들면 한 번 클릭으로 화면을 정식 Stitch로 넘긴다.
3. **CC(Google Labs) 이모지 선택** — 200개가 넘는 이모지 팔레트에서 알맞은 반응을 1초도 안 걸려 골라 지연 없이 바로 띄운다.
4. **Google Search의 MCP 도구 안전 계층** — 적대적 행동과 콘텐츠를 지연 예산 안에서 높은 효율로 탐지한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Teams across Google are already using the Decisions API as a low-latency control plane, shipping features simply by defining options rather than training custom classifiers or waiting on generative calls."</blockquote></details>

에이전트 오케스트레이션에서 툴 라우팅, 가드레일, 컨텍스트 등급을 붙이려는 팀이라면 같은 방식을 그대로 가져다 쓸 수 있다.

## 시작하기

1. **API 키**: [Google AI Studio](https://aistudio.google.com/api-keys){:target="_blank"}에서 발급
2. **SDK**: 파이썬은 `pip install google-genai`. [Decisions API 문서](https://aistudio.google.com/docs/decisions-api){:target="_blank"}에 파이썬·자바스크립트·REST 가이드가 있다
3. **플레이그라운드**: [AI Studio의 Decisions API 플레이그라운드](https://aistudio.google.com/apps/bundled/gemini-decisions-api-playground?fullscreenApplet=true){:target="_blank"}
4. **데모 앱**: [Doorstep Parcels](https://aistudio.google.com/apps/bundled/doorstep-parcels-qa?fullscreenApplet=true){:target="_blank"}, [Speed Sketch](https://aistudio.google.com/apps/bundled/speed-sketch?fullscreenApplet=true){:target="_blank"}, [Find the Zoo Keeper](https://aistudio.google.com/apps/bundled/find-the-zoo-keeper?fullscreenApplet=true){:target="_blank"}
5. **생태계 통합**: OpenRouter, LangChain, Vercel, LiteLLM, Pydantic AI, Pipecat, LiveKit, OpenCode에 차례로 들어가는 중이다. [OpenRouter](https://openrouter.ai/google/gemini-decision-1){:target="_blank"}에는 `google/gemini-decision-1` 페이지가 열려 있다. Gemini in Google Cloud 지원은 곧 나온다고 했다([관심 등록 폼](https://forms.gle/NFt1W6pL1gCY9FGM8){:target="_blank"})

## 관련 글

이 블로그에서 다룬 결정 모델 글들이다.

- [생성 대신 채점 — SGLang /v1/score로 오픈 LLM을 분류 엔진으로 쓰기]({{site.baseurl}}/dev/2026/09/28/sglang-scoring-decision-engine.html)
- [OpenAI DevDay 2026 — Decisions API 언급]({{site.baseurl}}/dev/2026/10/02/openai-devday-2026.html)
- [결정 전용 모델이 오픈웨이트로 — autotrust/JEV-27B]({{site.baseurl}}/dev/2026/10/06/jev-27b-open-decision-model.html)
- [심판도 글을 써야 하나 — Jev로 에이전트 평가 만들기]({{site.baseurl}}/dev/2026/10/06/jev-judge-bounded-evaluation.html)
- [카카오 Kanana-2-v-decide — 멀티모달 의사결정 모델 오픈소스 공개]({{site.baseurl}}/kakao-tech/llm-model/2026/11/07/kanana-2-v-decide.html)

## 참고

- [Gemini Decisions API: Fast, calibrated multimodal decisions — Google Blog (2026-11-11)](https://blog.google/innovation-and-ai/technology/developers-tools/gemini-decisions-api/){:target="_blank"}
- [Gemini Decisions API — Simon Willison's Weblog (2026-11-11)](https://simonwillison.net/2026/Nov/11/gemini-decisions-api/){:target="_blank"}
- [Decisions API 문서 — Google AI Studio](https://aistudio.google.com/docs/decisions-api){:target="_blank"}
- [google-genai — PyPI](https://pypi.org/project/google-genai/){:target="_blank"}
