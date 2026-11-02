---
layout: post
comments: true
title: "구글 딥마인드 EmbeddingGemma 2: 온디바이스 멀티모달 임베딩 모델 실무 가이드"
description: "텍스트·이미지·비디오·오디오를 768차원 단일 공간에 매핑하는 오픈 멀티모달 임베딩 모델. 모듈형 인코더·MRL·태스크 프리픽스로 온디바이스 RAG·검색·분류에 바로 쓴다."
img: embeddinggemma2_title.webp
date: 2026-11-02 19:30:00 +0900
last_modified_at: 2026-11-02 20:30:00 +0900
tags: [llm, llm-model, embedding, multimodal, gemma, on-device, rag, sentence-transformers]
related: llm-model
categories: dev
source_url: https://deepmind.google/research/publications/279443/
source_date: 2026-10-27
---

구글 딥마인드의 **EmbeddingGemma 2**는 텍스트(코드 포함)·이미지·비디오·오디오, 그리고 이들을 섞은 입력을 768차원 벡터 공간 하나에 매핑하는 오픈 멀티모달 임베딩 모델이다. 모델은 10월 6일 공개됐고 10월 27일 arXiv에 기술 보고서([2610.37364](https://arxiv.org/abs/2610.37364){:target="_blank"})가 올라왔다. 전체 7.4억 파라미터는 텍스트 모델 2.7억, 비전 인코더 1.7억, 오디오 인코더 3억으로 나뉘고 모바일·노트북 같은 일반 하드웨어에서 낮은 지연으로 임베딩을 뽑는 것을 목표로 설계됐다. 라이선스는 Apache 2.0이고 가중치는 허깅페이스(`google/embeddinggemma-2`)와 Kaggle에 있어서 내려받아 RAG·검색·분류·클러스터링 파이프라인에 바로 붙일 수 있다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:**
> - **모듈형 인코더**: `config_kwargs`로 비전·오디오 인코더를 빼고 텍스트만(2.7억), 텍스트+이미지(4.4억), 텍스트+오디오(5.7억), 전체(7.4억) 중 필요한 구성만 올린다.
> - **MRL(마트료시카 표현 학습)**: 768차원 출력을 512·256·128차원으로 잘라 쓸 수 있고 저장 공간은 최대 6분의 1로 준다. 256차원까지는 품질 손실이 작고 128차원은 텍스트 전용 워크로드에 맞다.
> - **태스크 프리픽스**: 검색·QA·팩트체크·코드 검색(비대칭)과 분류·클러스터링·유사도(대칭)용 프롬프트가 내장돼 있어 `prompt_name="SearchQuery"`처럼 이름만 넘기면 된다.
> - **정밀도**: `bfloat16`이나 `float32`로 돌린다. `float16`에서는 에러 없이 NaN이나 품질이 떨어진 임베딩이 나온다.
> - **배포 경로**: 공식 가중치 외에 llama.cpp용 GGUF, LiteRT, ONNX, MLX 변환본이 이미 올라와 있다.

## 1. 왜 지금 EmbeddingGemma 2인가

지금까지 임베딩 모델은 텍스트 전용 모델과 이미지·오디오 전용 모델로 갈려 있었다. 멀티모달 RAG를 만들려면 모달리티마다 모델을 따로 돌리고 벡터 공간을 맞추는 수고가 들었다. EmbeddingGemma 2는 모델 하나, 768차원 공간 하나로 네 모달리티를 다룬다. 기술 보고서는 모든 모달리티를 아우르는 종합 벤치마크에서 전문 모델과 2~12배 큰 모델을 함께 앞서 새로운 파레토 프런티어를 세웠다고 밝힌다. 모델 카드 기준 768차원 점수는 MTEB 다국어 61.36, MTEB 코드 78.68, MIEB 라이트 65.67, MMEB 비디오 50.67, MSEB 오디오 검색 69.54다. 전작 EmbeddingGemma 1과 비교하면 다국어 텍스트(61.15)는 비슷한 수준을 유지했고 코드는 68.76에서 78.68로 올랐다. EmbeddingGemma 1은 [RTX PC 로컬 LLM 글]({{site.baseurl}}/dev/2026/09/09/rtx-local-llm-guide.html)에서 Ollama 초경량 RAG 조합으로 다룬 적이 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Notably, in comprehensive embedding benchmarks across all modalities, EmbeddingGemma 2 establishes a new Pareto frontier, outperforming both specialist models and models two to twelve times larger."</blockquote></details>

## 2. 모듈형 인코더 — 필요한 모달리티만 올린다

실무에서 가장 쓸모 있는 기능은 **인코더 선택 로딩**이다. `sentence-transformers`의 `config_kwargs`로 비전·오디오 인코더를 끌 수 있다.

| 활성 모달리티 | `config_kwargs` | 유효 크기 |
|--------------|----------------|--------------|
| 텍스트만 | `{"vision_config": None, "audio_config": None}` | 270M |
| 텍스트 + 이미지 | `{"audio_config": None}` | 440M |
| 텍스트 + 오디오 | `{"vision_config": None}` | 570M |
| 전체 멀티모달 | `{}` | 740M |

텍스트 RAG만 할 거라면 270M만 올리면 되고 이미지 검색을 더하면 440M이 된다. 출시 블로그에 따르면 양자화를 적용한 Pixel 11 Pro에서 텍스트 전용 가중치는 약 191MB, 전체 멀티모달 모델은 약 567MB의 RAM을 쓴다. 인코더를 빼는 설정 방법은 라이브러리마다 다르니 쓰는 라이브러리의 문서를 확인해야 한다.

```python
from sentence_transformers import SentenceTransformer

# 텍스트만 — 가장 가볍고 빠르다
model = SentenceTransformer(
    "google/embeddinggemma-2",
    config_kwargs={"vision_config": None, "audio_config": None}
)
```

<details class="evidence"><summary>원문 근거</summary><blockquote>"Flexible footprint: Combines a 270M parameter text backbone (130M transformer + 140M embedder) with selectively loadable vision (170M) and audio (300M) encoders, allowing developers to load only the modalities required for their use case."</blockquote></details>

## 3. MRL — 차원을 잘라 저장 공간을 줄인다

Matryoshka Representation Learning으로 학습해서 768차원 출력의 **앞쪽 차원만 남겨 512·256·128차원으로 줄여 써도 된다**. `encode()`에 `truncate_dim`과 `normalize_embeddings=True`를 넘기면 절단과 정규화를 함께 처리한다.

| 출력 차원 | 압축률 | MTEB 다국어 | MTEB 코드 | MIEB 라이트 | MMEB 종합 |
|----------|--------|----------------|-----------|-------------|-----------|
| 768d (기본) | 1:1 | 61.36 | 78.68 | 65.67 | 59.01 |
| 512d | 1:1.5 | 61.17 | 77.24 | 65.34 | 58.38 |
| 256d | 1:3 | 60.41 | 76.18 | 64.16 | 56.24 |
| 128d | 1:6 | 57.89 | 71.41 | 59.97 | 45.65 |

256차원으로 줄여도 MTEB 다국어는 1점 안쪽(61.36→60.41), MIEB는 1.5점 남짓 떨어지는 데 그친다. 코드(−2.5점)와 MMEB 종합(−2.8점)은 하락 폭이 조금 더 크다. 벡터 DB 저장 공간을 3분의 1로 줄이는 대가로는 대체로 감수할 만하다. 128차원에서는 MMEB 종합이 45.65까지 내려가므로 텍스트 전용 워크로드에서만 검토하고 그때도 자기 데이터로 먼저 검증하는 편이 안전하다. 쿼리와 문서는 반드시 같은 차원으로 맞춰야 한다.

```python
query_emb = model.encode(
    query,
    truncate_dim=256,   # 128, 512도 가능
    normalize_embeddings=True,  # 필수 — 절단 후 L2 정규화를 빼면 순위 품질이 조용히 나빠진다
)
```

<details class="evidence"><summary>원문 근거</summary><blockquote>"Re-normalize after truncating: slicing a unit-length vector does not preserve unit length. The shortened vector must be L2-normalized before it is used for cosine similarity. Skipping this step degrades ranking quality silently—it produces plausible-looking scores rather than an error."</blockquote></details>

## 4. 태스크 프리픽스 — 이름 하나로 품질을 올린다

EmbeddingGemma 2는 텍스트 입력 앞에 짧은 태스크 지시어를 붙인 상태로 학습됐다. `sentence-transformers`에서는 `prompt_name`으로 고른다. 프리픽스는 텍스트에만 붙이고 이미지·비디오·오디오는 그대로 넘긴다.

| 용도 | 태스크 유형 | `prompt_name` | 쿼리 지시어 | 문서 지시어(제목 없으면 `none`) |
|------|------------|---------------|-------------|-------------------------------|
| 웹/문서 검색 | 비대칭 | `SearchQuery` | `task: search result \| query: {query}` | `title: {title} \| text: {content}` |
| 질의응답 | 비대칭 | `QuestionAnswering` | `task: question answering \| query: {question}` | `title: {title} \| text: {passage}` |
| 팩트체크 | 비대칭 | `FactChecking` | `task: fact checking \| query: {claim}` | `title: {title} \| text: {evidence}` |
| 코드 검색 | 비대칭 | `CodeRetrieval` | `task: code retrieval \| query: {query}` | `title: {title or filename} \| text: {code}` |
| 텍스트 분류 | 대칭 | `Classification` | `task: classification \| query: {content}` | N/A |
| 클러스터링 | 대칭 | `Clustering` | `task: clustering \| query: {content}` | N/A |
| 문장 유사도 | 대칭 | `SentenceSimilarity` | `task: sentence similarity \| query: {content}` | N/A |

문서 쪽 처리에서 실수하기 쉽다. `prompt_name="Document"`는 `title: none`을 붙여 주는 단축 옵션이라 **제목이 없는 문서에만** 쓴다. 제목이 있는 문서는 포맷을 직접 만들어 `prompt_name` 없이 인코딩해야 한다.

```python
# 검색 쿼리
query_emb = model.encode(query, prompt_name="SearchQuery")
# 제목 없는 문서 — "title: none | text: ..."가 자동으로 붙는다
doc_emb = model.encode(content, prompt_name="Document")
# 제목 있는 문서 — 직접 포맷하고 prompt_name은 넘기지 않는다
doc_emb = model.encode(f"title: {title} | text: {content}")
```

프리픽스를 빼도 동작은 하지만 정밀도가 떨어진다. 검색·QA 같은 비대칭 태스크는 쿼리 프리픽스와 문서 포맷을 짝지어 써야 한다. 분류·유사도 같은 대칭 태스크는 비교하는 모든 입력에 같은 프리픽스를 붙인다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"EmbeddingGemma 2 is trained with short task instruction prefixes prepended to text inputs. Using the right prefix improves quality; omitting it still works but reduces precision. Prefixes apply to text only. Pass images, video, and audio without any prefix."</blockquote></details>

## 5. 수치 정밀도 — float16은 쓰지 않는다

이 모델의 활성화 값 범위는 `float16`이 표현할 수 있는 범위를 넘는다. `float16`으로 돌리면 에러 없이 **NaN이나 품질이 떨어진 임베딩을 돌려준다**. `bfloat16`은 `float32`와 같은 8비트 지수부를 써서 안전하고 메모리도 절반이다. 하드웨어가 지원하면 `bfloat16`을 기본으로 쓰고 대부분의 CPU를 포함한 나머지 환경에서는 `float32`를 쓴다.

```python
import torch
from sentence_transformers import SentenceTransformer

dtype = torch.bfloat16 if torch.cuda.is_bf16_supported() else torch.float32
model = SentenceTransformer(
    "google/embeddinggemma-2",
    model_kwargs={"torch_dtype": dtype}
)
```

<details class="evidence"><summary>원문 근거</summary><blockquote>"Run inference in `bfloat16` or `float32`. Do not use `float16`. EmbeddingGemma 2's activation range exceeds the dynamic range of `float16`. In `float16` the model returns NaN or silently degraded embeddings rather than raising an error, so the failure is easy to miss."</blockquote></details>

## 6. 멀티모달 입력 — 섞어서 하나의 임베딩으로

텍스트·이미지·비디오·오디오를 **8,192토큰 컨텍스트 하나에 섞어 넣을 수 있다**. 미디어가 들어갈 위치는 텍스트 안에 플레이스홀더 토큰(`<|image|>`, `<|video|>`, `<|audio|>`)으로 표시한다. 모델 카드의 상품 리스팅 예시는 다음과 같다.

```python
emb_interleaved = model.encode({
    "text": "Waterproof running shoes. <|image|> Featuring a breathable mesh upper. <|image|> Grip test on wet rock: <|video|>",
    "image": ["shoe.jpg", "mesh.jpg"],
    "video": "demo.mp4",
})
```

플레이스홀더는 같은 이름의 키에서 순서대로 채워지고 결과는 텍스트·이미지·비디오를 함께 표현하는 임베딩 하나다. 이 벡터는 "waterproof shoes for trail running" 같은 텍스트 쿼리 임베딩과 바로 비교할 수 있다.

모든 모달리티가 같은 8,192토큰 예산을 나눠 쓴다. 기본 설정에서 이미지는 장당 280토큰(최대 약 29장), 비디오는 프레임당 140토큰(약 58프레임), 오디오는 초당 25토큰(약 327초)이다. 비디오는 기본적으로 초당 1프레임으로 샘플링하고 이 값은 바꿀 수 있다. 오디오는 16kHz 모노로 넣어야 한다. 이미지 토큰 예산은 70~1120 사이에서 조절할 수 있어서 예산을 낮추면 이미지·프레임을 약 114개까지 넣을 수 있고 높이면 세밀한 시각 정보를 더 잡는 대신 지연이 늘어난다.

## 7. 라이선스와 배포 경로

| 항목 | 내용 |
|------|------|
| **라이선스** | Apache 2.0 (상업 이용·수정·재배포 허용) |
| **공식 가중치** | 허깅페이스 `google/embeddinggemma-2`, Kaggle |
| **llama.cpp GGUF** | `ggml-org/embeddinggemma-2-GGUF` (BF16·Q8_0, 비전·오디오용 `mmproj` 파일 별도) |
| **온디바이스(LiteRT)** | `litert-community/embeddinggemma-2-740m-litert-lm`, 텍스트 전용 `-text-270m-`, 텍스트+비전 `-text-vision-440m-` 변형 |
| **ONNX / transformers.js** | `onnx-community/embeddinggemma-2-ONNX` (fp16·q4·quantized 그래프) |
| **MLX** | `mlx-community/embeddinggemma-2-bf16`, `-8bit` 등 |

출시 블로그에 따르면 transformers, sentence-transformers, MLX, vLLM, llama.cpp, SGLang, Ollama, LM Studio에서 바로 서빙할 수 있고 파인튜닝은 Unsloth 가이드를 따르면 된다. ggml-org GGUF는 모델 카드에 `llama serve -hf ggml-org/embeddinggemma-2-GGUF` 한 줄 실행법이 적혀 있다. Apache 2.0이어도 배포할 때는 [Gemma 금지 사용 정책](https://ai.google.dev/gemma/prohibited_use_policy){:target="_blank"}을 지켜야 한다.

## 8. 바로 써 보기 — 최소 코드

```bash
pip install -U sentence-transformers transformers torch
```

```python
from sentence_transformers import SentenceTransformer
import torch

# 1) 모델 로드 — 텍스트만, 가능하면 bfloat16
dtype = torch.bfloat16 if torch.cuda.is_bf16_supported() else torch.float32
model = SentenceTransformer(
    "google/embeddinggemma-2",
    config_kwargs={"vision_config": None, "audio_config": None},
    model_kwargs={"torch_dtype": dtype}
)

# 2) 검색용 임베딩
queries = ["파이썬 비동기 처리 베스트 프랙티스", "RAG 파이프라인 구성 방법"]
docs = [
    "title: asyncio 가이드 | text: asyncio.run()으로 진입점을 만들고...",
    "title: RAG 아키텍처 | text: 청킹→임베딩→벡터DB→리랭킹 순서로...",
]

q_embs = model.encode(queries, prompt_name="SearchQuery", truncate_dim=256, normalize_embeddings=True)
# 제목을 직접 포맷한 문서이므로 prompt_name은 넘기지 않는다
d_embs = model.encode(docs, truncate_dim=256, normalize_embeddings=True)

# 3) 코사인 유사도
scores = model.similarity(q_embs, d_embs)
print(scores)  # shape: (2, 2)
```

GPU가 없는 환경이면 `float32`로 돈다. 더 가볍게 올리고 싶다면 llama.cpp에서 `ggml-org/embeddinggemma-2-GGUF`의 Q8_0 파일(약 310MB)을 쓰거나, 모바일·엣지라면 LiteRT 변환본을 쓰면 된다.

## 9. 한계와 주의사항

- **128차원은 멀티모달 품질이 크게 떨어진다** — 텍스트 전용일 때만 검토하고 그 전에는 256차원을 쓴다.
- **쿼리와 문서는 같은 차원이어야 한다** — 768차원 쿼리를 128차원 코퍼스와 비교할 수 없다.
- **`float16`은 쓰지 않는다** — 에러 없이 품질만 떨어져서 원인을 찾기 어렵다.
- **태스크 프리픽스를 빼면 정밀도가 떨어진다** — 검색·QA는 쿼리 프리픽스와 문서 포맷을 짝지어 쓴다.
- **100개 이상 언어를 지원하지만 언어마다 성능이 같지는 않다** — 모델 카드도 언어별 편차를 한계로 명시한다. 한국어 워크로드라면 자기 데이터로 확인하는 편이 좋다.
- **사전 학습만 거친 임베딩 모델이다** — 생성 모델과 달리 사후 정렬이나 안전 튜닝을 거치지 않았으므로 검색 필터링·공정성 점검 같은 안전장치는 애플리케이션 쪽에서 챙겨야 한다([책임감 있는 생성형 AI 툴킷](https://ai.google.dev/responsible){:target="_blank"} 참고).
- **관리형 엔드포인트는 아직 없다** — 허깅페이스 Inference Providers 연결이 없고 Gemini Enterprise Agent Platform Model Garden 제공은 "곧" 예정이라고만 밝혔다. 당장은 직접 서빙해야 한다.

## 10. 참고 자료

- 원문: [EmbeddingGemma 2: Unified and Efficient Multimodal Representations](https://deepmind.google/research/publications/279443/){:target="_blank"} (Google DeepMind, 2026-10-27) / [arXiv:2610.37364](https://arxiv.org/abs/2610.37364){:target="_blank"}
- 허깅페이스 모델 카드: [google/embeddinggemma-2](https://huggingface.co/google/embeddinggemma-2){:target="_blank"}
- 공식 문서: [EmbeddingGemma Documentation](https://ai.google.dev/gemma/docs/embeddinggemma){:target="_blank"}
- 출시 블로그: [EmbeddingGemma 2: an open, lightweight multimodal embedding model](https://blog.google/innovation-and-ai/technology/developers-tools/embeddinggemma-2/){:target="_blank"} (2026-10-06)
- llama.cpp GGUF: [ggml-org/embeddinggemma-2-GGUF](https://huggingface.co/ggml-org/embeddinggemma-2-GGUF){:target="_blank"}
- Gemma 금지 사용 정책: [Prohibited Use Policy](https://ai.google.dev/gemma/prohibited_use_policy){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/ihh6yVyZIpw){:target="_blank"}
