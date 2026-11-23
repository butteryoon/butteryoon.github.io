---
layout: post
comments: true
title: "Inkling 2 Mini 로컬 실행 정리 — 20GB 체크포인트, GGUF 양자화, reasoning effort, SGLang 서빙"
description: "Thinking Machines Lab의 오픈 웨이트 모델 Inkling 2 Mini를 로컬에서 돌리는 방법을 Hugging Face 블로그와 모델 카드 기준으로 정리했다. Apache 2.0, 약 20GB 체크포인트, 32GB 메모리 권장, llama.cpp GGUF(IQ4_NL·Q8_0), transformers의 reasoning_effort와 도구 호출, SGLang 서빙과 DSpark 추측 디코딩, 모델 카드가 적은 에이전트 사용 제약까지."
img: inkling-2-mini-local_title.webp
date: 2026-11-23 19:00:00 +0900
last_modified_at: 2026-11-23 19:00:00 +0900
tags: [huggingface, thinking-machines, inkling-2-mini, local-llm, gguf, llama-cpp, sglang, mlx, moe, llm-model, llm-serving, llm]
related: llm-model
categories: dev
source_url: https://huggingface.co/blog/inkling2mini
source_date: 2026-11-20
---
Hugging Face가 11월 20일 [Welcome Inkling 2 Mini](https://huggingface.co/blog/inkling2mini){:target="_blank"} 글을 올렸다. Thinking Machines Lab이 공개한 오픈 웨이트 멀티모달 모델이고 같은 계열의 Inkling 2 Small보다 훨씬 작아 로컬 장비를 겨냥한다. 글은 구조 설명과 데모가 절반이지만 로컬 모델을 직접 돌리는 쪽에서 오늘 확인할 것은 라이선스와 필요 메모리, 런타임별 실행 명령, reasoning effort 설정, 그리고 모델 카드가 적어 둔 에이전트 사용 제약이다. 블로그 글에 없는 라이선스·하드웨어·제약은 [모델 카드](https://huggingface.co/thinkingmachines/Inkling-2-Mini){:target="_blank"}에서 가져왔고 출처를 구분해 적는다. (이 글의 초안은 Orca Bots 가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Inkling 2 Mini는 Apache 2.0 오픈 웨이트 모델이고 텍스트와 이미지를 입력받아 텍스트를 낸다. 체크포인트는 약 20GB(NVFP4·MXFP8 혼합 정밀도)이며 모델 카드는 32GB 이상 메모리를 권장한다. 컨텍스트 창은 최대 1M 토큰이다. 24개 물리 레이어 중 가운데 12개를 세 번 반복해 토큰당 48번 레이어를 거친다. transformers, llama.cpp(GGUF IQ4_NL·Q8_0), MLX, SGLang에서 돌릴 수 있다. reasoning effort 기본값은 high(0.9)이고 최소로 내려도 추론이 완전히 꺼지지 않는다. 모델 카드는 에이전트로 쓸 때 샌드박스와 파괴적 동작 전 확인, 출력 토큰·턴 수 제한을 권한다. 벤치마크 수치는 모두 제공자가 밝힌 값이고 독립 재현 결과가 아니다.

## 1. 라이선스와 필요 하드웨어

블로그 글에는 라이선스와 메모리 요구량이 없다. 모델 카드 기준으로 정리하면 다음과 같다.

| 항목 | 값 | 출처 |
|------|------|------|
| 라이선스 | Apache 2.0 | 모델 카드 |
| 입력 / 출력 | 텍스트·이미지 / 텍스트 | 모델 카드 |
| 체크포인트 크기 | 약 20GB (NVFP4·MXFP8 혼합) | 모델 카드 |
| 컨텍스트 창 | 최대 1M 토큰 | 블로그, 모델 카드 |
| 파라미터 | 26.4B, 활성 3.4B | 모델 카드 |
| 지원 장비 | NVIDIA DGX Spark 1대, 메모리 32GB 이상 Apple silicon | 모델 카드 |
| 그 밖의 GPU | 32GB 이상 VRAM 권장 | 모델 카드 |

라이선스와 별도로 Model Acceptable Use Policy 링크가 모델 카드에 걸려 있다. 제품에 넣을 계획이면 이 문서도 같이 읽어야 한다.

파라미터 수는 블로그와 모델 카드의 표기가 조금 다르다. 블로그 TL;DR은 물리 파라미터 약 26B, 토큰당 활성 3B라고 적었고 모델 카드는 26.4B, 활성 3.4B라고 적었다. 모델 카드는 여기에 주석을 하나 붙였다. 반복 구간을 한 번 더 지날 때마다 유효 활성 파라미터가 약 1.0B 늘어나므로, 세 번 도는 이 모델의 토큰당 유효 활성 파라미터는 5.4B라는 것이다. 모델 카드 벤치마크 표의 "activated / total" 칸도 5.4/26.4로 적혀 있다. 활성 파라미터 수만 보고 속도를 짐작하면 실제보다 빠르게 잡게 된다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The model is distributed as a single checkpoint of about 20GB, in mixed NVFP4 and MXFP8 precisions."</blockquote><blockquote>"Other GPUs and platforms are possible, with 32GB or more of VRAM recommended"</blockquote><blockquote>"Note: each extra pass through the looped layers adds about 1.0B effective active parameters. Since the model runs with three passes, there are 5.4B effective active parameters per token."</blockquote></details>

## 2. 구조 — 가운데 12개 레이어를 세 번 돈다

Inkling 2 Mini는 디코더 레이어가 24개인 Mixture-of-Experts 모델이다. 레이어 0–5를 한 번, 6–17을 세 번, 18–23을 한 번 실행한다. 세 번의 통과는 같은 가중치를 다시 쓴다. 그래서 물리 레이어는 24개인데 토큰당 레이어 방문은 48번(6 + 3×12 + 6)이다.

- 은닉 크기는 2,304다. Inkling 2 Small은 4,608이다.
- 레이어 방문마다 토큰을 전문가 8개로 보낸다. Mini는 레이어당 128개 중에서, Small은 384개 중에서 고른다.
- 6개 레이어 블록마다 슬라이딩 윈도 레이어 5개와 전체 어텐션 레이어 1개가 있다. 로컬 윈도는 최대 1,024 토큰이다.
- 반복 구간의 각 통과는 같은 입력(앞 6개 레이어의 출력)에서 시작하고 통과 사이의 정보는 cross-pass message로 넘어간다.

운영 관점에서 기억할 것은 한 가지다. 가중치를 재사용해 체크포인트는 작아졌지만 48번의 방문은 전부 실제로 실행되고 방문마다 KV 상태를 따로 갖는다. 체크포인트 크기가 작다고 연산량과 캐시 메모리까지 작아지는 구조가 아니다.

반복 횟수를 바꿀 수 있는 연구용 체크포인트 `TML-d24-varloop-research`도 함께 공개됐다. 통과 횟수를 1이나 2로 낮추면 추론이 빨라지는 대신 품질이 조금 떨어질 수 있다고 한다. Hugging Face 쪽은 동작은 확인했지만 깊이 평가하지는 않았다고 적었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The decoder runs layers 0–5 once, layers 6–17 three times, and layers 18–23 once. All three passes through layers 6–17 reuse the same weights."</blockquote><blockquote>"The model still executes all 48 visits, and each visit keeps its own KV and short-convolution state."</blockquote></details>

## 3. llama.cpp와 Llama App — GGUF 4비트·8비트

가장 손이 덜 가는 경로다. GGUF 양자화는 4비트(IQ4_NL)와 8비트(Q8_0) 두 가지가 고정 반복 모델과 가변 반복 모델 양쪽에 제공된다. Llama App은 llama.cpp 기반이고 macOS 메뉴 막대나 Windows 시스템 트레이에서 로컬 모델을 관리하며 로컬 OpenAI 호환 API를 연다. 코딩 에이전트나 에디터를 붙일 때 쓰는 것이 이 API다.

Linux이거나 CLI로 서빙할 때의 명령은 원문에 다음과 같이 적혀 있다.

```bash
# 고정 반복 모델
llama serve -hf ggml-org/Inkling-2-Mini-GGUF:IQ4_NL

# 가변 반복 연구용 모델
llama serve -hf ggml-org/TML-d24-varloop-research-GGUF:IQ4_NL
```

8비트 가중치를 쓰려면 접미사를 `:Q8_0`으로 바꾼다. 두 양자화 파일의 크기와 필요 메모리는 블로그 글에 없다. 내려받기 전에 저장소의 파일 목록에서 확인해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"For the Inkling 2 Mini release we provide GGUF 4-bit (IQ4_NL) and 8-bit (Q8_0) quants for both models."</blockquote></details>

## 4. transformers — reasoning effort와 도구 호출

transformers에서는 `AutoModelForMultimodalLM`과 `AutoProcessor`로 불러온다. 원문의 기본 예제다.

```python
import torch
from transformers import AutoModelForMultimodalLM, AutoProcessor

model_id = "thinkingmachines/Inkling-2-Mini"

processor = AutoProcessor.from_pretrained(model_id)
model = AutoModelForMultimodalLM.from_pretrained(
    model_id,
)

messages = [{"role": "user", "content": [{"type": "text", "text": "Why is the sky blue?"}]}]

inputs = processor.apply_chat_template(
    messages, tokenize=True, add_generation_prompt=True, return_dict=True, return_tensors="pt",
).to(model.device)
input_len = inputs["input_ids"].shape[-1]

outputs = model.generate(**inputs, max_new_tokens=1024)
print(processor.decode(outputs[0][input_len:], skip_special_tokens=False))
```

이미지를 넣을 때는 `content`에 `{"type": "image", "image": "/path/to/image.jpg"}` 항목을 텍스트 항목 앞에 추가한다.

**reasoning effort.** `processor.apply_chat_template(...)`에 `reasoning_effort`를 넘기면 요청 단위로 추론 강도를 정한다. 채팅 템플릿이 그 값을 숫자로 바꿔 시스템 메시지로 넣는 방식이다.

| 수준 | 값 |
|------|------|
| none | 0.0 |
| minimal | 0.1 |
| low | 0.2 |
| medium | 0.7 |
| high (기본값) | 0.9 |
| xhigh | 0.99 |
| max | 0.99 |

이름 대신 0.0에서 0.99 사이의 숫자를 직접 넘길 수도 있다(원문 예: `reasoning_effort=0.35`). 주의할 점은 두 가지다. 기본값이 high라서 아무 설정 없이 돌리면 추론 토큰이 많이 나온다. 그리고 모델 카드는 reasoning effort를 최소로 내려도 추론이 완전히 꺼지지 않는다고 적었다. 짧은 응답이 필요한 호출이라면 none으로 내린 뒤 실제 출력 길이를 재 보고 판단해야 한다.

**도구 호출.** 원문 예제는 파이썬 함수를 `tools=[get_current_temperature]`로 `apply_chat_template`에 넘기고 생성 결과를 `processor.parse_response(output, prefix=inputs["input_ids"][0])`로 파싱해 `reply["tool_calls"][0]["function"]`에서 이름과 인자를 꺼낸다. 도구 결과는 `{"role": "tool", "name": call["name"], "content": str(temperature)}` 메시지로 다시 넣는다. 전체 코드는 원문에 있다.

**가변 반복.** 연구용 체크포인트는 `from_pretrained`에 `text_config={"loop_num_passes": 2}`를 넘겨 통과 횟수를 정한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Available levels are none (0.0), minimal (0.1), low (0.2), medium (0.7), high (0.9; default), xhigh (0.99), and max (0.99)."</blockquote><blockquote>"Setting reasoning effort to its minimum does not fully disable reasoning."</blockquote></details>

## 5. SGLang 서빙과 DSpark 추측 디코딩

데이터센터 장비에서 서빙할 때는 SGLang이 출시 당일부터 지원한다. 원문의 서버 실행 명령이다.

```bash
export MODEL="thinkingmachines/Inkling-2-Mini"

python -m sglang.launch_server \
  --model-path $MODEL \
  --served-model-name inkling \
  --attention-backend fa4 --bf16-gemm-backend torch --dtype bfloat16 \
  --tp 1 --page-size 128 --grammar-backend xgrammar \
  --host 0.0.0.0 --port 30000
```

원문은 전제 조건으로 "SM103에서 실행할 것"을 들었는데 SM103이 무엇인지는 설명하지 않았다. 이 글에서도 추정해 적지 않는다. 실행 전에 SGLang 쪽 문서에서 확인해야 하는 항목이다. 멀티모달 입력은 `--enable-multimodal`로 켠다. 서버가 뜨면 `/v1/chat/completions`에 `"model": "inkling"`으로 요청하고 요청 본문에 `"reasoning_effort": "low"`처럼 추론 강도를 넣을 수 있다.

체크포인트에는 DSpark 드래프터가 들어 있다. 3개 레이어짜리 드래프트 모델이 토큰을 제안하고 본 모델이 검증하는 추측 디코딩이다. 드래프트 가중치가 있는 상태에서 서버 명령에 다음을 추가한다.

```bash
--speculative-algorithm DSPARK --context-length 131072
```

Hugging Face는 NVIDIA B300 GPU 1장에서 프롬프트 1,024 토큰, 생성 1,024 토큰, 동시 요청 1·4·16·64 조건으로 추측 디코딩 유무를 비교했다. 다만 결과는 그래프 이미지로만 제시됐고 본문에 수치가 없다. 그래서 이 글에는 배속 수치를 적지 않는다. 벤치마크 스크립트는 원문에 접힌 블록으로 공개돼 있어 같은 조건으로 직접 잴 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"With the draft weights present, add --speculative-algorithm DSPARK --context-length 131072 to the server command."</blockquote><blockquote>"One prerequisite to take care of is to run it on SM103."</blockquote></details>

## 6. MLX와 그 밖의 런타임

Apple silicon에서는 mlx-lm에 지원이 들어갔고 이미지·텍스트 입력도 된다. mlx-community 조직에 양자화본이 올라와 있고 원본 체크포인트를 그대로 써도 된다. 이 경우 재변환 없이 원래의 nvfp4 + mxfp8 양자화가 그대로 쓰인다. 원문의 CLI 예다.

```bash
mlx_lm generate --model thinkingmachines/Inkling-2-Mini --image image.jpg --prompt "Describe this image.<|end_message|><|message_user|><|content_image|><|unused_200054|>" -m 1000
```

모델 카드는 블로그 글이 다룬 것 외에 vLLM, TokenSpeed, NVIDIA Dynamo, Unsloth(GGUF 양자화본 포함)를 지원 프레임워크로 적었고 Apple Silicon용으로 Ollama와 Exo Labs를 따로 들었다. 이 런타임들의 실행 명령은 블로그 글에 없으므로 각 문서를 봐야 한다.

## 7. 벤치마크 — 숫자로 확인되는 범위

블로그 글은 Gemma 4 31B, Muse Glimmer, Qwen3.8-27B, Nemotron 3.5 Super VL과 비교해 48개 벤치마크 중 32개에서 앞선다고 적었다. 블로그의 벤치마크 그림은 이미지라 수치를 읽지 않았고 아래는 모델 카드 표에 텍스트로 있는 값이다.

| 벤치마크 | Inkling 2 Mini | Qwen3.8-27B |
|------|------|------|
| AA Index (v5.0) | 20.1 | 20.1 |
| FrontierCode v2.0 | 15.1% | 14.9% |
| Terminal Bench v4.0 | 5.6% | 5.6% |
| AutomationBench-AA v1.0.6 | 41.8% | 48.2% |

블로그 서술(FrontierCode v2.0 선두, Terminal-Bench 4.0 동률, AutomationBench-AA에서 Qwen에 뒤짐)과 표가 일치한다. AA Index는 Qwen3.8-27B와 같은 20.1이다. Hugging Face 쪽도 자체 사용 테스트에서 "적어도 Qwen 3.8 27B와 비슷하다"고만 했고 검증이 더 필요하다고 덧붙였다. 표의 값은 제공자가 밝힌 것이므로, 쓰려는 작업과 하네스로 직접 재 보기 전에는 두 모델의 우열을 정할 근거가 되지 못한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"In our vibe tests, it's at least comparable with Qwen 3.8 27B. More validation is needed, but we are super excited about this release!"</blockquote><blockquote>"It leads in FrontierCode v2.0, ties with Qwen3.8-27B on Terminal-Bench 4.0, and trails Qwen on AutomationBench-AA v1.0.6."</blockquote></details>

## 8. 에이전트로 쓸 때 — 모델 카드가 적은 제약

블로그 글의 데모는 대부분 Pi 코딩 하네스에 모델을 붙여 스크린샷으로 자기 결과물을 확인하게 한 사례다. 로컬 코딩 에이전트의 모델로 쓰려는 독자가 많을 텐데, 모델 카드의 제약 항목은 그 용도와 직접 관련된다.

- **반복 추론.** 내부 테스트에서 모델이 긴 추론 루프에 갇히거나 진전 없이 자기 작업을 반복 점검하는 경우가 있었고 코드 디버깅에서 두드러졌다. 출력 토큰과 에이전트 턴 수에 상한을 두라고 권한다.
- **도구 사용 신뢰도.** 세부 지시와 프로젝트 관례를 따르는 데 큰 모델보다 덜 안정적이다. 확인 없이 파일을 지우거나 덮어쓰는 파괴적 동작을 할 수 있고 작업 결과를 정확히 보고하지 않을 수 있다.
- **영어 외 언어.** 영어 밖에서는 출력 품질이 낮을 수 있다. 모델 카드가 테스트 결과를 적은 언어는 중국어뿐이고(끊긴 표현, 언어 혼용) 한국어에 대한 언급은 없다. 한국어 품질은 직접 평가해야 한다.
- **사실 정확도.** 사실 회상과 지식 위주 질문에서 Inkling 2 Small과 차이가 가장 크고 틀린 정보를 확신 있게 말할 가능성이 큰 모델보다 높다.
- **연산량.** 가운데 레이어가 세 번 돌기 때문에 토큰당 연산량은 활성 파라미터 수가 시사하는 것보다 크다. 로컬 런타임의 반복 구조 지원이 새로 들어간 것이라 속도는 런타임과 하드웨어에 따라 다르다.

이 제약에서 나오는 설정은 분명하다. 에이전트 작업은 샌드박스에서 돌리고, 삭제·덮어쓰기 같은 되돌릴 수 없는 동작 앞에는 확인 단계를 두고, 출력 토큰과 턴 수에 상한을 건다. 모델이 "끝냈다"고 보고한 결과는 테스트나 diff로 따로 확인한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"We recommend setting limits on output tokens and agent turns."</blockquote><blockquote>"It may take destructive actions, such as deleting or overwriting files, without confirmation, and may not report the outcome of its work accurately."</blockquote><blockquote>"Run agentic workloads in a sandbox, and require confirmation before destructive or irreversible actions."</blockquote></details>

## 9. 정리 — 오늘 해 볼 수 있는 것

1. 메모리 32GB 이상 장비가 있으면 `llama serve -hf ggml-org/Inkling-2-Mini-GGUF:IQ4_NL`로 4비트 모델을 띄우고 로컬 OpenAI 호환 API에 기존 도구를 붙여 본다.
2. reasoning effort 기본값이 high라는 점을 감안해 작업 종류별로 수준을 내려 가며 출력 토큰 수와 품질을 잰다.
3. 코딩 에이전트에 붙일 때는 샌드박스, 파괴적 동작 전 확인, 출력 토큰·턴 수 상한을 먼저 건다.
4. 한국어 작업에 쓸 계획이면 한국어 평가를 직접 한다. 제공자 문서에는 한국어 결과가 없다.
5. 서빙 처리량이 필요하면 SGLang에서 DSpark 유무를 원문의 벤치마크 스크립트로 직접 비교한다.

이 글은 원문과 모델 카드를 정리한 것이고 개발 환경에서 모델을 직접 실행해 확인한 결과는 포함하지 않았다.

## 참고

- [Welcome Inkling 2 Mini, Cutting-edge Performance for your Local Setup — Hugging Face Blog](https://huggingface.co/blog/inkling2mini){:target="_blank"}
- [thinkingmachines/Inkling-2-Mini 모델 카드](https://huggingface.co/thinkingmachines/Inkling-2-Mini){:target="_blank"}
- [Llama App](https://llama.app/){:target="_blank"}
- [브라우저 에이전트 데모의 트레이스와 결과 파일](https://huggingface.co/buckets/huggingface-projects/inkling-2-mini-release-assets/tree/browser-agent-demo){:target="_blank"}
