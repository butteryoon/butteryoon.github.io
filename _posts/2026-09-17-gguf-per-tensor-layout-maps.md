---
layout: post
comments: true
title: "GGUF 양자화, 텐서마다 비트를 다시 나눈다 — bartowski의 per-tensor layout map"
description: "bartowski가 Claude와 함께 1000개 넘는 양자화를 돌려 텐서별 민감도를 재고, 이를 솔버로 일반화해 GGUF 레시피를 자동 생성한 과정"
img: gguf-per-tensor-layout-maps_title.webp
date: 2026-09-17 18:30:00 +0900
last_modified_at: 2026-09-17 18:30:00 +0900
tags: [huggingface, gguf, quantization, llama-cpp, llm, optimization]
related: llm
categories: [huggingface-blog, llm, quantization]
source_url: https://huggingface.co/blog/bartowski/per-tensor-layout-maps-for-gguf-quantization
source_date: 2026-09-10
---

GGUF 양자화판을 받아 쓰는 사람이라면 `Q4_K_M` 같은 이름이 실제로 뭘 뜻하는지 한 번쯤 궁금했을 것이다. bartowski가 그 이름에 다시 의미를 넣고 덤으로 비트 배분까지 데이터로 새로 짠 과정을 공개했다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** bartowski가 Qwen3.5 0.8B/4B로 96시간 동안 1000개 넘는 양자화를 만들어 텐서별 KLD 민감도를 측정했다. 그 결과를 `prior.json`과 솔버로 일반화해 모델 shape만 넣으면 per-tensor layout map이 나오게 했고, `_S/_M/_L` 티어에 비율 규칙을 박아 이름의 의미를 되살렸다. 새 모델은 'canary' 테스트를 먼저 통과해야 맵이 적용되고, 실패하면 기존 휴리스틱으로 자동 폴백한다.

## 기존 휴리스틱이 버티지 못한 지점

llama.cpp의 `llama-quant.cpp`는 모델 구조만 보고 비트를 나눈다. `use_more_bits`가 대표적인데, 현재 레이어가 앞 1/8인지 뒤 1/8인지 아니면 중간의 세 번째마다인지를 따져 높은 양자화 타입을 얹는 식이다. 어텐션 레이어를 골라 일부는 올리고 일부는 깎아 목표 bpw를 맞추는 규칙도 여기 들어 있다.

문제는 이 규칙들이 결국 몇 해 전 테스트로 정해진 모델 비의존적 규칙이라는 점이다. expert가 8개를 넘는 MoE가 흔해지면서 한계가 드러났고(upstream은 mixtral의 expert 8개 케이스만 특수 처리한다), 아주 작은데도 민감한 shexp 텐서 같은 것도 걸렸다. bartowski는 자기 포크에 손을 대며 버텨왔지만, 새 모델이 나올 때마다 며칠씩 실험해 레시피를 짜는 방식은 오래 갈 수 없었다.

## 데이터 수집: degrade-one 스윕

질문은 단순했다. 텐서 하나하나의 상대적 민감도를 학습할 방법이 있는가, 있다면 일반화되는가.

실험은 Framework가 보내준 AMD AI Max+ 395 128GB 데스크톱에서 돌았다. 테스트 프레임워크는 Claude와 함께 짰고, 그 스크립트가 96시간 동안 1000개가 넘는 양자화를 만들고 평가했다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"I worked with Claude on a framework for testing, it wrote up some scripts, and I ran them for the next ~96 hours producing and evaluating over 1000 quants across various sizes to gather as much information as possible."</blockquote></details>

측정 지표는 KLD다. 양자화 모델의 토큰 확률 분포와 bf16 원본의 분포 사이 KL divergence를 `llama-perplexity --kl-divergence`로, wikitext-2-raw에서 컨텍스트 512로 쟀다. 낮을수록 좋다.

방법은 두 갈래였다. 모든 텐서를 `q8_0`에 두고 하나만 `q2_k`로 떨어뜨리는 degrade-one, 그 반대인 upgrade-one. 대상은 Qwen3.5-0.8B와 Qwen3.5-4B였다. 이 중 upgrade-one은 기대보다 쓸모가 없어서 — 대부분 노이즈로 묻혔다 — 이후 분석은 degrade-one 데이터가 끌고 갔다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"The first step was to attempt a wide sweep of &quot;degrade-one&quot; runs, where all tensors are set to q8_0 except for one which is set to q2_k, and &quot;upgrade-one&quot; runs, where all tensors are set to q2_k except for one which is set to q8_0."</blockquote></details>

Qwen에서 나온 경향이 다른 계열에도 통하는지 확인하려고 Gemma 4, Granite 4.2(3B/8B/30B), Laguna, Ling, Muse, Ornith로 추가 검증도 돌렸다. 깊이나 텐서 타입에서 Qwen만 유별난 건 아닌지 보는 용도였다.

## 데이터가 말한 것

**`token_embd`가 민감도 척도를 지배한다.** 0.8B에서 단일 가중치 텐서 중 최악인 것의 약 8배, 4B에서 약 16배다. 다만 `q4_k`만으로도 성능의 상당 부분을 되찾기 때문에 저비트 모델에서 늘 `q8_0`일 필요는 없고 대신 어느 정도의 bump는 항상 받는다. 기존의 `q8_0` 임베딩 버전들은 그래서 사라졌다. 그 자리에는 "그냥 한 단계 위 사이즈를 받으라"는 권고가 들어갔다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"token_embd dominates the sensitivity scale ... costing about 8x the worst single weight tensor at 0.8B and 16x at 4B. That said, it gets a large majority of its performance back from q4_k"</blockquote></details>

**깊이는 U자 곡선을 그린다.** 앞뒤 레이어가 민감하고 중간이 둔감하다. 이건 놀랄 일이 아니다. 몇 해 전부터 `use_more_bits`가 앞뒤를 올려온 이유가 바로 그거니까.

**비트당으로 보면 작은 어텐션 프로젝션이 제일 민감하다.** `attn_v`, `attn_output`, `ffn_up`, `ssm_out`이 두드러진다. 반대로 `ffn_gate`는 비트를 더 줘봐야 거의 의미가 없다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"Per bit, small attention projections are the most sensitive. In general, attn_v, attn_output, ffn_up, and ssm_out are extreme standouts on sensitivity. Interestingly, ffn_gate is basically never worth any extra bits"</blockquote></details>

## 솔버: prior.json이 레시피를 만든다

솔버는 모델 shape, `prior.json`, ggml 블록 크기 정보, 목표 양자화 타입을 받아 '최적' 레이아웃을 뱉는다. 이것도 Claude와 함께 만들었다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"With Claude's help, the solver was created with the goal of taking a model shape, the prior.json, some basic data about ggml block sizes, and the target quant type as input, and produce the &quot;optimal&quot; output."</blockquote></details>

솔버는 bump뿐 아니라 crush(덜 민감한 텐서를 깎아 비트를 확보하는 것)도 할 수 있다. 그런데 K-quant는 crush로 득을 보는 경우가 거의 없었다. 타입 간 간격이 너무 커서 가장 둔감한 텐서를 깎고 가장 민감한 텐서를 올려도 순손실이 난다. IQ-quant는 변화의 기울기가 더 촘촘해서 crush로 비트를 빼내 bump에 쓰는 거래가 성립할 때가 있다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"K quants almost never benefit from a crush, likely because the differences in K-quant types are just too large. IQ-quants benefit from a more granular slope of change"</blockquote></details>

두 번째 목표는 이름의 복권이었다. `Q3_K_`로 시작하는 파일이 정작 `Q3_K` 텐서는 얼마 없고 bpw가 5를 넘는다면 뭔가 잘못된 것이다. 그래서 본체 텐서의 bump 허용량에 규칙을 걸었다. `_S`는 90%가 지정된 타입이어야 하고, `_M`은 70%, `_L`은 최대 50%까지 bump를 허용한다. `Q3_K_S`를 봤다면 안에 `Q3_K` 텐서가 90%라는 뜻이다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"_S must be 90% the named tensor type, _M 70%, and _L is allowed to go up to 50% bump ... This means if you see Q3_K_S, you know it will be 90% Q3_K tensors inside."</blockquote></details>

목표가 다른 모델들이 결국 비슷한 크기로 수렴하던 'bpw collapse'를 피하려는 의도도 있었다. 덕분에 파일 크기도 달라진다. Qwen3.5-4B의 `Q3_K_M`은 15% 작아졌다. 비트를 제 자리에 쓰니 같은 파일 크기를 기준으로 고르면 전보다 나은 성능이 나온다는 이야기다.

호환성을 위해 K-quant는 IQ 텐서를 쓰지 못하게 막았다. IQ-quant 쪽에는 그런 제약이 없어 솔버가 원하는 타입을 자유롭게 고른다. `generate.py`에 있는 `TINY_PARAMS`는 극소 텐서를 F32로 고정해 속도와 품질을 챙기는 장치다.

## 일반화 시험과 canary 테스트

일반화 확인에는 Qwen-3.6-35B-A3B, Gemma 4 E4B, granite-4.2-8b, Ling 3.0 tiny/flash, MiniCPM5-2B, Muse Glimmer, 그리고 MLA 모델인 DeepSeek-V2-Lite가 동원됐다. 각각 솔버로 레시피를 만들어 양자화한 뒤, bartowski 자신의 `llama-quant.cpp` 포크로 만든 결과와 KLD를 맞붙였다.

첫 판에서 솔버가 이기지는 못했다. granite와 MiniCPM의 dense all-attention 레이아웃(그때까지 측정은 hybrid attention Qwen 모델로만 했다), 그리고 DeepSeek-V2-Lite가 모두 1차에서 실패했다. 본체 prior에 클래스별 테이블을 넣고 임베딩 규칙이 테이블의 파일 내 비중을 보도록 고친 뒤에야 통과했다. 3비트 아래에서는 dense 모델이 기존 방식과 사실상 동률이라는 점도 함께 적어두고 있다.

앞으로 새 모델 shape가 들어오면 먼저 테스트를 거친다. 키는 아키텍처 이름, 순서가 있는 텐서 이름·차원 목록, 그리고 방법이 바뀌면 재평가되도록 생성기 버전까지다. 맵과 기존 포크 양쪽으로 `Q4_K_M`, `Q3_K_M`, `IQ2_XS`(또는 그 모델의 최소 목표 타입)를 만들어 6개 모델의 KLD를 재고 'KLD per bit' 선 위에 찍는다. 맵으로 만든 것이 노이즈를 넘어 곡선 위로 벗어나면 맵을 버리고 기존 휴리스틱으로 되돌린다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"If any of the mapped quants fall above the curve (by more than noise), we reject the map and fallback to my own heuristic."</blockquote></details>

원문은 MiniCPM5-2B가 이 테스트에서 실제로 걸려 파이프라인이 원래 휴리스틱으로 정확히 폴백했다고 적는다. 다만 같은 글 앞부분에는 `MiniCPM5-2B-GGUF`와 `Gryphe_Pantheon-Reasoning-26B-A4B-1.1-V2-GGUF` 두 릴리스가 새 방식을 쓴다고도 쓰여 있어 두 서술이 그대로는 맞물리지 않는다. 원문 표현을 그대로 옮겨둔다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"While testing, MiniCPM5-2B actually failed my canary tests (mentioned above), and my pipeline correctly fell back to my original heuristic instead of pushing on."</blockquote></details>

canary 수치는 모델 카드의 `Per-tensor layouts` 섹션에 함께 올라간다. 튜닝 모델은 민감도 면에서 베이스와 같게 움직인다고 보고 shape를 키로 잡았는데, 아니라고 밝혀지면 다시 보겠다고 한다.

## 받는 쪽에서 달라지는 것

가장 눈에 띄는 변화는 `_L`의 의미다. 예전에 `Q2_K_L`, `Q3_K_XL`, `Q4_K_L`, `Q5_K_L`, `Q6_K_L`은 전부 "직전 양자화 그대로에 임베딩과 output만 `q8_0`"이라는 뜻이었다. 이제 `Q4_K_L`과 `Q6_K_L`은 텐서 타입 '예산'으로 남고, `Q2_K_L`·`Q3_K_XL`·`Q5_K_L`은 없어졌다. 임베딩 등급을 올리고 싶다면 사다리를 한 칸 올라가는 편이 낫다는 판단이다. 임베딩이 워낙 커서 `Q3_K_XL`이 `Q5_K_M`보다 커지던 gemma-4-E4B 같은 상황도 이걸로 정리된다.

파일 크기가 달라졌으니 전에 `Q4_K_M`이 겨우 들어갔다면 이제 `Q4_K_L`이 들어갈 수도 있다.

## 다음 과제와 한계

다음은 Qwen3.8-27B에 전체 맵을 돌려 KLD를 모으고 기존 업로드와 비교하는 일이다. 차이가 크면 정보를 붙여 올린다. 나머지 구형 모델은 수요가 있을 때만 손댈 계획이다.

막힌 곳도 있다. Qwen3.8-Flash-Next의 거대한 PLE n-gram table을 스크립트가 아직 다루지 못한다. 어렵진 않지만 우선순위에서 밀렸고 DeepSeek-V4.1-Flash도 n-gram table을 갖고 있어 다음 조사 대상이 됐다. Hy4-preview는 `q_lora_rank` 2048에 `kv_lora_rank` 512인 gated MLA에 sparse attention까지 얹어 준비가 안 된 상태고, 애초에 크기 때문에 만들 엄두를 못 냈다고 한다.

본인이 밝힌 한계는 네 가지다. KLD를 wikitext로만 쟀다는 점, prior를 작은 Qwen 모델 둘(과 나중의 granite)로만 적합시켰다는 점, ~Q5 위에서는 bump가 거의 노이즈라 논리가 유지된다고 가정할 뿐 증명하기 어렵다는 점, 그리고 아직 두 모델만 릴리스해 넓게 검증되지 않았다는 점이다.

## 참고 자료

- [원문: Per-tensor layout maps for GGUF quantization](https://huggingface.co/blog/bartowski/per-tensor-layout-maps-for-gguf-quantization){:target="_blank"} — bartowski, 2026-09-10
- [quantization-config 저장소](https://github.com/bartowski1182/quantization-config){:target="_blank"} — `solver.py`, `prior.json`, `generate.py`
- [MiniCPM5-2B-GGUF 모델 카드](https://huggingface.co/bartowski/MiniCPM5-2B-GGUF#per-tensor-layouts){:target="_blank"} — canary 수치 공개 예시
- [llama.cpp PR #28706](https://github.com/ggml-org/llama.cpp/pull/28706){:target="_blank"} — bartowski가 올린 양자화 생성 속도 개선 PR (진행 중)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/qOx9KsvpqcM){:target="_blank"}
