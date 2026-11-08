---
layout: post
comments: true
title: "카카오, AI 에이전트용 멀티모달 의사결정 모델 'Kanana-2-v-decide' 오픈소스 공개"
description: 문장 대신 확률·선택·점수로 판단을 돌려주는 3B 멀티모달 Decision 모델. 임베딩 기반이라 선택지 수 제한이 없고, H200 1장에서 판단 1건 약 42ms, Jev 호환 API로 호출한다.
img: kanana_2_v_decide_title.webp
date: 2026-11-07 21:00:00 +0900
last_modified_at: 2026-11-08 22:20:00 +0900
tags: [llm, llm-agent, llm-model, kanana, kakao-tech, decision-model, open-source, multimodal]
related: llm-agent
categories: [kakao-tech, llm-model]
source_url: https://tech.kakao.com/posts/846
source_date: 2026-11-06
---
에이전트가 요청 하나를 처리하는 동안 내리는 판단은 생각보다 많다. 문의를 어느 부서로 보낼지, 어떤 도구를 부를지, 사진 속 상품이 검색어와 맞는지 같은 것들이다. 이런 판단마다 LLM에 문장을 생성시키면 몇 초씩 걸린다. 카카오 Applied AI Model팀이 공개한 **Kanana-2-v-decide**는 이 자리를 노린 3B 멀티모달 의사결정(Decision) 모델이다. 텍스트와 이미지를 함께 받아 문장 대신 확률·선택·점수를 돌려준다. 체크포인트는 Hugging Face에 올라와 있어 직접 돌려볼 수 있다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Kanana-2-v-decide는 카카오의 멀티모달 LLM `kanana-2-v-3b`를 기반으로 학습한 3B Decision 모델이다. Noul(참·거짓 확률), Choice(선택지와 확률), Score(단계 점수) 세 형식으로 답하고 한 요청에 여러 질문을 묶을 수 있다. 선택지를 임베딩으로 따로 계산하기 때문에 선택지 수 제한(d3는 255개)이 없고 순서 쏠림이 줄며 선택지 벡터를 캐시해 재사용한다. H200 1장 기준 판단 1건 약 42ms로 GPT-6 Luna보다 약 43배 빠르다. 모델은 `kakaocorp/kanana-2-v-decide-3b-2611`로 공개됐다. transformers로 바로 부르거나 동봉된 HTTP 서버를 띄워 typesafe-sdk 같은 Jev 호환 클라이언트로 호출할 수 있다.

## Decision 모델이란

Decision 모델은 서비스 동작에 필요한 판단을 자연어 설명이 아니라 **확률·선택·점수**로 돌려준다. ChatGPT 같은 LLM에 "이 문의는 어느 부서로 연결해야 될까요?"라고 물으면 몇 초 뒤 문장으로 답이 온다. Decision 모델은 글을 쓰지 않고 판단에 필요한 값을 바로 계산한다.

| 형식 | 묻는 내용 | 돌려주는 값 | 활용 예 |
| --- | --- | --- | --- |
| **Noul** (참·거짓 판단) | 참인가요, 거짓인가요? | '예'라고 답할 확률 (0~1) | 상품 사진과 검색어 일치 여부, 가이드라인 위반 여부 |
| **Choice** (선택지 고르기) | 어느 선택지가 맞나요? | 고른 선택지와 선택지별 확률 | 문의 유형 분류, 에이전트 도구(tool) 선택, 상품 카테고리 분류 |
| **Score** (점수 평가) | 어느 정도인가요? | 단계별 확률을 반영한 점수 | 리뷰 만족도, 이미지와 텍스트의 연관성 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"문장을 차례로 생성하는 대신, 주어진 내용에 대해 특정 조건이 참일 확률(Noul), 최적의 답과 선택지별 확률(Choice), 단계별 확률을 반영한 정밀한 평가 점수(Score) 형태의 점수로 반환합니다."</blockquote></details>

흐름은 지난 9월 미국 스타트업 TypeSafe AI가 Jev를 공개하면서 시작됐다. 그 뒤 한 달 사이 OpenAI, Perplexity, Microsoft, Liquid AI, vLLM Semantic Router가 잇달아 비슷한 모델을 내놨다. TypeSafe AI가 발표한 수치로는 LLM이 8.6초 걸리는 판단을 Jev는 0.11초에 끝낸다. 판단에 맞는 점수 형태로 곧장 출력하니 처리 시간과 연산 비용이 함께 줄어든다.

## Kanana-2-v-decide 모델 정보

Kanana-2-v-decide는 카카오의 자체 멀티모달 LLM `kanana-2-v-3b`를 기반으로 학습했다. 상황(텍스트·이미지)과 질문을 주면 Jev와 호환되는 요청 형식으로 세 가지 판단을 수행한다.

| 항목 | 내용 |
| --- | --- |
| 모델명 | Kanana-2-v-decide-3b (Hugging Face: `kakaocorp/kanana-2-v-decide-3b-2611`) |
| 기반 모델 | kanana-2-v-3b (카카오 멀티모달 LLM) |
| 입력 | 텍스트 + 이미지 (비디오는 미지원) |
| 판단 형식 | Noul · Choice · Score, 한 요청에 여러 질문 가능 |
| 응답 시간 | 판단 1건 약 42ms (H200 GPU 1장, 한국어 상품 리뷰 데이터 기준, 토큰 수에 따라 다름) |
| 호출 방식 | Jev와 호환되는 API |
| 라이선스 | Kanana Open License |

<details class="evidence"><summary>원문 근거</summary><blockquote>"H200 GPU 1장을 기준으로 응답 시간 약 42ms로, GPT-6 Luna 대비 약 43배 빠른 응답 속도를 기록했습니다."</blockquote></details>

한 요청에 질문을 여러 개 묶을 수 있다는 점이 실무에서 쓸모 있다. "상품이 깨져서 도착했어요"라는 문의와 사진이 들어오면 파손 여부(Noul), 문의 유형(Choice), 고객 감정(Score)을 한 번에 판단하게 하고 그 결과에 따라 에이전트가 상담 연결이나 교환 시스템으로 넘기는 식이다. 다만 42ms는 판단 1건 기준이다. 저장소 코드는 질문을 배치로 묶지 않고 하나씩 처리하므로, 한 요청에 질문을 N개 넣으면 질문 쪽 연산도 N번 일어난다.

## 임베딩 기반 구조: 선택지 개수와 순서 제약을 없앤 방법

Jev는 내부 구조가 공개되지 않았다. 구조를 공개한 Decision 모델 중 대표 격은 vLLM Semantic Router의 **d3**다. d3는 LLM에 head를 붙인 구조로, 입력과 질문, A·B·C 답 코드를 붙인 선택지 목록을 프롬프트 하나에 넣는다. LLM이 한 번 읽고 나면 head가 답 코드마다 점수를 낸다. 답 코드가 255개로 고정돼 있어 선택지도 255개까지만 된다. 선택지가 순서대로 프롬프트에 들어가니 순서가 결과를 흔들 수 있다.

Kanana-2-v-decide는 **선택지를 임베딩으로 표현해 비교**한다. 입력과 질문은 벡터 q로, 선택지는 하나씩 따로 벡터 a₁ … aₙ으로 만든 뒤 q와 각 선택지 벡터의 코사인 유사도를 재고 softmax로 선택지별 확률을 낸다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"입력과 질문을 벡터 q로, 선택지는 하나씩 따로 벡터 a₁ … aₙ으로 표현합니다. q와 각 선택지 벡터가 얼마나 가까운지(cos)를 재서 softmax로 선택지별 확률을 계산합니다."</blockquote></details>

이 구조에서 세 가지 장점이 따라 나온다.

| 특징 | 설명 |
| --- | --- |
| 1. 선택지 개수의 확장성 | 선택지 수에 제한이 없다. 벡터를 추가하면 되므로 ImageNet처럼 class가 1,000개인 분류도 한 번에 할 수 있다 |
| 2. 선택지 순서의 독립성 | 선택지마다 따로 벡터를 만들어 점수를 매기므로, 선택지 쪽에서는 앞쪽이나 특정 번호를 더 고르는 쏠림이 생기지 않는다 |
| 3. 임베딩 재사용 | 선택지 벡터를 한 번 계산해 저장해 두고 state가 바뀐 부분만 새로 계산한다. 고정된 분류 체계나 도구 목록으로 많은 요청을 처리할 때 유리하다 |

다만 "순서와 완전히 무관"은 아니다. 공개된 저장소 코드 기준으로 기본 설정(`show_criteria=true`)에서는 선택지 목록이 질문 프롬프트 안에도 `Options:` 블록으로 함께 들어간다. 선택지 벡터 쪽은 순서의 영향을 받지 않지만 질문 벡터 q는 영향을 받을 여지가 남는다. 선택지가 255개를 넘으면(`MAX_SHOWN_CHOICES = 255`) 질문 쪽에서는 목록을 자동으로 숨기고, 질문별로 숨기도록 지정할 수도 있다. 순서 민감도가 문제가 되는 분류라면 이 설정을 끄고 비교해 보는 편이 좋다.

구조상 이 모델은 문장을 만들지 못한다. 저장소 코드에 생성용 LM head가 없고, 마지막 토큰의 hidden state를 L2 정규화해 벡터로 쓰는 것이 전부다. Noul도 내부적으로는 Choice와 같은 방식으로 풀린다. 질문마다 `true: Yes. This is true: {질문}`, `false: No. This is false: {질문}` 두 문장을 자동으로 만들어 두고 질문 벡터가 어느 쪽에 더 가까운지 보는 식이다. 판단 전용 모델이라 대화나 요약으로 돌려 쓸 수는 없다.

여기부터는 해석이다. 질문과 선택지 사이에 cross-attention이 없는 bi-encoder 구조이므로, 기본값에서 질문 프롬프트에 선택지 목록을 같이 넣는 것은 이 한계를 보완하려는 설계로 읽힌다.

대가도 있다. 처음 보는 선택지는 최초 한 번 임베딩해야 한다. class가 1,000개인 분류를 처음 돌리면 그 호출에서 선택지 연산이 1,000번 일어난다는 뜻이다. 선택지 캐시는 최대 4,096개까지 LRU로 유지되고 키는 (dtype, device, 토큰 ID)이므로, 분류 체계가 그보다 크면 밀려난 선택지는 다시 계산된다. 원문은 실제 서비스에서는 선택지가 고정되고 사용자 입력만 바뀌는 경우가 많아 선택지 임베딩을 미리 계산해 두면 불필요한 연산을 줄일 수 있다고 설명한다.

| 항목 | Jev | d3 | Kanana-2-v-decide |
| --- | --- | --- | --- |
| 구조 | 비공개 · API로만 제공 | LLM + Answer-code head | 질문 벡터와 선택지 벡터를 비교 |
| 기반 모델 | 비공개 | Qwen3.5 (2B · 4B · 27B) | kanana-2-v-3b |
| Choice 선택지 수 | 최대 255개 | 최대 255개 (head의 행 수) | 제한 없음 |
| 선택지 순서 | 쏠림이 생길 수 있음 | 쏠림이 생길 수 있음 | 영향 최소화 |
| 처음 보는 선택지 | 프롬프트에 추가 | 프롬프트에 추가 | 벡터 캐싱 필요 |
| 같은 선택지 반복 | 비공개 | 질문마다 다시 읽음 | 저장한 벡터 재사용 |

## MOPD로 안전성 끌어올리기

Decision 모델의 판단은 서비스 로직으로 곧장 넘어간다. 그래서 해가 되는 쪽으로 기울지 않는 것도 중요하다. Safety 데이터로 학습한 모델을 기존 모델과 가중치 합(Weight Sum)으로 합치면 Safety 판단은 좋아졌지만 다른 판단 능력이 같이 떨어졌다.

카카오는 두 모델을 **MOPD(Multi-teacher On-Policy Distillation)**로 합쳤다. 학습 데이터의 행마다 따라 할 Teacher 모델을 고른다. Safety 데이터(전체 학습 데이터의 약 5%)에서는 Safety 모델을, 나머지에서는 학습 전의 자기 자신을 따른다. Teacher의 선택지별 확률을 학습 중에 구해 맞춘다. Safety 판단이 필요한 곳만 바뀌고 나머지는 원래 판단을 지키는 구조다. 원문은 이름은 On-Policy지만 Student가 만든 응답을 다시 학습에 쓰지 않으니 엄밀히는 Online에 가깝다고 덧붙였다.

| 기존 모델 대비 | Weight Sum | MOPD (Seed A / Seed B) |
| --- | --- | --- |
| **Decision Index** (텍스트 공개 벤치마크) | −0.77 | −0.22 / −0.47 |
| **Decision Index Vision** (멀티모달 공개 벤치마크) | −1.34 | −0.31 / −0.96 |
| **Kanana Decision bench** (멀티모달 한국어 자체 데이터셋) | +0.68 | +1.35 / +1.45 |
| **JevBench** (Decision 모델용 공개 판단 벤치마크) | −3.03 | +1.30 / +0.43 |
| **trolley: 불필요한 손해 선택** (기존 모델 286, 낮을수록 좋음) | 185 | 118 / 77 |
| **trolley: 판단 모순** (기존 모델 1343, 낮을수록 좋음) | 995 | 956 / 851 |

trolley 문제(누구를 구할지 고르는 딜레마) 형식의 안전 평가([trolley-bench](https://github.com/zachlandes/trolley-bench){:target="_blank"})에서 불필요하게 손해를 끼치는 선택은 최대 73%, 비슷한 상황에서 답이 뒤바뀌는 모순은 최대 37% 줄었다. 두 번 학습한 모델의 가중치를 평균하자 Decision Index 손실이 사라졌고(+0.03) 안전 개선은 그대로 남았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"특히 trolley 문제(누구를 구할지 고르는 딜레마) 형식의 안전 평가에서는 불필요하게 손해를 끼치는 선택이 최대 73%(286 → 77), 비슷한 상황에서 답이 뒤바뀌는 모순이 최대 37%(1,343 → 851) 감소해, 안전 교사인 가중합 모델보다도 좋았습니다."</blockquote></details>

## Calibrated Confidence: 확신도를 실제 정답률에 맞추기

모델이 어떤 답에 80% 확신도(confidence)를 줬다면, 그런 판단들의 실제 정답률도 80% 근처여야 한다. Kanana-2-v-decide는 validation 데이터셋으로 confidence를 보정했다. 보정에 쓰지 않은 테스트셋에서 정답률은 그대로였고, confidence와 정답률의 차이를 나타내는 보정 오차(ECE)는 2.06%에서 1.12%로 내려갔다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"보정에 사용하지 않은 테스트셋에서 정답률은 그대로 유지됐고, confidence와 정답률의 차이를 나타내는 보정 오차(ECE)는 2.06%에서 1.12%로 낮아졌습니다."</blockquote></details>

운영 관점에서는 이 부분이 핵심이다. confidence가 정답률과 맞으면 threshold를 정해 확신이 낮은 판단만 사람 검토나 상위 LLM으로 넘길 수 있다.

한 가지 짚어 둘 것은 confidence가 1위 확률 그 자체가 아니라는 점이다. 저장소 코드는 1위 확률에서 나머지 선택지 확률의 평균을 뺀 값(margin)을 confidence로 돌려준다. 선택지가 둘뿐일 때와 열 개일 때 같은 정답이라도 값이 달라진다는 뜻이고, threshold를 선택지 목록과 묶어서 관리해야 하는 이유도 여기에 있다.

## 최종 성능 평가

비교 대상은 Microsoft-Decision-1, Jev 1.14.0, pplx-decider-flash-v1.2, d3-mini-4B, Solar Decide다. 한국어 중심 자체 평가셋(Kanana Decision Bench)과 공개 평가셋(Decision Index 0.4.1 public)에서 결과는 다음과 같다.

- **Kanana Decision Bench 텍스트**: 72.62점. Microsoft-Decision-1(71.98점), Jev 1.14.0(71.66점)을 조금 앞섰다.
- **Decision Index public 텍스트**: 61.35점. Jev(60.70점), pplx-decider-flash-v1.2(60.13점)보다 높다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Kanana Decision Bench 텍스트는 72.62점으로 Microsoft-Decision-1(71.98점), Jev 1.14.0(71.66점)을 조금 앞섰습니다. Decision Index public 텍스트에서도 61.35점으로 Jev(60.70점)와 pplx-decider-flash-v1.2(60.13점)보다 높은 점수를 기록했습니다."</blockquote></details>

차이가 1점 안팎이라 "압도적"이라기보다는 3B 자체 모델로 상용 Decision 모델과 비슷한 수준에 올라섰다고 읽는 편이 맞다.

수치를 인용할 때는 어느 출처인지 같이 적는 편이 안전하다. 위 점수는 카카오 기술블로그 기준이고, Hugging Face 모델 카드는 같은 항목을 Kanana Decision Bench 텍스트 72.50점, Decision Index public 61.34점(Vision 61.12점, 평균 61.23점)으로 적는다. 비교 대상도 StartLux-Decision-v1.1-2B-Instant, Liquid AI d1-3B, Base Labs Joey-1로 달라 집계 시점과 비교군이 다른 것으로 보인다. 모델 카드 기준으로는 Decision Index public에서 Joey-1이 63.31점으로 더 높아, 모든 지표에서 1위인 모델은 아니다.

## Kanana-2-v-decide 사용하기

### typesafe-sdk로 호출 (원문 예시)

원문은 Jev와 호환되는 API로 공개 모델을 직접 실행하고 typesafe-sdk로 호출하는 예시를 보여 준다. 고객 문의 하나에 문의 유형(Choice), 파손 여부(Noul), 고객 감정(Score)을 함께 묻는다. `base_url`은 원문에서도 비워 둔 자리라 아래 HTTP 서버처럼 직접 띄운 엔드포인트 주소를 넣으면 된다.

```python
# pip install typesafe-sdk
from typesafe_sdk import TypeSafeClient, Choice, Noul, Score

with TypeSafeClient(
    base_url="https://",
    api_key="unused",
) as client:
    result = client.system_one(
        model="Kanana-2-v-decide",
        state="주문한 상품이 파손되어 도착했습니다. 교환해 주세요.",
        questions={
            "category": Choice(
                instructions="고객 문의를 분류하세요.",
                criteria={
                    "delivery": "배송 조회",
                    "exchange": "교환 요청",
                    "payment": "결제 문의",
                },
            ),
            "damaged": Noul(instructions="상품이 파손되었는가?"),
            "sentiment": Score(
                instructions="고객의 감정은 얼마나 긍정적인가?",
                criteria=["부정적", "중립적", "긍정적"],
            ),
        },
    )
print(result.choices["category"].choice)  # 문의 유형
print(result.nouls["damaged"].noul)       # 파손되었을 확률 (0~1)
print(result.scores["sentiment"].score)   # 감정 점수 (0~2)
```

이미지는 state에 함께 넣는다. 상품 사진과 검색어를 주고 "사진 속 상품이 와이드 슬랙스인가?"를 Noul로 묻는 식이다.

### 모델 카드 기준 로컬 실행

[모델 카드](https://huggingface.co/kakaocorp/kanana-2-v-decide-3b-2611){:target="_blank"}에는 SDK 없이 쓰는 경로도 나와 있다. `pip install -r requirements.txt`로 의존성을 맞춘 뒤 transformers로 바로 부를 수 있다(`trust_remote_code=True` 필요).

```python
import torch
from transformers import AutoModel

model = AutoModel.from_pretrained("kakaocorp/kanana-2-v-decide-3b-2611", trust_remote_code=True,
                                  dtype=torch.bfloat16).to("cuda")

result = model.system_one(
    state="The item I ordered arrived damaged. Please replace it.",
    questions={
        "category": {"type": "choice", "instructions": "Classify the customer's inquiry.",
                     "criteria": {"delivery": "Delivery inquiry", "exchange": "Replacement request", "payment": "Payment inquiry"}},
        "damaged": {"type": "noul", "instructions": "Is the item damaged?"},
    },
)
print(result["answers"]["category"]["choice"])   # "exchange"
print(result["answers"]["damaged"]["noul"])      # probability of yes (0-1)
```

이미지는 `images=["photo.jpg"]`처럼 로컬 경로, URL, base64 data URI, PIL 이미지로 넘긴다. 기존 System One 클라이언트를 그대로 쓰고 싶다면 저장소에 들어 있는 `server_kanana_v_decide.py`를 띄우고 base URL만 바꾸면 된다.

```bash
python server_kanana_v_decide.py --port 8000
curl -s localhost:8000/v1/systemone -H 'content-type: application/json' -d '{
  "state": "The parcel never arrived.",
  "questions": {"refund": {"type": "noul", "instructions": "Does the customer want a refund?"}}
}'
```

모델 카드에서 운영 전에 알아 둘 점을 추렸다.

- 서버는 기본으로 127.0.0.1에만 바인딩된다. `--host`로 외부에 열기 전에 `DECISION_API_KEY`를 설정해 Bearer 인증을 켜야 한다. 요청은 한 번에 하나씩 처리한다.
- HTTP 서버에 넣는 이미지는 base64 data URL이어야 한다. 입력은 최대 20,480 토큰, 이미지는 최대 10장까지이고 비디오는 지원하지 않는다.
- 확률은 주어진 후보 안에서의 상대값이다. 선택지를 추가하거나 바꾸면 분포가 달라지니 threshold를 정했다면 선택지 목록이 바뀔 때 다시 맞춰야 한다.
- 너무 긴 state는 앞뒤를 남기고 잘린다. 잘리는 대신 거부하게 하려면 서버에 `--strict-inputs`를 준다.
- 선택지 임베딩은 (dtype, device, 토큰 ID)를 키로 호출 간에 캐시되며 최대 4,096개까지 LRU로 유지된다.

## 마치며

카카오는 이 모델을 웹 검색, 상품 카테고리 자동 분류 같은 내부 프로젝트에 먼저 적용하고 있다. 임베딩 기반 구조를 발전시켜 검색과 판단을 한 모델로 합치는 방향도 계획하고 있다.

에이전트를 직접 운영하는 입장에서 써 볼 지점은 분명하다. 라우팅, 도구 선택, 문의 분류처럼 선택지가 고정된 판단을 LLM 호출에서 떼어 내 3B 모델 한 번으로 처리하고 보정된 confidence가 낮은 건만 큰 모델이나 사람에게 넘기는 구성이다. 한국어와 이미지가 섞인 판단이 많다면 더 그렇다. 체크포인트가 공개돼 있으니 자기 데이터로 정확도를 재 보고 필요하면 미세조정(fine-tuning)까지 해 볼 수 있다.

같은 Kanana 2 계열의 비전 쪽 이야기는 [카카오 Kanana-2-vision-encoder 개발기]({{site.baseurl}}/kakao-tech/llm-model/vision/2026/10/22/kanana2-vision-encoder.html)에서 다뤘다.

## 참고

- [AI 에이전트를 위한 멀티모달 의사결정 모델 'Kanana-2-v-decide' 오픈소스 공개 - tech.kakao.com](https://tech.kakao.com/posts/846){:target="_blank"} (2026-11-06)
- [kakaocorp/kanana-2-v-decide-3b-2611 - Hugging Face](https://huggingface.co/kakaocorp/kanana-2-v-decide-3b-2611){:target="_blank"}
- [Kanana-2-v-decide 컬렉션 - Hugging Face](https://huggingface.co/collections/kakaocorp/kanana-2-v-decide-6adf5d04812736d0430c365e){:target="_blank"}
- [typesafe-sdk - PyPI](https://pypi.org/project/typesafe-sdk/){:target="_blank"}
- [trolley-bench - GitHub](https://github.com/zachlandes/trolley-bench){:target="_blank"}
- [if(kakao)26 발표: Kanana 멀티모달 임베딩 모델 개발기 - YouTube](https://www.youtube.com/watch?v=2crSurA7R-U){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/a-signpost-with-several-arrows-sOdzJEsGDLA){:target="_blank"}
