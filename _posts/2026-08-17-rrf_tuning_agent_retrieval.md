---
layout: post
comments: true
title: "RRF(Reciprocal Rank Fusion) 튜닝 가이드 해설 — 하이브리드 검색 가중치 자동 최적화"
description: "Google의 Agent Retrieval RRF 튜닝 가이드 핵심 정리. Search Parity Dilemma, 가중치 기반 RRF 수식(k=60 고정), Optuna 베이지안 가중치 스윕, VertexRanker 서버사이드 리스코어링, 검증셋 구성과 셀프호스팅 구현까지."
img: tools_title.jpg
date: 2026-08-17 23:22:00 +0900
last_modified_at: 2026-08-17 23:50:00 +0900
tags: [rrf, reciprocal-rank-fusion, hybrid-search, rag, retrieval, vertex-ranker, bm25, optuna] # add tag
related: llm
categories: tools
---

[Gemma 4 MTP 서빙 가이드]({{site.baseurl}}/tools/2026/08/15/gemma4_mtp_serving_guide.html)에서 모델 서빙 레이어를 다뤘다면, 이번에는 **리트리버 레이어**다. 하이브리드 검색(키워드 + 벡터)의 결과를 어떻게 합치고 그 가중치를 어떻게 자동 튜닝하는지, Google이 Developer 포럼에 공개한 **Agent Retrieval RRF 튜닝 가이드**를 재구성한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 대조·정정 후 발행했다.)

<!--more-->

> **TL;DR:** 점수 스케일이 다른 리트리버들을 가중 선형 합으로 섞는 것은 안티패턴이다 — RRF는 **순위(rank)만으로** 합친다. Google Agent Retrieval(구 Vector Search 2.0)의 RRF는 `Σ wᵢ/(k + rᵢ)` 형태로 **k는 60에 고정**되어 있고, 튜닝 레버는 **검색 스트림별 가중치(wᵢ)와 서브서치 top_k**다. 가중치는 감이 아니라 **Optuna 베이지안 스윕**으로 검증셋(NDCG@10 또는 RAG면 Hit Rate) 위에서 찾고, 최종 정밀도는 **VertexRanker**(`semantic-ranker-fast@latest`) 서버사이드 리스코어링으로 올린다.

## 1. 배경 — Search Parity Dilemma

벡터 검색 프로토타입은 10분이면 만든다. 그런데 프로덕션에 가면 **기존 키워드 검색의 정밀도를 못 따라가는** 현실에 부딪힌다 — 밀집 임베딩은 상품 SKU, 에러 코드, 사내 약어 같은 **정확 일치 뉘앙스를 씻어내 버리기** 때문이다. 가이드는 이를 Search Parity Dilemma라 부른다.

| 리트리버 | 강점 | 약점 | 점수 스케일 |
|----------|------|------|-------------|
| 키워드 (BM25/텍스트) | 정확한 용어·SKU·에러코드·약어 | 의미적 유사도, 동의어 | 비제한 (0~수십) |
| 밀집 벡터 (임베딩) | 의도·개념 매칭 ("beachwear"→"swimsuit") | 도메인 밖 정확 토큰 | -1~1 또는 0~1 |

두 점수를 `w₁·dense + w₂·sparse`로 정규화해 더하는 건 위험한 안티패턴이다 — 코퍼스를 갱신하거나 인덱싱을 손대는 순간 raw score 분포가 움직여서, 하드코딩한 가중치가 깨지고 수동 재튜닝의 무한 루프에 빠진다.

## 2. RRF 원리 — 점수를 버리고 순위로 투표한다

RRF는 민주적 투표에 비유된다. 두 친구(의미 검색, 키워드 검색)에게 점수 대신 **순위가 매겨진 top-N 목록**만 받아서 순위 위치로 합산한다. Agent Retrieval의 수식:

```
RRF(d) = Σᵢ  wᵢ / (k + rᵢ(d))
```

- `rᵢ(d)`: 검색 스트림 i에서 문서 d의 순위 (1부터 시작)
- `wᵢ`: 스트림별 가중치 — **여기가 튜닝 대상** (합이 1일 필요 없음, 상대 배율)
- `k`: 스무딩 상수. **플랫폼에서 60으로 고정** (표준 RRF 기본값)

k가 없으면 한 목록에서만 1등인 아웃라이어가 두 목록 모두에서 2등인 합의(consensus) 후보를 이겨버린다. k=60이 순위 감쇠 곡선을 눌러 "양쪽 상위에 꾸준히 등장하는" 문서를 우대한다. 수동 계산 예(w=1, k=60):

| 문서 | 키워드 순위 | 벡터 순위 | RRF 점수 | 최종 |
|------|-----------|------------|---------------|-----------|
| C | 3 | 1 | 1/63 + 1/61 = 0.03226 | **1위** |
| A | 1 | 4 | 1/61 + 1/64 = 0.03177 | 2위 |
| D | 4 | 2 | 1/64 + 1/62 = 0.03151 | 3위 |
| B | 2 | — | 1/62 = 0.01613 | 4위 |
| E | — | 3 | 1/63 = 0.01587 | 5위 |

한쪽에서만 상위인 A·B보다 양쪽에 걸친 C가 올라온다 — 이것이 합의 효과다.

주의할 트레이드오프: RRF는 raw score의 **상대적 격차 정보도 함께 버린다**. 그래서 가이드는 상위 N 후보에 대해 VertexRanker로 2차 의미 리스코어링을 병행하라고 권한다(4장).

## 3. 튜닝 플레이북 — 가중치는 감이 아니라 스윕으로

### 필드 부스팅의 대체물: 병렬 서브서치

레거시 검색의 `title^3 OR body^1` 같은 필드 부스팅은 Agent Retrieval에 없다. 대신 **필드별 병렬 서브서치**(예: `name` 대상 텍스트 검색 + `category` 대상 텍스트 검색 + 의미 검색)를 던지고 각각의 **RRF 가중치를 튜닝**하는 방식으로 재현한다.

### 검증셋 구성

| 용도 | 검증셋 매핑 | 최적화 지표 |
|------|-----------|-----------|
| 검색 엔진 | 쿼리 → 등급 라벨 문서 ID (3=완벽 ~ 0=무관) | **NDCG@10** |
| RAG | 쿼리 → 정답 컨텍스트(참조 문서 ID) | **Contextual Recall / Hit Rate** (top-K 내 정답 소스 회수율) |

엔터프라이즈 기준: 실제 검색 로그에서 **최소 1,000개 이상의 쿼리**를 추출하고, 수동 등급 대신 CTR·장바구니 추가·전환 같은 **암묵적 피드백 신호**로 등급 행렬을 만든다. 쿼리 2개짜리 샌드박스 셋은 SDK 연결 확인용일 뿐이다.

### Optuna 베이지안 가중치 스윕

가이드는 `google-cloud-vectorsearch`(`vectorsearch_v1beta`)의 `BatchSearchDataObjectsRequest`로 서브서치 3개를 한 요청에 묶고, **Optuna**로 스트림별 가중치(0.0~2.0, step 0.2)를 50 트라이얼 베이지안 탐색하는 [전체 스크립트](https://discuss.google.dev/t/tuning-reciprocal-rank-fusion-in-agent-retrieval-a-practical-guide/378525){:target="_blank"}를 제공한다. 골격만 보면:

```python
def objective(trial):
    dense_w    = trial.suggest_float("dense_weight", 0.0, 2.0, step=0.2)
    name_w     = trial.suggest_float("name_text_weight", 0.0, 2.0, step=0.2)
    category_w = trial.suggest_float("category_text_weight", 0.0, 2.0, step=0.2)
    # BatchSearchDataObjectsRequest: 서브서치 3개 + RRF(weights) + VertexRanker를 한 요청에
    ...
    return mean_ndcg_at_10  # 검증셋 평균

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=50)
```

실전에서 최적 가중치는 `[1.0, 1.0]` 기본값에 머무는 일이 드물다:

- **SKU·모델번호·약어가 많은 쿼리** → 희소(키워드) 가중치 상향 (예: `[1.0, 1.6]`)
- **대화형·장문 쿼리** → 희소 가중치 하향 (예: `[1.0, 0.6]`) — 의미 매칭 우선

## 4. VertexRanker — 서버사이드 리스코어링

RRF의 순위 안정성은 얻었지만 정밀도를 더 올리려면 cross-encoder로 상위 후보를 재평가해야 한다. Agent Retrieval은 검색 요청에 **`semantic-ranker-fast@latest`** 모델을 지정하면 **서버 쪽에서** 쿼리-본문 리스코어링을 수행한다 — 후보를 클라이언트로 가져와 애플리케이션 메모리에서 리랭킹하는 왕복 지연과 코드 복잡도가 사라져 TTFT(첫 토큰까지 시간)를 지킬 수 있다.

| 단계 | 방식 | 역할 |
|------|------|---------|
| 1차 퓨전 | RRF (서브서치 N개, 가중치 wᵢ) | 광역 리콜 — 스케일 불일치 없이 합의 후보 확보 |
| 2차 리스코어 | VertexRanker (cross-encoder, 서버사이드) | top-N 정밀 랭킹 — LLM 컨텍스트 주입용 |

cross-encoder를 전체 코퍼스에 돌리지 않고 RRF가 좁힌 후보에만 적용하므로 비용·지연이 통제된다.

## 5. 셀프호스팅이라면 — 클라이언트사이드 RRF 구현

가이드는 GCP Agent Retrieval 전제지만, RRF 자체는 [2009년 논문](https://doi.org/10.1145/1571941.1572114){:target="_blank"} 이후 표준 기법이라 온프레미스 스택(Elasticsearch + Qdrant 등)에서도 몇 줄로 구현된다. 참고로 **플랫폼 밖에서는 k도 조정 가능한 파라미터**지만, 문헌 표준값이 60이고 대부분 그대로 쓴다 — 튜닝 효과가 큰 쪽은 여기서도 가중치다.

```python
def reciprocal_rank_fusion(ranked_lists, weights=None, k=60):
    """ranked_lists: 리트리버별 문서 ID 리스트(순위순). weights: 스트림별 가중치."""
    weights = weights or [1.0] * len(ranked_lists)
    scores = {}
    for w, ranked in zip(weights, ranked_lists):
        for rank, doc_id in enumerate(ranked, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + w / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)
```

운영 팁: 각 리트리버에서 가져오는 후보 수(retrieval_k)는 최종 top_k의 **5~10배**로 잡아 RRF가 합의할 후보 풀을 확보하고, 정밀도가 더 필요하면 상위 후보에 [bge-reranker 같은 오픈 cross-encoder]({{site.baseurl}}/dev/2026/08/09/rag_agent_weekly.html)를 클라이언트사이드 2차 랭커로 붙이면 VertexRanker와 같은 구도가 된다.

## 마무리

가이드의 결론은 "검색 relevance는 일회성 설정이 아니라 **데이터 기반 엔지니어링 규율**"이라는 것. 체크리스트로 요약하면:

1. **가중 선형 합 금지** — 스케일이 다른 점수를 정규화해 더하는 순간 재튜닝 루프에 갇힌다
2. **RRF로 순위 기반 퓨전** — Agent Retrieval에서 k는 60 고정, 튜닝 레버는 **스트림 가중치와 서브서치 top_k**
3. **가중치는 Optuna 스윕으로** — 검증셋(검색: NDCG@10, RAG: Hit Rate) 위에서 베이지안 탐색
4. **검증셋이 출발점** — 로그에서 1,000+ 쿼리, 암묵적 피드백으로 등급화
5. **정밀도는 2차 리스코어링으로** — VertexRanker 서버사이드(또는 셀프호스팅 cross-encoder)

## 참고

- [Tuning Reciprocal Rank Fusion in Agent Retrieval (Google Developer 포럼 원문)](https://discuss.google.dev/t/tuning-reciprocal-rank-fusion-in-agent-retrieval-a-practical-guide/378525){:target="_blank"}
- [RRF 원논문 — Cormack, Clarke & Buettcher (SIGIR 2009)](https://doi.org/10.1145/1571941.1572114){:target="_blank"}
- [Vertex AI Ranking API 문서](https://cloud.google.com/generative-ai-app-builder/docs/ranking){:target="_blank"}
- [Azure AI Search의 RRF 하이브리드 랭킹](https://learn.microsoft.com/en-us/azure/search/hybrid-search-ranking){:target="_blank"}
- [Redis: Reciprocal Rank Fusion 해설](https://redis.io/blog/reciprocal-rank-fusion/){:target="_blank"}
- [Gemma 4 MTP 서빙 가이드 (관련글)]({{site.baseurl}}/tools/2026/08/15/gemma4_mtp_serving_guide.html)
- [RAG & AI 에이전트 주간 동향 (관련글)]({{site.baseurl}}/dev/2026/08/09/rag_agent_weekly.html)
