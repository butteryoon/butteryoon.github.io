---
layout: post
comments: true
title: "RAG & AI 에이전트 주간 연구 동향 (2026-07-12 ~ 07-19)"
description: "이번 주 arXiv에 올라온 RAG, 벡터 검색, 리랭킹, 에이전트 프레임워크·평가 분야 주요 논문 요약"
img: ai_abstract_title.jpg
date: 2026-07-19 12:00:00 +0900
last_modified_at: 2026-07-19 12:00:00 +0900
tags: [rag, ai agent, llm, vector database, reranking, arxiv, weekly] # add tag
related: llm
categories: dev
---

Perplexity 배치 작업으로 수집한 RAG·AI 에이전트 분야 주간 논문 보고서(arXiv cs.IR/cs.CL/cs.MA/cs.AI + Hugging Face Daily Papers, 2026-07-12~19)를 요약해서 정리한다.

<!--more-->

> **TL;DR:** 이번 주 RAG 연구의 흐름은 "정적 파이프라인 → 학습된 검색 정책"으로 요약된다. RL로 검색 행동을 조율하는 agentic RAG, 도메인 특화 GraphRAG가 강세였고, 벡터 검색은 소형·고효율화, 리랭킹은 프로덕션 실용화, 에이전트 평가는 정확도 너머의 다차원 계측으로 성숙하는 중이다.

## RAG 아키텍처: 검색 정책을 "학습"하는 방향으로

- **[GRASP](https://arxiv.org/abs/2607.10463)**: 강화학습으로 에이전트가 semantic search·keyword search·문단 읽기 세 검색 행동을 상황에 맞게 골라 쓰도록 학습. 폭넓은 탐색은 semantic, 국소 검증은 문단 읽기, 개체 특정은 keyword라는 해석 가능한 정책이 나온다.
- **[NGM-RAG](https://arxiv.org/abs/2607.11159)**: 그래프 매칭(GNN)과 텍스트 매칭을 adaptive weighting으로 결합한 graph RAG. multi-hop QA에서 GraphRAG·LightRAG를 능가.
- **도메인 특화 GraphRAG 확산**: 바이오 데이터에 FAIR 원칙을 통합한 [FAIR GraphRAG](https://arxiv.org/abs/2607.11464), 자율주행 규제 문서로부터 시나리오 DSL을 생성하는 [Chat2Scenic](https://arxiv.org/abs/2607.14387), 그리고 양자화 1.7B 백본으로 18배 큰 모델급 multi-hop 성능을 스마트폰에서 내는 온디바이스 [SmartRAG](https://arxiv.org/abs/2607.14661)까지 — 그래프 RAG가 특정 도메인·기기로 파고드는 주였다.

## 벡터 검색·임베딩: "작지만 빠른" retriever

- **[Cluster with Auctions](https://arxiv.org/abs/2607.13728)** (Meta 계열): DB 파티셔닝과 query probing을 함께 학습해, DB와 query 분포가 다를 때 동일 recall에서 최대 4.7배 throughput.
- **[SHEAF](https://arxiv.org/abs/2607.12229)**: 그래프 ANN 검색에서 query별 난이도를 답변집합의 변화량(flux)으로 추정해 beam-width를 적응적으로 예측.
- **[Score-Only Distillation](https://arxiv.org/abs/2607.11465)**: 교사의 score 벡터만으로 0.6B 소형 dense retriever에 증류 — 성능 격차의 50%를 회복하면서 인코딩은 5~10배 빨라진다.
- **[Walmart의 EBR 파이프라인](https://arxiv.org/abs/2607.10096)**: hybrid hard-negative 채굴 + Warm-Start Distillation으로 프로덕션 A/B에서 NDCG@5 +7.34%, 총매출 +0.50%. 학술 기법이 매출 지표로 검증된 사례.

## 시맨틱 서치: LLM 리랭커의 실용화

- **[Tool-Adaptive LLM Reranker](https://arxiv.org/abs/2607.10555)**: 확신할 때는 툴 호출을 건너뛰고 불확실할 때만 외부 근거를 검색하는 cost-aware 리랭커 — pointwise급 throughput으로 SOTA.
- **[LLaMA 3 리랭커 증류](https://arxiv.org/abs/2607.11933)**: SFT+LoRA 후 4-bit 양자화한 8B 리랭커를 BM25+dense 하이브리드 RAG에 투입, RAGAS 지표 14~21% 개선.
- **[QuintoAndar 부동산 검색](https://arxiv.org/abs/2607.14835)**: 대화 맥락 기반 LLM 리랭킹을 프로덕션 A/B로 검증 — 클릭률 +5.3%. 리랭킹 연구가 벤치마크를 넘어 실서비스 지표로 이동 중이다.

## 에이전트 프레임워크: 툴 스케일링과 라우팅

- **[GRADE](https://arxiv.org/abs/2607.10836)**: multi-agent 시스템의 에이전트 선택·깊이·통신·가지치기를 4개의 경량 게이트로 통합 제어.
- **[계층형 오케스트레이션](https://arxiv.org/abs/2607.11138)**: 수백~수천 개 툴의 flat registry가 유발하는 컨텍스트 포화를, 능력 트리 + 스택 실행 + lazy loading으로 해결 — 비용이 전체 툴 수가 아닌 탐색 경로에 비례.
- **[ToolAnchor](https://arxiv.org/abs/2607.14145)**: 고정 toolset으로 후학습된 에이전트가 새 툴을 못 쓰는 "behavioral inertia"를 규명하고 counterfactual context 주입으로 해소.
- **[LLM Planning의 두 축](https://arxiv.org/abs/2607.11197)**: planning 능력이 사실 operational reasoning과 structural enumeration 두 역량으로 나뉘며, 후자는 모델 규모를 키워도 잘 늘지 않음을 IRT 분석으로 제시.

## 에이전트 평가: 정확도 하나로는 부족하다

- **[AgentCompass](https://arxiv.org/abs/2607.13705)**: Benchmark·Harness·Environment를 분리한 오픈소스 평가 인프라, 20개+ 벤치마크 기본 지원.
- **[OmniaBench](https://arxiv.org/abs/2607.14989)**: 1,431개 태스크의 범용 에이전트 벤치마크 — 최신 모델도 Pass@1 60% 미만.
- **새로운 평가 축들**: 미래 시점 의도 실행 능력을 재는 [PM-Bench](https://arxiv.org/abs/2607.12385)(최고 모델도 F1 65%), 에이전트가 디스크에 남기는 데이터를 지표화한 [AgentFootprint](https://arxiv.org/abs/2607.11149)(프레임워크 간 저장량 15.7배 차이), 부분 실행 벤치마크의 신뢰성을 검증한 [Replay 분석](https://arxiv.org/abs/2607.12338)(필요 표본이 벤치마크별 15~90%로 제각각).

## 정리

- **RAG**: 고정 파이프라인 대신 RL로 검색 행동 자체를 학습시키는 agentic RAG, 도메인·온디바이스 특화 GraphRAG
- **벡터 검색**: query 적응형 파라미터, 분포 인지 파티셔닝, 소형 모델 증류 — 효율이 최우선 과제
- **리랭킹**: cost-aware 툴 호출과 양자화로 지연시간-정확도 균형, 프로덕션 A/B 검증 사례 증가
- **평가**: prospective memory, 저장 footprint, 표본 충분성 등 단일 정확도를 넘어선 다차원 계측이 자리 잡는 중

RAG를 운영 중이라면 이번 주 논문 중 SmartRAG(온디바이스), Score-Only Distillation(소형 retriever), 리랭커 증류 세 편이 바로 적용해볼 만한 실용 아이디어에 가깝다.

## 참고

- 각 논문의 arXiv 링크는 본문에 표기 (2607.xxxxx 시리즈)
- [지난 글: turbovec 벡터 인덱스 분석]({{site.baseurl}}/dev/2026/07/18/turbovec.html)
- [지난 글: Caller/Executor 멀티 에이전트 패턴]({{site.baseurl}}/dev/2026/07/16/caller_executor_agent.html)
