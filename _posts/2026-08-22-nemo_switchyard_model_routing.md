---
layout: post
comments: true
title: "NVIDIA NeMo Switchyard — 에이전트 워크로드를 여러 모델에 라우팅하기"
description: "NVIDIA 테크니컬 블로그의 NeMo Switchyard 해설. 단일 프론티어 모델 대신 특화 모델 조합으로 에이전트 요청을 라우팅하는 System of Models 접근, 4가지 라우팅 알고리즘, LangChain·Cognition 벤치마크(비용 74%/28% 절감)를 정리한다."
img: multi_agent_network.jpg
date: 2026-08-22 20:30:00 +0900
last_modified_at: 2026-08-22 20:30:00 +0900
tags: [nvidia, nemo-switchyard, model-routing, agentic-ai, llm-router, nemotron, cost-optimization] # add tag
related: llm
categories: tools
---

에이전트 시스템의 API 비용은 대부분 프론티어 모델 호출에서 나온다. 그런데 에이전트가 처리하는 요청의 상당수는 굳이 프론티어급이 필요 없다. NVIDIA가 이 간극을 겨냥해 내놓은 것이 **NeMo Switchyard**다. NVIDIA 테크니컬 블로그의 [원문](https://developer.nvidia.com/ko-kr/blog/route-ai-agent-workloads-across-models-with-nvidia-nemo-switchyard/){:target="_blank"}(2026-08-14, Tanay Varshney 외)을 해설한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 대조·정정 후 발행했다.)

<!--more-->

> **TL;DR:** NeMo Switchyard는 단일 모델 대신 특화 모델과 프론티어 모델을 조합하는 **System of Models** 접근으로 에이전트 요청을 라우팅한다. 튜닝 없이 쓰는 라우터 3종(LLM 분류기·스테이지·에스컬레이션)과 학습 기반 프리필 라우터 1종을 제공하고, 공급업체 독립적 SDK(switchyard-libsy) 위에서 동작한다. LangChain 딥 에이전트 평가(145개 멀티턴 작업)에서 Nemotron 3.5 Lightning ↔ Claude Opus 4.8 에스컬레이션 라우팅으로 **프론티어 전용 대비 비용 74% 절감**(프론티어 호출은 전체의 7%, 정확도 약 6포인트 트레이드오프), Cognition Devin의 FrontierCode 벤치마크에서는 Opus 5 ↔ Kimi K2.7 라우팅으로 **평균 비용 28% 절감**(정확도 차이 2.8포인트, 50.6% vs 53.4%)을 보고했다.

## 1. System of Models — 왜 라우팅인가

Switchyard의 전제는 "모든 요청에 가장 비싼 모델을 쓰는 것은 낭비"라는 것이다. 요청·세션·작업 단계별로 적합한 모델이 다르므로, 라우터가 요청을 분석해 모델 풀에서 최적 대상을 고른다. 핵심 설계는 공급업체 독립성이다 — SDK(`switchyard-libsy`)가 모델 타깃에 시맨틱 이름을 부여해 라우팅 로직을 특정 공급업체·엔드포인트에서 분리하고, 라우터는 세션 상태 유지(stateful)와 스테이트리스 모드를 모두 지원한다.

## 2. 라우팅 알고리즘 4종

튜닝 없이 바로 쓰는 라우터가 3종이다.

- **LLM 분류기**: LLM을 판별자로 사용해 후보 모델을 고른다. 세션 친화성을 유지해 매 턴 재분류하는 낭비를 막는다.
- **스테이지 라우터**: 에이전트 작업 단계(탐색 → 구현 등)를 툴 사용 패턴으로 감지해 단계별로 모델 역량을 다르게 할당한다.
- **에스컬레이션 라우터**: 저비용 모델로 시작하고, 지속적으로 어려움이 감지되면 고성능 모델로 승격한다.

학습이 필요한 조정형 라우터가 **프리필 라우터**다. LLM의 잔류 스트림(residual stream)에서 프리필 상태를 추출하고, 공유 트렁크 MLP로 각 후보 모델의 성공 확률을 예측한 뒤 비용·지연 시간과 혼합해 최적 모델을 고른다. 기술 상세는 [arXiv:2603.20895](https://arxiv.org/abs/2603.20895){:target="_blank"} 참고. 라우팅 풀의 각 모델에 대한 정확도 레이블 데이터로 오프라인 학습이 필요하다는 점이 진입 장벽이다.

## 3. 벤치마크

- **LangChain 딥 에이전트 평가**(145개 멀티턴 작업, 5회 실행): 에스컬레이션 라우터로 Nemotron 3.5 Lightning과 Claude Opus 4.8 사이를 라우팅한 결과 프론티어 전용 대비 **비용 74% 절감**. 프론티어 모델 호출은 전체의 7%에 그쳤고 정확도는 약 6포인트를 내줬다. [벤치마크 상세](https://www.langchain.com/blog/switchyard-agent-routing-benchmark){:target="_blank"}
- **Cognition Devin Desktop**([FrontierCode](https://cognition.com/frontiercode){:target="_blank"} Main): Opus 5와 Kimi K2.7 간 라우팅으로 평균 비용 3.11달러에 정확도 50.6%를 달성 — 프론티어 전용 대비 **비용 약 28% 절감**, 정확도 차이 2.8포인트(50.6% vs 53.4%).

비용 절감 폭과 정확도 트레이드오프는 워크로드와 모델 조합에 따라 달라지므로, 도입 전 자기 워크로드 기준의 벤치마크 구축이 선행되어야 한다. 라우터 자체의 지연 시간·인프라 오버헤드도 실측이 필요하다.

## 4. 생태계와 도입 포인트

LangChain, LiteLLM, Kong, Boomi, Cognition(Devin), Cadence, Siemens, Nous Research, Ramp, Classmethod 등과 사전 통합되어 있고, 참조 서버 구현은 OpenAI·Anthropic 호환 API를 제공한다. 코드는 [GitHub NVIDIA-NeMo/Switchyard](https://github.com/NVIDIA-NeMo/Switchyard){:target="_blank"}에 공개되어 있다.

온프렘 관점에서 흥미로운 지점은 공급업체 독립적 설계다. vLLM·KServe 같은 스택으로 여러 로컬 모델을 서빙하는 환경이라면 라우터를 앞단에 두어 "단순 조회는 소형 모델, 복잡한 추론만 대형 모델"로 분기하는 구성이 가능하다. 프론티어 API와 로컬 모델의 하이브리드 라우팅은 [프롬프트 캐싱]({{site.baseurl}}/dev/2026/08/05/prompt-caching-guide.html)과 함께 에이전트 운영비를 줄이는 대표적 레버가 될 것이다.
