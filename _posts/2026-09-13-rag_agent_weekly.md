---
layout: post
comments: true
title: "RAG & AI 에이전트 주간 연구 동향 (2026-09-07 ~ 09-13)"
description: "이번 주 arXiv에 올라온 RAG, 벡터 검색, 리랭킹, 에이전트 프레임워크·평가 분야 주요 논문 요약"
img: ai_abstract_title.jpg
date: 2026-09-13 23:00:00 +0900
last_modified_at: 2026-09-14 09:20:00 +0900
tags: [rag, ai agent, llm, vector database, reranking, arxiv, weekly] # add tag
related: llm
categories: dev
---

Hermes 에이전트가 배치 작업으로 수집한 RAG·AI 에이전트 분야 주간 논문 보고서(arXiv cs.IR/cs.CL/cs.MA/cs.AI/cs.LG + Hugging Face Daily Papers, 2026-09-07~09-13)를 요약해서 정리한다. 이번 주는 편수가 14편으로 지난주보다 적은 대신, 보고서가 논문마다 arXiv 초록 페이지를 직접 열어 서지를 대조했다고 밝혔다. 목표 기간 직전(8월 31일~9월 5일)에 등록된 논문도 일부 포함되어 있다.

<!--more-->

> **TL;DR:** RAG 쪽은 에이전틱 검색을 얼마나 싸게 돌리느냐가 화두였다. VikingRAG는 다중 라운드 검색 궤적을 재사용해 토큰을 5~32% 수준까지 눌렀고, Q2D-Web은 1.9억 문서 규모의 에이전틱 리트리벌 벤치마크를 내놨다. 검색 문서 셋 중 셋이 오염되면 Llama 3.1 8B 정확도가 77.9%에서 43.5%로 떨어진다는 측정도 나왔다. 에이전트 쪽에서는 "에이전트를 만드는 일" 자체를 벤치마크로 삼은 τ^τ-Bench(최강 구성이 23.9% 통과, 전문가 레퍼런스 82.2%), 하네스 생성을 평가하는 HarnessDev, 모델을 갈아타면 에이전트 메모리가 살아남는지 따진 이식성 연구가 눈에 띈다. 평가 쪽은 리더보드 순위 차이가 통계적으로 의미 있는지 되묻는 논문과, 무해한 문서를 검색해도 유해 답변이 늘 수 있다는 RAG-Safety-Bench가 나왔다.

## RAG 아키텍처: 에이전틱 검색의 비용과 신뢰

- **[VikingRAG](https://arxiv.org/abs/2609.11390)**: 구조화 문서용 RAG에서 디렉토리 인식 의미 데이터 관리로 구조 컨텍스트 토큰을 기존 방법의 11.6~51.9% 수준으로 줄이면서 SOTA 정확도를 유지한다. 에이전트가 여러 라운드 검색하며 남긴 궤적을 경험 에지로 굳혀 다음 질의에 재사용하고, 증거가 충분하면 1라운드에서 끝내는 적응형 에스컬레이션까지 켜면 토큰 비용이 5.1~32.5%까지 내려간다.
- **[Q2D-Web](https://arxiv.org/abs/2609.08887)**: 1.9억 문서 웹 코퍼스와 에이전트가 재작성한 7만 개 쿼리(10개 언어)로 만든 대규모 에이전틱 리트리벌 벤치마크다. 에이전트 인용, 프로덕션 랭킹, 결합 판정 세 종류 정답 세트를 제공하고, 13개 리트리버를 돌려 보니 도메인·언어·쿼리 유형에 따라 순위가 갈렸다.
- **[In RAG We Trust?](https://arxiv.org/abs/2609.09243)**: 검색된 문서가 오염됐을 때 RAG가 얼마나 무너지는지 Llama 3.1 8B로 측정했다. 엔티티 치환·숫자 치환·부정 세 가지 공격을 검색 문서 3개 중 0~3개에 적용했더니 정확도가 77.9%에서 43.5%로 떨어졌고, 오염 문서가 다수가 되는 순간 급락한다. 모델은 거짓을 새로 지어내기보다 답을 회피하는 쪽으로 반응했고, 어휘 겹침 기반 비지지 생성 프록시는 공격 하에서 오히려 줄었다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"Accuracy falls from 77.9% on clean context to 43.5% when all three passages are corrupted. Entity swap flips the largest share of answers that were correct on clean context. Number-based corruption stays flat while poisoned passages are a minority and jumps once they form a majority"</blockquote></details>
- **[Agent-Enhanced Heterogeneous Graph RAG](https://arxiv.org/abs/2609.00761)**: 학술 QA용 이종 그래프 RAG를 쿼리 인식 검색, 충분성 평가 리랭킹, 그래프 기반 검증 세 단계의 에이전틱 결정으로 나눴다. OpenAlex/DBLP 그래프에서 LLM 단독, 그래프 증강 RAG, 에이전트 베이스라인을 모두 앞섰다.
- **[ViSAR](https://arxiv.org/abs/2609.02486)**: 후기 상호작용 시각 문서 검색에서 쿼리 복잡도에 따라 k를 동적으로 정하는 학습 불필요 방식이다. 임베딩 공간의 유사도 행렬로 쿼리 관련 의미를 짚어내고, RAG 지연을 58.7% 줄이면서 정확도는 유지하거나 올렸다.

## 벡터 검색·임베딩: 임베딩 역공격 방어

- **[Shadow Queries (SHAQ)](https://arxiv.org/abs/2609.04767)**: 벡터 DB에 문서 임베딩을 그대로 두면 임베딩 역공격(EIA)으로 원문이 복원될 수 있다. SHAQ는 문서 임베딩 대신 의미를 분해한 섀도 쿼리를 생성해 저장한다. 복원율은 0.2104에 그쳤고 베이스라인보다 19.5% 더 많은 토큰을 지켰으며, MAP@10 0.7967로 검색 유틸리티가 오히려 5.53% 올랐다.

이번 주 벡터 검색 분야는 이 한 편이지만, 위 VikingRAG의 경험 에지 재사용과 적응형 에스컬레이션도 서빙 단계 토큰·지연 최적화라는 점에서 같은 결에 놓인다.

## 시맨틱 서치: LLM 리랭커는 후보 풀에 민감하다

- **[Retrieval, Scoring, and Decoding Shape Performance and Stability in LLM-based Conversational Recommendation](https://arxiv.org/abs/2609.00086)**: ReDial 벤치마크에서 LLM 리랭커 성능이 후보 풀 크기, 1단계 리트리버, 디코딩 온도에 크게 흔들린다는 점을 실증했다. 공통 의미론적 top-250 풀에서 상용 리랭커 NDCG@10은 0.1497, 비LLM 리랭커는 0.0939였다. 후보 없이 바로 생성하는 제로샷은 0.2925로 더 높게 나오지만 과대평가 위험이 있다고 본다. 후보 생성을 협업 필터링으로 바꾸면 50% 넘게 좋아졌다.

## 에이전트 프레임워크: 에이전트 만들기, 하네스 만들기, 메모리 옮기기

- **[τ^τ-Bench](https://arxiv.org/abs/2609.04611)**: 에이전트를 "쓰는" 게 아니라 "만드는" 일을 태스크로 삼았다. 개발자 에이전트에게 실제 비즈니스 레코드, 요구사항을 가진 클라이언트, 프로덕션 API, 물려받을 코드베이스, 서빙 비용·모델 제한을 주고 고객 서비스 에이전트를 완성하게 한 뒤 홀드아웃 시뮬레이션 사용자로 채점한다. 4개 도메인 53개 태스크에서 최강 구성인 Claude Opus 5 + Claude Code가 23.9%를 통과했고, 전문가가 쓴 레퍼런스는 82.2%였다. 레코드를 얕게만 조회하고, 클라이언트와 거의 소통하지 않고, 처음 돌아가는 설계를 그대로 내보내는 실패 양상이 사람 개발자와 닮았다고 한다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"Across 53 tasks spanning four domains, the strongest configuration, Claude Opus 5 under Claude Code, passes just 23.9% of evaluation simulations. Meanwhile, an expert-authored reference ceiling scores 82.2%."</blockquote></details>
- **[AgentBrew](https://arxiv.org/abs/2609.05837)**: 실제 환경에서 수집한 원시 궤적만으로, 태스크 검증자 없이 오프라인으로 툴 사용 정책을 학습한다. 회고적 태스크 추론으로 궤적에 지시문을 붙이고 PMI 기반 크레딧 할당으로 액션별 가중치를 만든다. GitHub/Notion/PostgreSQL MCP 앱에서 Qwen3-32B가 정확도 +8.7, 점수 +9.7 올라 Qwen3-235B와 리젝션 샘플링을 넘었다.
- **[Does Your Agent's Memory Survive a Model Upgrade?](https://arxiv.org/abs/2609.05339)**: 모델을 바꿔도 메모리 저장소는 그대로 두는 게 보통인데, 새 모델이 옛 노트를 다르게 읽거나 임베딩 버전이 섞여 검색이 깨질 수 있다. 통제 실험에서 고정 스키마 지식 그래프는 정확도 변화가 +0.0004±0.0020로 거의 없었고, 압축 노트는 업그레이드 방향에 따라 +9.91/-13.28%p로 비대칭 변동했다. RAG를 절반만 재임베딩하면 전체 재임베딩 이득 11.90%p 중 4.96%p만 회복된다. 원시 히스토리를 남겨 두면 48케이스 중 34개에서 복구가 됐다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"Model upgrades are routine; memory migrations are not. An agent can keep the same memory store and still forget: a new model may interpret old notes differently, mixed embedding versions may break retrieval, and repair may fail without the original evidence."</blockquote></details>
- **[HarnessDev](https://arxiv.org/abs/2609.01437)**: 에이전트 하네스(실행 인프라) 자체를 LLM이 만들고 고쳐 나갈 수 있는지 재는 벤치마크다. 최소 시드에서 완전한 실행 시스템을 만드는 생성 단계와 자기 하네스를 반복 개선하는 진화 단계로 나뉜다. 6개 LLM × 4개 도메인 × 5개 다운스트림 벤치마크(2,207 인스턴스)에서 코드·검색은 인간 레퍼런스에 못 미쳤고 쓰기·ML 실험은 동등하거나 넘었다. 진화 이득은 불안정하고 다른 태스크로 잘 옮겨가지 않았다.
- **[Latency-Aware Orchestration for Multi-Agent LLM Workflows](https://arxiv.org/abs/2609.03335)**: 이종 GPU 풀 위에서 다중 에이전트 워크플로의 물리 실행 그래프를 예측 기반 런타임으로 최적화한다. 디바이스별 활성화 지연·피크 메모리·모델 로드 비용을 예측해 의존성을 전파하고, 의미를 보존하는 퓨전과 모델 라이프사이클 대안을 라이브 풀 상태와 함께 공동 최적화한다. 3개 시나리오에서 메이크스팬 36.8%, p95 지연 25.9%가 줄고 세션당 24.63 GPU-s를 아꼈다.

## 에이전트 평가: 순위표를 의심하고, 안전성을 분리 측정한다

- **[What Does an LLM-Agent Leaderboard Rank Actually Compare?](https://arxiv.org/abs/2609.07785)**: 리더보드 순위가 "A가 B보다 낫다"는 쌍별 결론을 정말 정당화하는지 추정량 관점에서 따졌다. 비교 대상과 측정 소스를 명시하고, 공통 지지를 확인하고, 불확실성 규칙과 실용적 마진으로 판단하는 절차를 제안한다. SWE-bench/AgentRewardBench/τ²-bench에서 근소한 순위 차이는 대부분 미해결로 남았고, 어떤 프록시 라벨과 유틸리티 규칙을 쓰느냐에 따라 선택되는 시스템이 바뀌었다.
- **[RAG-Safety-Bench](https://arxiv.org/abs/2609.11758)**: RAG가 LLM 안전성에 미치는 영향을 리트리버 품질이라는 교란 요소를 걷어내고 측정한다. 비RAG, 유해 답변이 담긴 오라클 문서, 유해 관련 문서, 안전과 무관한 문서 네 조건으로 나눠 5개 오픈소스 LLM을 봤더니 무해·유해 능력이 역상관이었고, 베이스라인 가드레일이 RAG 하류 안전성을 보장하지 못했다. 안전과 무관한 문서라도 검색이 켜지는 것만으로 유해 생성이 유발될 수 있다는 점이 가장 불편한 결과다.

## 정리

- **RAG**: 고정 top-k를 에이전틱 다중 라운드로 바꾸되 비용은 궤적 재사용·조기 종료로 누른다(VikingRAG). 에이전틱 리트리벌 평가 인프라(Q2D-Web)와 오염 문서 강건성 측정(In RAG We Trust?)이 함께 나왔다
- **벡터 검색**: 문서 임베딩 대신 섀도 쿼리를 저장해 역공격을 막으면서 유틸리티까지 올린 SHAQ
- **리랭킹**: LLM 리랭커 점수는 후보 풀·1단계 리트리버·온도에 따라 널뛴다. 제로샷 생성 점수는 과대평가를 의심할 것
- **에이전트·평가**: 평가 단위가 태스크 해결에서 에이전트 구축(τ^τ-Bench)·하네스 생성(HarnessDev)·메모리 이식성으로 넓어졌고, 리더보드 순위와 RAG 안전성 측정 방법 자체를 다시 묻는다

RAG를 운영 중이라면 검색 문서 오염 시 성능 곡선을 미리 그려 보게 하는 In RAG We Trust?(2609.09243), 모델 교체 전 메모리·임베딩 이식성을 점검하게 하는 메모리 이식성 연구(2609.05339), 그리고 검색을 켠 뒤 안전성 회귀 테스트가 필요하다는 RAG-Safety-Bench(2609.11758) 세 편이 바로 써먹을 만한 체크리스트에 가깝다.

## 참고

- 각 논문의 arXiv 링크는 본문에 표기 (2609.xxxxx 시리즈)
- [지난 글: RAG & AI 에이전트 주간 연구 동향 (2026-08-31 ~ 09-06)]({{site.baseurl}}/dev/2026/09/06/rag_agent_weekly.html)
