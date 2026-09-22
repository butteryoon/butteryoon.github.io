---
layout: post
comments: true
title: "transformers가 llama.cpp 양자화를 그대로 돌린다 — GGUF 네이티브 지원"
description: "Hugging Face transformers가 GGUF 체크포인트를 from_pretrained 한 줄로 로드한다. ggml Metal 커널을 재사용해 Apple Silicon에서 llama.cpp와 대등한 속도를 내고, 27B에서는 오히려 앞선다."
img: transformers_gguf_title.webp
date: 2026-09-22 21:30:00 +0900
last_modified_at: 2026-09-22 21:30:00 +0900
tags: [huggingface, transformers, gguf, llama-cpp, local-llm, apple-silicon, inference, llm] # add tag
related: llm
categories: dev
---
Hugging Face 블로그의 [「Transformers now runs llama.cpp quants」](https://huggingface.co/blog/transformers-llama-cpp-quants){:target="_blank"}(Marc Sun·Arthur Zucker·Lysandre, 2026-09-22)를 정리했다. 로컬 추론은 llama.cpp 계열(Ollama·LM Studio)이, 학습·평가는 transformers가 맡던 분업이 있었는데 그 경계가 흐려지는 변화다. 며칠 전 정리한 [RTX 로컬 LLM 가이드]({{site.baseurl}}/dev/2026/09/09/rtx-local-llm-guide.html)가 도구별 생태계를 훑었다면, 이번 건은 두 생태계가 같은 파일을 공유하기 시작한 이야기다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문·차트 대조 후 전면 재작성해 발행했다.)

<!--more-->

> **TL;DR:** transformers가 GGUF를 네이티브 지원한다. `from_pretrained(model_id, gguf_file="...")` 한 줄이면 llama.cpp 양자화 체크포인트가 그대로 로드된다. 내부에서 **ggml Metal 커널을 재사용**해 Apple Silicon에서 llama.cpp와 대등한 속도를 내고(Qwen3.5-4B 70.4 vs 71.8 tok/s), **Qwen3.8-27B에서는 15.9 대 13.4로 오히려 앞선다**. 다만 패킹된 추론 경로는 현재 **MPS 전용**이고 지원 아키텍처도 Qwen3.5·3.8 계열로 한정된다.

## 한 줄로 끝나는 로딩

GGUF는 llama.cpp 팀이 만든 로컬 추론용 단일 파일 포맷이다. 가중치·토크나이저·챗 템플릿이 한 파일에 들어 있고 `Q4_K_M` 같은 양자화 레벨로 메모리와 품질을 맞바꾼다. Hub에는 이미 수만 개의 GGUF 체크포인트가 올라와 있다.

이제 그 파일을 transformers가 직접 읽는다.

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

model_id = "unsloth/Qwen3.5-4B-GGUF"
filename = "Qwen3.5-4B-Q4_K_M.gguf"

tokenizer = AutoTokenizer.from_pretrained(model_id, gguf_file=filename)
model = AutoModelForCausalLM.from_pretrained(model_id, gguf_file=filename)
```

가중치가 Metal에서 패킹된 채 유지되면 `ggml-org/ggml-attn` 어텐션 커널이 자동으로 붙고, 조건이 안 맞으면 `sdpa`로 폴백한다. 서빙도 CLI 한 줄로 OpenAI 호환 엔드포인트가 뜬다.

```bash
transformers serve "unsloth/Qwen3.5-4B-GGUF:Qwen3.5-4B-Q4_K_M.gguf"
```

## 속도 — 대등하고, 한 곳에서는 앞선다

이 글에서 제일 눈에 띄는 대목이다. MacBook Pro M2 Max(32GB, macOS 26.6, PyTorch 2.12.1, kernels 0.17.0) 측정값이다.

| 모델 | transformers | llama.cpp |
|---|---|---|
| Qwen3.5-4B (Q4_K_M, 2.74GB) | 70.4 tok/s | 71.8 ± 0.4 |
| **Qwen3.8-27B (UD-Q4_K_M, 16.5GB)** | **15.9 tok/s** | **13.4 ± 0.9** |
| Qwen3.5-35B-A3B (UD-IQ4_XS, 16.3GB) | 60.2 tok/s | 61.3 ± 0.5 |

4B와 35B-A3B는 오차 범위에 가깝게 붙었고, **27B에서는 transformers가 더 빠르다.** 다만 측정 조건이 같지 않다는 점은 감안해야 한다 — transformers는 12토큰 프롬프트에서 128토큰을 생성하며 **prefill을 포함**한 값이고, llama.cpp는 `llama-bench tg128`의 **디코딩 전용** 값이다. 조건이 불리한 쪽이 transformers인데도 대등하게 나왔다는 의미다.

## 속도의 출처 두 가지

**하나, ggml 레이어 커널.** 양자화 가중치를 풀지 않고 그대로 읽는 `ggml-quantization`, 정규화를 퓨전하는 `ggml-norm`, Metal 플래시 어텐션 `ggml-attn`, Qwen3.5·3.8 하이브리드 구조의 선형 어텐션을 처리하는 `ggml-gated-delta-net`, 그리고 MoE 라우팅용 자체 `topk`가 붙는다. 양자화 커널만 켠 상태와 전체 커널을 켠 상태를 비교하면 차이가 분명하다.

| 모델 | 양자화 커널만 | 전체 레이어 커널 | 배수 |
|---|---|---|---|
| Qwen3.5-4B | 44.2 | 70.4 | 1.59× |
| Qwen3.8-27B | 10.5 | 15.9 | 1.51× |
| Qwen3.5-35B-A3B | 28.8 | 60.2 | **2.09×** |

**둘, 생성 루프 자체의 개선.** 패딩 없는 디코더 전용 입력에서 all-ones 어텐션 마스크를 생성 시작 시 버리고([#48814](https://github.com/huggingface/transformers/pull/48814){:target="_blank"}), 스토핑 검사를 비동기로 미뤄 CPU가 GPU를 기다리지 않게 했다([#47975](https://github.com/huggingface/transformers/pull/47975){:target="_blank"}).

| 모델 | 변경 전 | 변경 후 | 배수 |
|---|---|---|---|
| Qwen3.5-4B | 49.6 | 70.4 | 1.42× |
| Qwen3.8-27B | 13.7 | 15.9 | 1.16× |
| Qwen3.5-35B-A3B | 33.7 | 60.2 | **1.79×** |

이 두 번째 변경은 **GGUF와 무관하게 모든 transformers 모델의 생성 속도에 적용**된다. 커널은 Apple Silicon 한정이지만 루프 개선은 전역이다.

## 파인튜닝까지 이어진다

GGUF를 읽어 들여 디퀀타이즈한 뒤 표준 학습 워크플로로 넘길 수도 있다.

```python
model = AutoModelForCausalLM.from_pretrained(
    "unsloth/Qwen3.5-4B-GGUF",
    gguf_file="Qwen3.5-4B-Q4_K_M.gguf",
    quantization_config=GgufConfig(dequantize=True),
    dtype=torch.bfloat16,
)
```

같은 체크포인트로 추론·평가·파인튜닝을 한 생태계 안에서 끝낼 수 있다. 이번 통합에서 실제로 손에 잡히는 이득은 여기다.

## 제약 — 아직 좁다

- **MPS 전용**: 패킹된 추론 경로는 현재 Apple Silicon에서만 동작한다. 다른 디바이스는 디퀀타이즈 폴백이다.
- **패딩·배치 미완성**: 패딩된 배치는 마스크 최적화를 못 받아 성능이 떨어진다. `generate_batch`의 MPS 확장이 예정돼 있다.
- **아키텍처 커버리지**: Qwen3.5 dense/MoE와 호환되는 Qwen3.8까지다. 나머지는 차차 늘린다고 한다.

## 읽으면서 든 생각

- 벤치마크 표를 처음 봤을 때 "transformers가 llama.cpp의 90% 수준"쯤이려니 했는데 실제로는 대등하거나 앞섰다. 전용 C++ 런타임의 속도 우위가 **커널만 제대로 빌려오면 사라진다**는 게 이 글의 진짜 메시지로 읽힌다.
- 커널이 텐서 단위 연산이라 GGUF 포맷에 묶이지 않는다는 대목도 중요하다. 새 아키텍처가 llama.cpp 구현을 기다릴 필요 없이 transformers 쪽에서 먼저 가속될 수 있다는 뜻이다.
- 개인 개발 환경 관점에서는 M 시리즈 맥북에서 4B~35B급을 실사용 속도로 돌리면서, 평가·비교 코드를 표준 transformers API로 통일할 수 있게 됐다. 양자화 품질 검증(원본 대 GGUF 변환본)을 같은 코드로 돌린다는 것도 실무에서 꽤 쓸모 있다.

## 참고

- [원문: Transformers now runs llama.cpp quants](https://huggingface.co/blog/transformers-llama-cpp-quants){:target="_blank"}
- [transformers GGUF 문서](https://huggingface.co/docs/transformers/main/en/quantization/gguf){:target="_blank"} · [kernels 라이브러리](https://huggingface.co/docs/kernels/index){:target="_blank"}
- [GGUF 스펙 (ggml)](https://github.com/ggml-org/ggml/blob/master/docs/gguf.md){:target="_blank"}
- [관련 글: RTX PC에서 로컬 LLM 시작하기]({{site.baseurl}}/dev/2026/09/09/rtx-local-llm-guide.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/P6NY5x3ivYg){:target="_blank"}
