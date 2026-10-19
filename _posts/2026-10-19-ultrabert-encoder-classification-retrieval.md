---
layout: post
comments: true
title: "UltraBERT — 분류와 검색을 모델 하나로, 32K 컨텍스트 영어 인코더 써 보기"
description: "DMIS Lab이 공개한 UltraBERT(150M·380M)와 임베딩 모델 UltraBERT-Embed-150M을 정리한다. 분류와 검색이 서로 다른 마스킹 비율을 원한다는 발견, LM 헤드 분리 레시피, 32K 패스키 99%, MTEB 67.24, 그리고 sentence-transformers로 바로 붙이는 법과 도입 전에 따질 점까지."
img: ultrabert-encoder_title.webp
date: 2026-10-19 20:20:00 +0900
last_modified_at: 2026-10-19 20:20:00 +0900
tags: [huggingface, ultrabert, encoder, embedding, retrieval, classification, sentence-transformers, llm-rag, rag-method]
related: rag-method
categories: [huggingface-blog, llm-rag, embedding]
source_url: https://huggingface.co/blog/dmis-lab/ultrabert
source_date: 2026-10-14
---

RAG 파이프라인에서 문서를 벡터로 바꾸는 일, LLM 트래픽을 걸러내는 분류기를 돌리는 일은 대부분 작은 인코더가 맡는다. 그런데 지금까지는 분류에 강한 인코더와 검색에 강한 인코더가 따로였다. DMIS Lab이 10월 14일 Hugging Face 블로그에 공개한 UltraBERT는 이 둘을 모델 하나로 합쳤다고 주장한다. 무엇이 공개됐고 내 파이프라인에 어떻게 붙이며 도입 전에 무엇을 따져야 하는지 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** UltraBERT는 150M·380M 영어 인코더이고 컨텍스트는 32K 토큰이다. 여기에 바로 쓸 수 있는 임베딩 모델 UltraBERT-Embed-150M이 함께 나왔고 셋 다 Apache 2.0이다. 핵심 발견은 마스킹 비율이 낮으면 분류가, 높으면 검색이 좋아진다는 것이다. 그래서 5%와 50% 마스킹을 한 배치에 섞고 비율마다 LM 헤드를 따로 뒀다. 저자 측정으로 Embed-150M은 MTEB(eng, v2) 67.24로 200M 이하 dense 임베딩 모델 중 1위이고 32K 길이까지 패스키 정확도 99%를 유지한다. 다만 **영어 전용**이라 한국어 문서가 중심인 RAG에는 그대로 쓸 수 없다.

## 무엇이 공개됐나

| 모델 | 크기 | 용도 |
|---|---|---|
| [UltraBERT-150M](https://huggingface.co/dmis-lab/UltraBERT-150M){:target="_blank"} | 150M | 사전학습 인코더(분류·NER·QA 파인튜닝용) |
| [UltraBERT-380M](https://huggingface.co/dmis-lab/UltraBERT-380M){:target="_blank"} | 380M | 사전학습 인코더 |
| [UltraBERT-Embed-150M](https://huggingface.co/dmis-lab/UltraBERT-Embed-150M){:target="_blank"} | 150M | 바로 쓰는 텍스트 임베딩(768차원) |

세 모델 모두 컨텍스트가 32K 토큰이고 라이선스는 Apache 2.0이다. 모델 카드의 언어 태그는 영어(`en`) 하나뿐이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"We're releasing UltraBERT, a family of English encoders (150M and 380M) with a native 32K context, plus UltraBERT-Embed-150M, a ready-to-use embedding model built on top of it. Everything is Apache 2.0 and lives in the UltraBERT collection."</blockquote></details>

## 분류와 검색은 반대 방향의 마스킹을 원한다

저자들은 150M·380M·1.3B 인코더를 같은 데이터로 학습시키며 레시피를 다시 따져 봤다. 실험에 쓴 토큰을 모두 합치면 70조 개가 넘는다. 실무에 쓸 만한 발견은 세 가지다.

- **인코더는 일찍 포화한다.** 손실은 8.8T 토큰까지 계속 내려가지만 GLUE·SQuAD는 1~2T 토큰 언저리에서, BEIR는 그보다 먼저 멈춘다. 같은 레시피로 데이터만 늘려서는 더 나아지지 않는다는 뜻이다.
- **마스킹 비율이 모델의 성격을 정한다.** 마스킹이 낮을수록 분류가 좋아지고 검색은 50% 마스킹일 때 모든 크기에서 BEIR가 가장 높다. 기존 인코더는 둘 중 하나를 고른 셈이라 GLUE 1위와 BEIR 1위가 서로 다른 모델이다.
- **증류는 낮은 마스킹에서만 효과가 있다.** 380M 기준으로 15% 마스킹에서는 1.3B 교사 모델 증류가 GLUE를 90.2에서 90.8로 올렸지만, 50% 마스킹에서는 BEIR를 47.2에서 46.5로 떨어뜨렸다.

## 레시피: 배치는 섞고, LM 헤드는 나눈다

해법은 단순하다. 한 배치에 5% 마스킹 시퀀스와 50% 마스킹 시퀀스를 4:1로 섞는다. 인코더 본체는 공유하고 LM 헤드는 비율마다 따로 둔다. 헤드를 같이 쓰면 두 체제가 출력층을 서로 다른 모양으로 끌어당겨 다툰다는 것이 저자들의 분석이다. 증류는 앞의 규칙대로 5% 헤드에만 걸고 50% 헤드는 일반 MLM으로 학습한다. 사전학습은 2T 토큰이고 마지막 100B 토큰은 32K 컨텍스트로 학습했다.

아래는 150M 모델로 마스킹 전략을 비교한 결과다.

| 전략(150M) | GLUE | BEIR |
|---|---|---|
| 단일 비율 6% | 89.5 | 43.5 |
| 단일 비율 50% | 87.5 | 45.0 |
| 5~50% 균등 샘플링 | 88.0 | 44.2 |
| 5%+50% 혼합, 헤드 공유 | 88.9 | 44.7 |
| 5%+50% 혼합, 헤드 분리 | **89.4** | **45.1** |

<details class="evidence"><summary>원문 근거</summary><blockquote>"With decoupled heads, one 150M encoder reaches 89.4 GLUE and 45.1 BEIR, matching the best single-ratio model on classification (89.5 at 6%) and beating the best one on retrieval (45.0 at 50%)."</blockquote></details>

## 성능: 같은 크기 인코더와 비교

저자들은 최근 인코더들을 과제마다 같은 그리드 서치로 파인튜닝해 비교했고 두 크기 모두 10개 벤치마크 중 9개에서 1위를 했다고 밝혔다. 자주 비교되는 모델만 추렸다. 검색 점수는 각 백본을 MS MARCO로 똑같이 파인튜닝해 잰 값이므로, 완성된 임베딩 모델의 점수가 아니라 백본 자체의 잠재력으로 읽어야 한다.

| 모델 | 크기 | GLUE | BEIR | RAR-b Code | LongEmbed |
|---|---|---|---|---|---|
| ModernBERT | 149M | 88.4 | 42.6 | 29.6 | 63.2 |
| Ettin Encoder | 149M | 88.9 | 42.4 | 31.8 | 60.1 |
| **UltraBERT** | 152M | **89.5** | **45.2** | **39.1** | **72.0** |
| ModernBERT | 395M | 90.4 | 45.2 | 34.0 | 63.5 |
| Ettin Encoder | 395M | 90.8 | 45.9 | 38.5 | 64.3 |
| **UltraBERT** | 380M | **91.0** | **47.7** | **47.0** | **74.9** |

긴 문서를 다루는 사람이라면 패스키 결과를 눈여겨볼 만하다. LongEmbed 패스키 테스트에서 512 토큰 모델은 1K를 넘기면 실패하고 8K 모델은 16K·32K에서 급락한다. UltraBERT는 두 크기 모두 32K까지 모든 길이에서 99%를 유지했다. 문서를 잘게 자르지 않고 통째로 넣어도 된다는 뜻이고 분류나 판정 작업에서도 대화 전체를 한 번에 보고 결정할 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"UltraBERT holds 99% at every length up to 32K, at both sizes."</blockquote></details>

## UltraBERT-Embed-150M: 바로 쓰는 임베딩 모델

RAG에 바로 붙일 수 있는 쪽은 임베딩 모델이다. UltraBERT-150M을 두 단계 대조 학습으로 만들었는데, 1단계는 약 5억 쌍의 약지도 텍스트·코드 쌍, 2단계는 하드 네거티브를 붙인 570만 개 정제 예제(입력 최대 32K)다. 공개 데이터만 썼고 더 큰 임베딩 모델에서 증류하지도 않았다.

| 모델 | 크기 | MTEB(eng, v2) | MTEB(Code, v1) | LongEmbed |
|---|---|---|---|---|
| **UltraBERT-Embed-150M** | 151M | **67.24** | **73.30** | **79.30** |
| gte-modernbert-base | 149M | 66.26 | 71.66 | 57.0 |
| bge-base-en-v1.5 | 109M | 65.14 | 50.35 | 33.9 |
| nomic-embed-text-v1.5 | 137M | 62.20 | 47.36 | 30.60 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"On MTEB(eng, v2) it scores 67.24, the highest of any dense model under 200M parameters."</blockquote></details>

이미 `bge-base-en-v1.5`나 `nomic-embed-text-v1.5`로 영어 문서를 임베딩하고 있다면 같은 크기대에서 교체를 검토해 볼 만하다. 차이가 큰 쪽은 코드 검색과 긴 문서다.

## 써 보려면

`transformers>=5.20`부터 기본 지원한다. 그보다 낮은 5.x 버전에서는 `trust_remote_code=True`를 줘야 한다. 긴 입력에는 FlashAttention 2를 권장하며 SDPA도 동작한다.

```bash
pip install -U "transformers>=5.20" "sentence-transformers>=6.0"
```

임베딩은 sentence-transformers로 붙인다.

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer(
    "dmis-lab/UltraBERT-Embed-150M",
    model_kwargs={"attn_implementation": "flash_attention_2", "dtype": "bfloat16"},
)
queries = ["Win big, click now!"]
labels = ["spam", "promotional", "announcement"]
query_embeddings = model.encode(queries, prompt="Instruct: Labeling spam emails\nQuery: ")
label_embeddings = model.encode(labels, prompt="Label: ")
print(model.similarity(query_embeddings, label_embeddings))
```

프롬프트 규칙은 지켜야 한다. 쿼리에는 `Instruct: {task}\nQuery: ` 접두어를 붙이고 문서는 그대로 넣는다. `prompt_name="query"`를 쓰면 기본 검색 지시문이 붙는다. 모델 카드에 따르면 위 예제처럼 레이블 문구에는 `Label: ` 접두어를 붙이고 STS·중복 탐지 같은 대칭 과제에서는 양쪽에 같은 지시문을 붙인다.

백본을 직접 파인튜닝한다면 **학습률을 낮게 잡아야 한다.** 저자들에 따르면 UltraBERT는 다른 인코더보다 최대 10배 낮은 학습률을 선호한다. MNLI에서는 1e-5(150M)·8e-6(380M)가 가장 좋았는데, 다른 인코더는 보통 3e-5~5e-5를 쓴다. 검색 파인튜닝에서는 5e-5(150M)·3e-5(380M)가 가장 좋았고 다른 인코더는 보통 3e-4~5e-4를 쓴다. 평소 쓰던 하이퍼파라미터를 그대로 가져오면 UltraBERT를 과소평가하게 된다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Tip: UltraBERT likes lower learning rates than other encoders, by up to 10×. On MNLI our best values were 1e-5 (150M) and 8e-6 (380M), where most encoders prefer 3e-5 to 5e-5."</blockquote></details>

```python
from transformers import AutoModelForSequenceClassification

model = AutoModelForSequenceClassification.from_pretrained(
    "dmis-lab/UltraBERT-150M", num_labels=2
)
```

공개 체크포인트에는 인코더와 5%(증류) MLM 헤드가 들어 있다. 블로그 공개 뒤 댓글로 요청이 들어와 50% 헤드도 별도 파일(`head_50%.pth`)로 추가됐다. 스파스 검색기를 학습하려는 사람에게 필요한 부분이다. 교체 방법은 [380M 모델 카드](https://huggingface.co/dmis-lab/UltraBERT-380M#what-is-in-this-checkpoint){:target="_blank"}에 있다.

## 도입 전에 따질 것

- **언어.** 영어 전용이다. 한국어 문서가 중심인 RAG라면 다국어 임베딩 모델을 계속 쓰는 편이 맞다. 영어 기술 문서, 코드, 영문 논문을 검색하는 경로라면 교체 후보로 올릴 만하다.
- **청킹 전략.** 32K까지 패스키 99%를 유지하므로, 짧게 자르던 청크를 길게 바꿨을 때 검색 품질이 어떻게 달라지는지 다시 재 볼 만하다. 다만 긴 입력은 그만큼 연산이 늘어나니 FlashAttention 2 환경에서 처리량을 함께 확인해야 한다. RRF로 BM25와 섞는 하이브리드 구성이라면 [RRF 튜닝 가이드]({{site.baseurl}}/tools/2026/08/17/rrf_tuning_agent_retrieval.html)의 절차대로 가중치를 다시 맞추는 것이 안전하다.
- **분류기·가드레일.** 저자들이 결론에서 기대하는 쓰임새 가운데 하나가 대규모 LLM 트래픽을 걸러내는 가드레일이다. 정해진 레이블 중 하나를 고르는 판정이라면, LLM 호출 대신 파인튜닝한 150M 인코더로 비용을 크게 줄일 수 있다.
- **수치는 저자 측정값이다.** 벤치마크 비교는 저자들이 같은 조건으로 돌린 결과다. 블로그 댓글에서 Hugging Face의 Tom Aarsen이 MS MARCO 3.2만 트리플릿으로 소규모 비교를 해 봤는데, 같은 방식으로 파인튜닝한 UltraBERT-150M이 ModernBERT-large(~400M)와 Ettin-400M을 앞섰다고 남겼다. 그래도 교체 판단은 내 데이터로 [RAGAS 같은 평가]({{site.baseurl}}/dev/2026/08/29/ragas-evaluation-methodology.html)를 돌려 보고 내리자.

## 참고

- [UltraBERT: One Encoder for Classification and Retrieval (Hugging Face 블로그, 원문)](https://huggingface.co/blog/dmis-lab/ultrabert){:target="_blank"}
- [UltraBERT: Beyond the Limit of BERT Pre-Training (arXiv 2610.15028)](https://arxiv.org/abs/2610.15028){:target="_blank"}
- [dmis-lab/ultrabert 컬렉션](https://huggingface.co/collections/dmis-lab/ultrabert){:target="_blank"}
- [UltraBERT-Embed-150M 모델 카드](https://huggingface.co/dmis-lab/UltraBERT-Embed-150M){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/Ph4ZJrwf4x8){:target="_blank"}

