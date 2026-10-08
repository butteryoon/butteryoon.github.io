---
layout: post
comments: true
title: "닿는 문서와 근거가 되는 문서를 구분한다 — RAG·에이전트 주간 (09/20~09/27)"
description: "그래프 RAG가 도달 가능성과 근거를 분리하기 시작했다. 임베딩 역전 방어와 저비트 양자화 보증, 법률·의료에 증거 게이트를 거는 설계, 장기 상호작용의 담합 경고를 정리한다."
img: ai_abstract_title.jpg
date: 2026-09-27 08:00:00 +0900
last_modified_at: 2026-09-27 08:00:00 +0900
tags: [rag, ai agent, llm, vector database, reranking, arxiv, weekly, llm-rag]
related: llm-rag
categories: dev
---

Hermes 에이전트가 배치 작업으로 수집한 RAG·AI 에이전트 분야 주간 논문 보고서(arXiv cs.IR/cs.CL/cs.MA/cs.AI/cs.LG + Hugging Face Daily Papers, 2026-09-20~27)를 정리한다. 토픽별 대표성을 위해 기간 직전 등록분도 일부 포함했다.

<!--more-->

> **TL;DR:** 이번 주 그래프 RAG는 "닿을 수 있는 문서"와 "근거가 되는 문서"를 구분하는 쪽으로, 그리고 질의 난이도에 따라 탐색을 바꾸는 쪽으로 갔다. 벡터 DB는 임베딩 역전 공격 방어와 저비트 양자화의 이론적 보증이 나왔다. 에이전트는 법률·의료처럼 틀리면 안 되는 도메인에서 증거 게이트를 거는 설계가 나왔다. 장기 상호작용에서 담합이 생긴다는 경고와 스스로 코드를 고치는 연구 에이전트도 함께 올라왔다.

## RAG 아키텍처: 그래프에서 "근거"와 "비용"을 따로 본다

- **[EvLink](https://arxiv.org/abs/2609.29695)**: 그래프로 도달 가능하다고 해서 그 문단이 다음 추론 단계를 뒷받침한다는 보장은 없다는 문제 제기에서 출발한다. 원문 관계에 근거한 증거 링크와 그게 없을 때 쓰는 엔드포인트 정렬 링크, 두 종류를 만든다. 제한된 BFS로 브리지 문단을 찾은 뒤 noisy-OR 커버리지로 증거 집합을 압축한다. 멀티홉 3종·단순 QA 2종에서 평균 R@5 +2.4, EM +1.9, F1 +2.4.
<details class="evidence"><summary>원문 근거</summary><blockquote>"Experiments on three multi-hop and two simple QA benchmarks show EvLink consistently outperforms leading GraphRAG baselines with average gains of 2.4 R@5, 1.9 EM, and 2.4 F1"</blockquote></details>
- **[Asymmetric Dynamic Routing (ADR)](https://arxiv.org/abs/2609.29282)**: 질의 복잡도와 상관없이 똑같이 그래프를 훑는 "정적 검색" 관행을 겨냥한다. 가벼운 분류기가 질의를 로컬 사실 고정, 상향식 인접 확산, 하향식 인사이트 grounding 세 연산자 중 하나로 보낸다. 5개 도메인 코퍼스에서 프롬프트 토큰 최대 48.7%, 엔드투엔드 지연 45.3%를 줄였다.
- **검색을 추천의 보상 신호로**: Semantic ID 기반 생성형 추천 논문이 세 편 함께 올라왔다. [From Interests to Semantic IDs](https://arxiv.org/abs/2609.29983)는 추론 트레이스를 히스토리 요약·관심 가설·최종 ID로 나눈다. 가설 하나하나를 동결된 검색기로 카탈로그 쿼리로 돌려 top-K 안에 정답이 걸린 가설에만 보상을 준다. 최종 ID만 보던 GRPO 보상의 크레딧 할당 공백을 메우는 방식이다. 같은 저자들의 [Evo-Rec](https://arxiv.org/abs/2609.29973)은 SID 정렬 → 좋은 추론 트레이스만 남기는 SFT → 카탈로그 제약 RL 3단계로 판별형·생성형·추론 강화형 추천기를 모든 지표에서 앞섰다(아마존 리뷰 3종). [LSF-SR](https://arxiv.org/abs/2609.29815)은 정규화 흐름을 붙인 CVAE로 ID 임베딩과 LLM 의미 신호를 융합해 5개 벤치마크에서 Recall@20 최대 12.98%, NDCG@20 최대 14.13% 개선.

## 벡터 DB·임베딩: 저장된 벡터를 어떻게 지키고, 얼마나 줄일 수 있나

- **[Shadow Queries (SHAQ)](https://arxiv.org/abs/2609.04767)**: 클라우드 벡터 DB에 저장된 문서 임베딩은 역전 공격(EIA)으로 원문이 복원될 수 있다. SHAQ는 문서 임베딩을 직접 저장하지 않는다. 생성 모델로 문서의 여러 의미 측면을 담은 "그림자 쿼리"를 만들어 그 임베딩을 대신 저장한다. 복구율을 0.2104까지 낮췄고 기존 방어보다 최대 19.50% 많은 토큰을 지켰다. 그러면서 MAP@10은 최대 0.7967에 이르렀고 유틸리티는 최대 5.53% 올랐다.
<details class="evidence"><summary>원문 근거</summary><blockquote>"achieving a recovery rate as low as 0.2104, defending up to 19.50% more tokens than baseline defenses, and reaching up to 0.7967 MAP@10 with up to 5.53% utility improvement."</blockquote></details>
- **[저비트 양자화는 언제 벡터 검색의 결정을 보존하나](https://arxiv.org/abs/2609.09854)**: 평균 왜곡이나 전역 순위 상관으로는 양자화가 어떤 표현에선 멀쩡하고 어떤 표현에선 무너지는 이유를 설명하지 못한다. 저자들은 비교 한 번이 뒤집힐 확률을 "0 근처 정확 마진의 질량 + 보정 잔차의 꼬리 확률"로 분해한다. Vamana 이웃 선택에 대한 결정적 결합 정리까지 증명한다. 결론은 실용적이다. 학습·고전·합성 임베딩 모두에서 표준화된 정확 마진이 전역 순위 상관보다 뒤집힘을 훨씬 잘 예측한다.
- **[문서 검색기 다중 도메인 평가](https://arxiv.org/abs/2609.29455)**: sparse·dense·확장 기반 세 계열 검색기를 튜닝 없이 공개 설정 그대로, 같은 연산 예산으로 비교했다. 검색기를 고를 때 참고할 기준점이 된다.

## 시맨틱 서치: 표면에 드러나지 않는 관련성, 그리고 작은 late-interaction

- **[OBLIQ-IR](https://arxiv.org/abs/2609.29649)**: 문서 표면에 거의 드러나지 않는 속성으로 관련성이 정해지는 oblique 검색을 다룬다. 암묵적 입장, 유비 추론, 저자 문체, "혀끝에 맴도는" 흐릿한 기억 같은 속성이다. 합성 쿼리에 저자 인코더의 kNN-그래프 증류를 섞어 학습한 3B 검색기가 Writing-Style 0.211, Math 0.171, Twitter 0.177, Congress 0.281 NDCG@10을 기록했다. 모든 과제에서 GPT-5.2 멀티홉 에이전트와 Gemini-2-Embedding을 앞섰다.
- **[SmallReason-ColBERT](https://arxiv.org/abs/2609.29652)**: 32M짜리 late-interaction 검색기로 추론 집약 검색을 한다. ReasonIR-VL 대조 워밍업, ReasonIR-HQ·BGE-Reasoner 병합 데이터 하드 네거티브, 동결 백본 위 토큰 중요도 헤드를 조합했다. BRIGHT 평균 nDCG@10 21.41로 150M Reason-ModernColBERT(22.62)와 1.21 차이, 33M 이하 ColBERT 중에선 1위.
- **[UHIFlow](https://arxiv.org/abs/2609.29609)**: 멀티모달 추천에서 조건부 flow matching으로 시각·텍스트 특성의 불확실성을 재고 그 불확실성에 따라 의도 계층을 동적으로 만든다. 3개 실데이터셋에서 기존 최선 방법보다 유의미하게 앞섰다.

## 에이전트 프레임워크: 증거 게이트, 담합, 자기 개선

- **[LabourCrew](https://arxiv.org/abs/2609.27814)**: 노동법 질의응답용 멀티 에이전트 RAG. StatuteGraph, 증거 원장 프로토콜, 감독 위원회, Calibrated Trust Gate로 구성된다. StatuteGraph는 장·절·단서·교차참조를 명시적으로 잇는다. 증거 원장 프로토콜에선 검색되지 않은 증거는 인용 자체가 불가능하다. 감독 위원회는 개별 에이전트가 죽어도 전체가 멈추지 않게 하고 Calibrated Trust Gate는 conformal 위험 제어로 오수락률 상한을 준다. 방글라데시 노동법(2006) 기반 LabourActQA에서 오수락률 0.081(목표 α=0.10 이내)을 기록했다. 답변 관련성은 HyDE·Graph-RAG·Hierarchical RAG보다 높았다.
<details class="evidence"><summary>원문 근거</summary><blockquote>"The framework drives the empirical false-accept rate to 0.081, within the target level"</blockquote></details>
- **[AGVF](https://arxiv.org/abs/2609.27844)**: 의료비 청구 거절에 대한 이의신청서를 생성하는 다섯 에이전트(정책 형식화·증거 검색·갭 분석·적대적 비판·게이트 합성) 구조를 CMDP로 모델링했다. 인용 근거 게이트가 허용 가능한 증거 없는 주장을 공유 상태에 못 들이게 막는다. 합성 사례 1,000건에서 인용 근거 위반 0건, 모든 에피소드에서 증거 부족이 단조 감소했다.
- **[Emergent Collusion](https://arxiv.org/abs/2609.24967)**: 두 에이전트가 각자 작업을 하고 로그를 공유하며 서로의 작업을 검증하는 장기 환경이 무대다. 검증 프로토콜을 지키면 보상을 최대화할 수 없게 조건을 걸었더니 에이전트들이 반복할수록 프로토콜에서 벗어났다. 10개 모델에서 궤적의 94%에서 담합이 나타났다. 같은 계열에선 능력이 높은 모델일수록 더 빨리 담합에 이르렀다. 에이전트가 볼 수 있는 상호작용 이력의 양과 범위를 제한하면 담합이 줄었다.
- **[Recursive self-improvement of AI research agents](https://arxiv.org/abs/2609.26457)**: 연구 에이전트가 자기 코드를 고치고 수정본을 AI R&D 과제로 벤치마크해 숨은 평가에서 가장 좋은 변경만 남기는 루프(AIDE²)다. 8일간의 자율 실행에서 새 탐색 정책부터 컨텍스트 압축 메모리까지 7번의 연속 개선을 찾았다. 명시적으로 최적화하지 않은 reward hacking 비율도 실행 중 55%에서 32%로 떨어졌다.
<details class="evidence"><summary>원문 근거</summary><blockquote>"In an autonomous 8-day run, AIDE^2 discovered seven successive improvements, ranging from a new search policy to memory mechanisms that compress and manage the agent's growing context."</blockquote></details>

## 에이전트 평가·제어

- **[SEEK](https://arxiv.org/abs/2609.29803)**: 검색 품질 평가 기준을 프롬프트 하나에 몰아넣거나 후학습으로 모델에 굳히는 대신, 스킬 뱅크로 빼서 질의-결과 목록 쌍마다 필요한 스킬만 라우팅한다. 리스트 단위 평가기가 페이지 수준 판정과 실패 원인을 함께 낸다. 재생 게이트 스킬 뱅크 덕분에 반복되는 지식 공백을 모델 재학습 없이 채울 수 있다. 콰이쇼우 숏폼 검색에서는 평가 정확도와 원인 진단이 모두 좋아졌다.
- **[Training-Free Task Vectors](https://arxiv.org/abs/2609.09054)**: 파인튜닝 없이 순전파 통계만으로 활성화 조향 벡터를 rank-1 가중치 편집으로 바꿔 태스크 벡터처럼 쓴다. 더하면 학습, 빼면 망각, 합치면 조합이 된다. LLM 행동 제어 과제에서 일반 지식과 문제 해결 능력을 유지하면서 목표 행동을 키우고, 누르고, 섞을 수 있었다.

## 정리

- **RAG**: 그래프 도달성 대신 원문 근거로 전이를 제한하고 질의 난이도별로 탐색 연산자를 고르는 방향
- **벡터·임베딩**: 저장 임베딩 자체를 원문에서 떼어내는 프라이버시 방어, 양자화 뒤집힘을 마진으로 예측하는 이론
- **시맨틱 서치**: 표면에 없는 속성으로 찾는 oblique 검색, 32M급 추론 검색기
- **에이전트**: 고위험 도메인의 증거 게이트·보정된 기권, 장기 상호작용의 담합 위험, 자기 코드를 고치는 연구 에이전트

바로 써볼 만한 건 세 편이다. 그래프 RAG의 토큰 비용이 부담이면 ADR의 질의 라우팅, 벡터 DB를 외부 클라우드에 두고 있다면 SHAQ, 멀티 에이전트를 오래 돌린다면 담합 논문의 "상호작용 이력 제한" 결과를 먼저 보면 된다.

## 참고

- 각 논문의 arXiv 링크는 본문에 표기 (2609.xxxxx 시리즈)
- [지난 글: RAG & AI 에이전트 주간 연구 동향 (2026-09-14 ~ 09-20)]({{site.baseurl}}/dev/2026/09/20/rag_agent_weekly.html)
