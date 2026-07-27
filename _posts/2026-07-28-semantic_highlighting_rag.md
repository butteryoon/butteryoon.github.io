---
layout: post
comments: true
title: "RAG의 숨은 병목 — 시맨틱 하이라이트(의미 강조)로 검색과 설명의 불일치 해결"
description: "대부분의 RAG는 검색은 시맨틱인데 하이라이트는 키워드로 틀어진다. Zilliz가 공개한 semantic-highlight-bilingual-v1(BGE-M3 Reranker 기반 0.6B)이 검색-설명 불일치와 컨텍스트 노이즈를 어떻게 푸는지 실제 모델 카드와 벤치마크로 정리"
img: command-title.webp
date: 2026-07-28 00:43:00 +0900
last_modified_at: 2026-07-28 01:27:56 +0900
tags: [rag, semantic highlighting, zilliz, milvus, context pruning, llm, nous research] # add tag
related: llm
categories: dev
---

X의 [@akshay_pachaar 포스트](https://x.com/akshay_pachaar/status/2014688141970702358)에서 "RAG는 검색은 잘하는데 사용자 경험은 조용히 망가뜨린다"는 지적을 봤다. 핵심은 **검색은 시맨틱(의미)인데 하이라이트는 키워드(정확일치)라는 불일치**. 이 글은 그 주장을 받아, 실제로 Zilliz가 공개한 오픈소스 모델 [`zilliz/semantic-highlight-bilingual-v1`](https://huggingface.co/zilliz/semantic-highlight-bilingual-v1)과 Milvus 엔지니어링 블로그를 근거로 어떻게 푸는지 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** RAG에서 벡터 검색은 "의미"로 문서를 찾지만, 대부분의 UI는 여전히 키워드 매치로 하이라이트한다 — 그래서 "iPhone performance" 검색 시 A15 Bionic 칩 문서가 나와도 아무것도 강조 안 된다. Zilliz는 BGE-M3 Reranker v2 기반 **0.6B 인코더 모델**(`zilliz/semantic-highlight-bilingual-v1`, MIT)로 토큰 단위 relevance scoring → 문장 집계 → 임계값 필터링을 밀리초에 수행한다. 하이라이트뿐 아니라 **컨텍스트 프루닝(토큰 비용 70~80% 절감)** 으로 확장되며, EN/ZH 양언어 SOTA를 기록한다.

## 1. 문제 — 검색과 설명의 비대칭

벡터 검색은 쿼리와 문서를 임베딩해 *의미*로 매칭한다. 그런데 검색된 문서를 사용자에게 보여줄 때, 대부분의 시스템은 **키워드 하이라이트**로 회귀한다.

```
검색(retrieval):  시맨틱 — "iPhone performance" ↔ A15 Bionic 칩·벤치마크·무지연
하이라이트(show): 키워드 — "iPhone"·"performance" 글자가 있는 곳만 강조
```

문제: 검색된 문서는 정확히 답인데, 그 단어가 없으면 **아무것도 강조 안 됨**. 사용자는 3,000단어를 직접 훑어야 "왜 이 문서가 나왔나"를 안다. 결과: 신뢰 하락 → 도구 이탈. 개발자도 "검색이 틀린가, 하이라이트만 실패한가"를 맹목적으로 디버깅.

Milvus 블로그는 이걸 **RAG 노이즈/토큰 낭비** 문제로 확장한다. 잘 튜닝된 인덱스도 청크 단위로는 broad-relevant하지만, 청크 안 문장 중 실제 답은 소수다. 에이전트 워크플로우에서는 쿼리 자체가 다단계 추론 출력이라 불일치가 더 심해진다.

불일치가 에이전트에서 왜 더 심한지 보자. 에이전트의 검색 쿼리는 사용자 원문이 아니라 **추론·태스크 분해로 파생된 지시**다.
- 사용자: "최근 시장 동향 분석해줘"
- 에이전트 쿼리: "Q4 2024 가전 판매 데이터, YoY 성장률, 경쟁사 점유율 변화, 공급망 비용 변동 추출"
- 키워드 하이라이트는 "2024"/"sales data"만 잡고, 진짜 인사이트("iPhone 15 시리즈가 시장 회복 견인", "칩 공급 제약으로 비용 15% 상승")는 놓침

즉 에이전트일수록 하이라이트가 의미 기반이어야 한다 — 수많은 검색 결과에서 "진짜 유용한 문장"을 키워드 없이 골라내야 하기 때문.

## 2. 해결 — Semantic Highlighting

키워드 매치가 아니라 **쿼리와 의미 정렬된 텍스트 스팬**을 강조한다. "iPhone"/"performance" 대신 **"A15 Bionic chip", "benchmarks", "lag"** 를 강조. 단순 UX를 넘어 **context pruning**(문장 단위 relevance filtering)으로 쓰인다 — LLM에 가기 전에 노이즈 문장을 지운다.

### 왜 그냥 LLM 안 쓰나?
하이라이트는 **매 쿼리·문서마다** 발생한다. LLM 호출마다 지연/비용이 폭발하니, 밀리초 추론하는 **소형 전문 모델**이 필요하다.

## 3. 실제 모델 — zilliz/semantic-highlight-bilingual-v1

X 포스트는 수치를 과소표기("100만 샘플", "5시간")했으나, **실제 모델 카드/HF 블로그**는 더 정확하다.

| 항목 | 실제 스펙 (HF model card + Milvus blog) |
| --- | --- |
| 모델 | `zilliz/semantic-highlight-bilingual-v1` |
| 베이스 | `BAAI/bge-reranker-v2-m3` (BGE-M3 Reranker v2) 파인튜닝 |
| 파라미터 | **0.6B**, encoder-only, F32 |
| 컨텍스트 | 8192 tokens |
| 언어 | English + Chinese (auto-detect) |
| 라이선스 | **MIT** (상업 사용 가능) |
| 학습 데이터 | **500만+ 양언어 샘플** (EN: MS MARCO/NQ/GooAQ, ZH: DuReader/중국어 위키/mmarco_zh) |
| 학습 | 8×A100, 3 epoch, **약 9시간** |
| 다운로드 | 월 ~1,254 (공개 시점) |

### 동작 파이프라인 (실제)
```
1. 입력: [BOS] + Query + Context 를 하나의 시퀀스로 연결
2. 토큰 스코어링: 컨텍스트 내 각 토큰에 relevance 0~1 부여
3. 문장 집계: 토큰 점수를 문장별로 평균 → 문장 점수
4. 임계값 필터: threshold(기본 0.5) 이상 문장만 하이라이트/보존, 나머지 제거
```
즉 **단일 패스**로 컨텍스트를 한 번 순회하며 "어디가 주목할 부분인지" 점수화한다.

### 실제 사용 예시 (HF 카드 발췌)
```python
from transformers import AutoModel
model = AutoModel.from_pretrained(
    "zilliz/semantic-highlight-bilingual-v1", trust_remote_code=True)
question = "What are the symptoms of dehydration?"
context = """Dehydration occurs when your body loses more fluid...
Common signs include feeling thirsty and having a dry mouth...
Dizziness, fatigue, and headaches can indicate severe dehydration..."""
result = model.process(question=question, context=context,
                       threshold=0.5, return_sentence_metrics=True)
print(result["highlighted_sentences"])
# ['Common signs include feeling thirsty...',
#  'Dark yellow urine and infrequent urination are warning signs.',
#  'Dizziness, fatigue, and headaches can indicate severe dehydration.']
# sentence_probabilities: [0.017, 0.990, 0.002, 0.947, 0.0009, 0.972, 0.0009]
```
출력에 `compression_rate`(제거된 텍스트 비율)와 `sentence_probabilities`가 붙는다 — 디버깅/관측에 직결.

## 4. 왜 이 아키텍처인가

- **Encoder-only**: decoder보다 빠르고, 모든 토큰 위치 relevance를 병렬 스코어링. 프로덕션에선 속도가 모델 크기보다 중요.
- **토큰 단위 스코어링**: Naver의 [Provence](https://arxiv.org/abs/2501.16214) (ICLR 2025)에서 "청크를 고르는 것"이 아니라 "모든 토큰을 점수화"하는 framing 차용. 하이라이트와 자연스럽게 맞물림.
- **BGE-M3 Reranker v2 베이스**: 인코더 구조 + EN/ZH 최적화 + 8192 컨텍스트 + reranking 사전학습(관련성 판단과 정렬)이라 적합.

## 5. 학습 데이터 품질 — reasoning-as-quality-check

핵심 인사이트는 **라벨에 추론 과정을 붙인 것**. LLM(Qwen3-8B, 로컬 vLLM)이 문장 라벨을 낼 때 "왜 relevant/irrelevant한가" 짧은 reasoning을 같이 생성. 효과:

- **라벨 품질 상승**: reasoning이 self-check 역할 → 불안정/랜덤 라벨 감소
- **관측성**: 왜 선택됐는지 블랙박스가 아님
- **디버깅**: 프롬프트/도메인/어노테이션 중 어디 문제인지 즉시 파악
- **재사용**: 라벨링 모델을 바꿔도 reasoning trace는 감사/재라벨링에 유효

Open Provence의 방식(공개 QA + 소형 LLM 라벨링)을 베이스로 삼되, reasoning 부착으로 안정성을 확보. 데이터셋도 [HuggingFace에 공개](https://huggingface.co/zilliz/datasets).

## 6. 성능 — EN/ZH SOTA

4개 데이터셋(in/out-of-domain):
- EN multi-span QA: **multispanqa**
- EN out-of-domain Wiki: **wikitext2**
- ZH multi-span QA: **multispanqa_zh**
- ZH out-of-domain Wiki: **wikitext2_zh**

비교군: Open Provence 시리즈, Naver Provence/XProvence, OpenSearch semantic-highlighter. **모든 데이터셋에서 1위**, 그리고 더 중요하게는 **EN과 ZH 모두에서 유일하게 안정적** — 경쟁 모델은 영어만 되거나 중국어에서 성능 급락.

실측 효과(Milvus 블로그):
- **토큰 비용 70~80% 절감** (하이라이트 문장만 LLM 전달)
- 답변 품질 향상 (무관련 컨텍스트 감소)
- 해석성·디버깅성 개선

## 6b. 기존 솔루션의 한계 — 왜 이 모델인가

Zilliz는 공개 전 가용한 솔루션들을 평가했고, **프로덕션 RAG에 필요한 정밀도·지연·다국어·견고함을 갖춘 게 없어 직접 훈련**했다고 명시한다. 구체적 한계:

| 모델 | 치명적 약점 (실측) |
| --- | --- |
| **OpenSearch `opensearch-semantic-highlighter-v1`** | BERT 베이스, **컨텍스트 512 토큰** (≈400~500 영어 단어) — 긴 문서 truncating. **중국어 미지원**. In-domain F1 ≈0.72 → out-of-domain **0.46** 급락 |
| **Naver Provence** (영어 전용) | CC BY-NC 4.0 → **상업 사용 불가**. context pruning 목적(recall 중시)이라 highlighting(정밀도 중시)과 objective mismatch → 하이라이트가 넓고 노이즈 많음 |
| **Naver XProvence** (다국어) | 영어 성능이 Provence보다 떨어짐(용량 분산 trade-off), 중국어도 "간신히 쓸만" 수준. 라이선스 동일 제약 |

반면 `semantic-highlight-bilingual-v1`은 **8192 토큰 + EN/ZH SOTA + MIT** 로 이 세 가지를 동시에 풀었다. 즉 "가벼운 모델로 밀리초"라는 제약 아래, 정밀도·다국어·상업 라이선스를 다 잡은 유일한 선택지다.

## 6c. 대안 구현 — OpenSearch 방식

의미 하이라이트는 Zilliz 모델 외에도 OpenSearch 3.0(2025-07)에 자체 도입됐다. 통합 방식은 쿼리의 `highlight.type: semantic` + `model_id` 지정:

```json
POST /neural-search-index/_search
{
  "query": { "neural": { "text_embedding": { "query_text": "treatments for neurodegenerative diseases", "model_id": "<embed-model>" } } },
  "highlight": {
    "fields": { "text": { "type": "semantic" } },
    "options": { "model_id": "<semantic-highlighting-model-id>" }
  }
}
```

- 모델 배포: 클러스터 내 로컬(torchscript) 또는 **SageMaker 원격 GPU** (CPU 대비 **4.5배 빠름**)
- 하이라이트 문장은 `<em>` 태그로 래핑, 신경망/하이브리드 쿼리 모두 지원
- Zilliz 모델과 달리 OpenSearch는 자체 `opensearch-semantic-highlighter-v1`(512토큰/중국어 미지원)을 기본 쓰므로, 프로덕션에선 Zilliz 모델을 외부 endpoint로 붙이는 식

한편 Zilliz 모델 자체는 **Milvus와 Zilliz Cloud에 네이티브 API**로 내장돼, 검색 시 관련 문장을 자동 노출한다("why retrieved" 가시화). 별도 코드 없이 벡터 DB 계층에서 해결되는 구조다.

## 7. 우리 블로그 관점에서

- **[OKF 글]({{site.baseurl}}/tools/2026/07/25/hermes_okf_knowledge.html)**: 큐레이션된 정형 지식은 OKF, 대량 비정형은 벡터 RAG. 이 모델은 벡터 RAG의 약점(의미는 찾는데 설명/압축 못함)을 **하이라이트 + 프루닝**으로 보완하는 정확한 층위다.
- **[가드레일 Security 글]({{site.baseurl}}/tools/2026/07/25/hermes_guardrail_security.html)**: 하이라이트 모델 자체가 "의미 정렬 스팬"을 판정하니, RAG 출력의 **근거(faithfulness) 시각화** 계층으로 활용 가능 — 가드레일의 "출력이 문서 근거인가"를 검증하는 도구.
- **[AI Factory 글]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)**: NVIDIA가 요구한 동적 라우팅/지속 컨텍스트에서, 검색→하이라이트→프루닝까지의 **관측성**이 사용자 신뢰와 직결.
- **[에이전트 학습 메타글]({{site.baseurl}}/tools/2026/07/25/hermes_agent_learning.html)**: Zilliz가 "피드백(라벨에 reasoning 부착)을 데이터셋에 영속화해 모델 품질을 높인" 것도, 에이전트가 클로드 피드백을 스킬에 박는 것과 같은 **관측 가능한 학습 루프** 사례.

## 마무리

RAG의 병목은 검색 정확도가 아니라 **"왜 이 문서인가"를 사용자에게 보여주는 층위**다. Zilliz의 `semantic-highlight-bilingual-v1`은 0.6B 소형 모델로 밀리초 scoring → 하이라이트 + 컨텍스트 프루닝을 풀며, EN/ZH 양언어 SOTA와 70~80% 토큰 절감을 동시에 잡는다. X 포스트가 던진 "검색은 시맨틱인데 하이라이트는 키워드"라는 날카로운 지적은, 실제 모델 카드를 보면 "하이라이트를 넘어 프루닝까지"로 확장되는 깔끔한 해법이 된다. 다음 RAG 파이프라인을 설계할 땐, 검색과 **설명/압축**을 같은 의미 수준에서 맞출 것을 권한다.

## 참고

- [Zilliz Semantic Highlight Model (HuggingFace)](https://huggingface.co/zilliz/semantic-highlight-bilingual-v1){:target="_blank"}
- [How We Built a Semantic Highlighting Model (Milvus Blog)](https://milvus.io/blog/semantic-highlighting-model-for-rag-context-pruning-and-token-saving.md){:target="_blank"}
- [Provence: efficient and robust context pruning (arXiv:2501.16214, ICLR 2025)](https://arxiv.org/abs/2501.16214){:target="_blank"}
- [Open Provence (구현)](https://github.com/hotchpotch/open_provence){:target="_blank"}
- [OpenSearch Semantic Highlighting (대안 구현)](https://opensearch.org/blog/introducing-semantic-highlighting-in-opensearch/){:target="_blank"}
- [원본 X 포스트 @akshay_pachaar](https://x.com/akshay_pachaar/status/2014688141970702358){:target="_blank"}
- [OKF 지식 번들 (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_okf_knowledge.html)
- [가드레일 Security (관련글)]({{site.baseurl}}/tools/2026/07/25/hermes_guardrail_security.html)
- [NVIDIA AI Factory (관련글)]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)
