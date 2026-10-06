---
layout: post
comments: true
title: "결정 전용 모델이 오픈웨이트로 — autotrust/JEV-27B"
description: "고정 선택지 결정을 생성 없이 처리하는 폐쇄 모델 Jev의 오픈웨이트 대안이 나왔다. 백본을 얼린 채 0.4%만 학습해 빠른 결정과 기존 생성 능력을 한 가중치에 담은 Blocks of Experts 구조를 정리한다."
img: jev27b_blocks_of_experts_title.webp
date: 2026-10-06 22:20:00 +0900
last_modified_at: 2026-10-06 22:20:00 +0900
tags: [huggingface, jev, decision-model, open-weights, qwen, lora, inference, llm-serving, llm]
related: llm-serving
categories: dev
source_url: https://huggingface.co/blog/autotrust/autotrustjev-27b-fast-calibrated-decisions-and-ful
source_date: 2026-09-27
---

Hugging Face 블로그에 올라온 [autotrust/JEV-27B 소개 글](https://huggingface.co/blog/autotrust/autotrustjev-27b-fast-calibrated-decisions-and-ful){:target="_blank"}을 읽고 정리했다. 이 블로그에서 9월 말부터 세 번 다룬 주제 — 답이 정해진 작업에서 텍스트를 생성시키지 말고 확률만 받자는 접근 — 에 **오픈웨이트 모델**이 나온 셈이라 묶어서 본다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 전면 재작성해 발행했다.)

<!--more-->

> **TL;DR:** 에이전트가 내리는 결정 대부분은 긴 추론이 필요 없다. 이 메일이 피싱인가, 이 코드 변경이 규칙을 어겼는가 같은 것들이다. 폐쇄 모델 TypeSafe Jev 1.13이 이 자리를 차지하고 있었는데, Apache-2.0 오픈웨이트 대안 **JEV-27B**가 나왔다. Qwen3.8-27B를 **얼린 채 0.4%(1억 890만 파라미터)만 학습**해, 빠른 결정(System 1)과 기존 생성 능력(System 2)을 한 가중치에 담았다. 6개 결정 벤치마크 평균 84.07로 폐쇄 모델 83.85를 앞섰고(4승 2패), HumanEval은 78.0%로 학습 전후가 같다. 결정 1건 중앙값 137ms, 학습 비용은 B200 한 장으로 9.2시간이다.

## 1. 어떤 문제를 푸는가

에이전트를 돌리면 작은 판단이 끝없이 나온다. 이 메일이 피싱인지, 이 코드 변경이 규칙을 어겼는지 같은 것들이다. 여기에는 긴 사고 과정이 아니라 **빠르고 보정된 답**이 필요하다.

JEV-27B가 받는 질문은 세 가지 형태뿐이다.

| 형태 | 질문 | 반환 |
|---|---|---|
| `noul` | "이 서술이 참인가?" | `[P(거짓), P(참)]` |
| `choice` | "2~16개 보기 중 어느 것인가?" | 보기별 확률 |
| `score` | "0~5점 중 어디인가?" | 여섯 단계 분포 + 기댓값 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"Each answer takes one forward pass: no decoding, no JSON parsing, no prompt engineering. What you get back is a probability distribution, so you can act on it: auto-approve above 0.9, escalate below."</blockquote></details>

포워드 패스 한 번으로 끝나고, 디코딩도 JSON 파싱도 프롬프트 엔지니어링도 없다. 돌아오는 게 확률 분포라서 **0.9 넘으면 자동 승인, 그 아래는 사람에게**처럼 임계값 운영을 바로 얹을 수 있다.

[SGLang `/v1/score`로 같은 일을 하는 방법]({{site.baseurl}}/dev/2026/09/28/sglang-scoring-decision-engine.html)을 지난주에 정리했는데, 그쪽이 범용 모델의 로짓을 읽어 쓰는 방식이라면 이쪽은 **그 용도로 따로 학습한 모델**이다.

## 2. Blocks of Experts — 두 시스템이 가중치를 공유한다

구조가 이 글의 핵심이다. 세 부분으로 나뉜다.

- **얼린 백본**: Qwen3.8-27B. 학습하지 않는다
- **손대지 않은 생성 블록(`lm_head`)**: 기존 대화·코딩 능력이 그대로 남는다
- **새로 학습한 결정 블록**: LoRA + 디시전 헤드

학습한 파라미터는 **1억 890만 개로 백본의 0.4%**이고, **B200 한 장에서 9.2시간**이면 끝난다. 27B 모델에 새 능력을 붙이는 비용치고 작다.

여기서 확인해야 할 건 "결정 능력을 붙이면서 원래 능력이 깨지지 않았는가"다. 보통은 깨진다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"System 2 is untouched: HumanEval 78.0 % before and after; all 164 completions are byte-identical to the base model."</blockquote></details>

HumanEval 점수가 78.0%로 같은 정도가 아니라 **164개 완성 결과가 전부 바이트 단위로 동일**하다. 생성 경로를 아예 건드리지 않았다는 뜻이고, 점수만 같은 경우보다 훨씬 강한 주장이다.

## 3. 폐쇄 모델과 얼마나 가까운가

두 가지 방식으로 비교했다. 하나는 벤치마크 점수다.

| | JEV-27B | TypeSafe Jev 1.13 |
|---|---|---|
| 6개 결정 벤치마크 평균 | **84.07** | 83.85 |
| 승패 | 4승 (JevBench, OpenJev text, Nimble, MASSIVE-en) | 2승 (Kev +1.77, VitaminC +1.00) |

다른 하나가 더 흥미롭다. 점수가 아니라 **확률 분포 자체가 얼마나 닮았는지**를 쟀다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"mean KL ≈ 0.017 on 25,376 held-out questions labelled with Jev's own outputs. An observer needs about 60 sampled decisions to gather one nat of evidence about which model produced them."</blockquote></details>

53개 도메인에 걸친 홀드아웃 질문 25,376개에서 평균 KL 발산이 **약 0.017**이다. 원문은 이걸 "어느 모델이 낸 답인지 구분하려면 결정 60건을 표본으로 모아야 1 nat의 증거가 쌓인다"고 바꿔 말한다. 점수 비교보다 훨씬 까다로운 기준이고, 결정 모델에서는 **분포가 닮았는지**가 임계값 운영의 전제라 적절한 지표다.

## 4. 속도와 운영 요건

| 항목 | 값 |
|---|---|
| 결정 1건 중앙값 (B200) | **137ms** |
| 호스팅 API (독립 측정) | 238~301ms |
| H100 80GB, bf16 | 초당 약 100결정 · 동시 대화 64 · 약 1,250 tok/s |
| B300 | 초당 약 220결정 · 동시 대화 128 · 약 3,000 tok/s |

양자화 없이 **H100 한 장에 bf16 전체 정밀도로** 올라가고, B300에서와 같은 결정을 낸다고 밝혔다.

## 5. 읽고 나서

**"결정"이 하나의 제품 범주가 되고 있다.** 지난주 OpenAI DevDay에서 공개된 [Decisions API]({{site.baseurl}}/dev/2026/10/02/openai-devday-2026.html)도 미리 정해둔 선택지에서 고르게 해 응답을 1초 아래로 줄이는 물건이고, 원문 작성자인 Simon Willison은 그걸 Jev에 대한 대응으로 봤다. 같은 자리를 두고 폐쇄 API, 오픈 모델, 범용 모델의 로짓 읽기가 동시에 경쟁하는 모양새다.

**비용 구조가 이 발표의 진짜 내용이다.** 백본을 얼린 채 0.4%만 학습해 B200 한 장 9.2시간이면 된다는 건, 조직이 **자기 도메인 결정 모델을 직접 만들 수 있다**는 뜻이다. 공개된 건 범용 결정 모델이지만, 레시피 쪽이 더 쓸모 있어 보인다.

**다만 분포가 닮았다는 것과 정확하다는 것은 다르다.** KL 0.017은 "폐쇄 모델과 비슷하게 답한다"는 뜻이지 "맞게 답한다"는 뜻이 아니다. 임계값을 정하려면 결국 자기 데이터로 라벨을 붙여 보정해야 한다. SGLang 글에서도 같은 결론이었다 — 작업량의 대부분은 모델 연동이 아니라 평가셋 구축이다.

**한국어 쪽은 확인되지 않았다.** 벤치마크 6종 중 다국어 성격이 있는 건 MASSIVE-en인데 이름 그대로 영어 부분이다. 한국어 민원·티켓 분류 같은 용도로 쓰려면 직접 재 봐야 한다.

## 6. 참고 자료

- 원문: [autotrust/JEV-27B: fast calibrated decisions and full generation](https://huggingface.co/blog/autotrust/autotrustjev-27b-fast-calibrated-decisions-and-ful){:target="_blank"} (Hugging Face 블로그, 2026-09-27)
- [autotrust/JEV-27B 모델 카드](https://huggingface.co/autotrust/JEV-27B){:target="_blank"} — Apache-2.0
- [Qwen/Qwen3.8-27B (백본)](https://huggingface.co/Qwen/Qwen3.8-27B){:target="_blank"}
- 관련 글: [생성 대신 채점 — SGLang /v1/score]({{site.baseurl}}/dev/2026/09/28/sglang-scoring-decision-engine.html) · [OpenAI DevDay 2026]({{site.baseurl}}/dev/2026/10/02/openai-devday-2026.html)
