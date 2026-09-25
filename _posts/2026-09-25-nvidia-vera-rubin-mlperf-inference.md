---
layout: post
comments: true
title: "NVIDIA Vera Rubin NVL72, MLPerf Inference v6.1 첫 출전에서 최고 성능 기록"
description: "Vera Rubin NVL72가 MLPerf Inference v6.1 프리뷰 제출에서 GB300 NVL72 대비 Qwen3-VL 최대 3.7배, DeepSeek-R1 최대 2.5배 처리량을 냈다. GB300 NVL72는 4랙 288-GPU 구성에서 99% 확장 효율을 기록했다."
img: vera_rubin_mlperf_inference_title.webp
date: 2026-09-25 19:30:00 +0900
last_modified_at: 2026-09-25 20:35:00 +0900
tags: [nvidia, mlperf, inference, vera-rubin, gb300, ai-infrastructure, llm]
related: llm
categories: dev
source_url: https://blogs.nvidia.co.kr/blog/vera-rubin-nvl72-mlperf-inference/
source_date: 2026-09-21
---

NVIDIA 한국 블로그가 9월 21일 MLPerf Inference v6.1 결과를 정리해 올렸다. 처음 출전한 Vera Rubin NVL72가 프리뷰 제출에서 GB300 NVL72보다 Qwen3-VL 처리량이 최대 3.7배, DeepSeek-R1 처리량이 최대 2.5배 높게 나왔다. GB300 NVL72 쪽은 랙 4대(GPU 288개)로 늘렸을 때 확장 효율 99%를 찍었고 같은 하드웨어에서 소프트웨어 최적화만으로 v6.0보다 최대 1.6배 빨라졌다.

(이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Vera Rubin NVL72가 MLPerf Inference v6.1 프리뷰 제출에서 GB300 NVL72 대비 Qwen3-VL 최대 3.7배, DeepSeek-R1 최대 2.5배 처리량을 냈다. GB300 NVL72는 4랙 288-GPU에서 확장 효율 99%를 기록했고 소프트웨어 최적화로 v6.0 대비 최대 1.6배 성능이 올랐다. 파트너 19곳이 결과를 제출했고 그중 8곳은 멀티 노드 Blackwell NVL72 시스템을 썼다.

## 1. 원문 정보

- **원문 제목**: MLPerf Inference v6.1 첫 출전에서 최고 성능을 기록한 NVIDIA Vera Rubin NVL72
- **원문 URL**: [blogs.nvidia.co.kr/blog/vera-rubin-nvl72-mlperf-inference](https://blogs.nvidia.co.kr/blog/vera-rubin-nvl72-mlperf-inference/){:target="_blank"}
- **발행일**: 2026-09-21
- **작성자**: NVIDIA Korea
- **결과 기준**: MLPerf Inference v6.1 Closed Division, 2026년 9월 16일 www.mlcommons.org 확인 결과

## 2. 핵심 기술 내용

### Vera Rubin NVL72 데뷔 성능

NVIDIA는 v6.1 벤치마크 가운데 가장 까다로운 두 가지인 DeepSeek-R1과 Qwen3-VL에 Vera Rubin NVL72 프리뷰 결과를 냈다.

- **Qwen3-VL**: vLLM과 NVIDIA Dynamo를 써서 오프라인·서버·인터랙티브 시나리오 전반에서 GB300 NVL72 대비 **최대 3.7배** 처리량
- **DeepSeek-R1**: NVIDIA TensorRT-LLM을 써서 **최대 2.5배** 처리량

<details class="evidence"><summary>원문 근거</summary><blockquote>"Vera Rubin NVL72는 vLLM과 NVIDIA Dynamo 오픈 소스 추론 프레임워크를 활용해 Qwen3-VL에서 오프라인·서버·인터랙티브 시나리오 전반에 걸쳐 GB300 NVL72 대비 최대 3.7배 높은 처리량을 냈습니다. DeepSeek-R1에서는 NVIDIA TensorRT-LLM 라이브러리를 써서 처리량이 최대 2.5배 높았죠."</blockquote></details>

원문은 이 성능이 하드웨어와 소프트웨어를 함께 설계한 결과라고 설명한다.

- 향상된 텐서 코어와 트랜스포머 엔진이 프리필과 디코드 단계를 모두 가속한다.
- NVFP4 정밀도로 모델 가중치, 어텐션, KV 캐시의 메모리 사용량을 줄인다. 출력 품질 저하는 최소화하면서 처리량을 올리는 방식이다.
- 프리필과 디코드를 나누는 분리형 서빙에 대규모 전문가 병렬화(EP)를 더해 DeepSeek-R1·Qwen3-VL의 MoE 계층 효율을 끌어올렸다.
- NVL72 스케일업 도메인은 6세대 NVLink와 NVLink 스위치로 묶인다. 범용 이더넷보다 패킷 처리율은 10배 높고 지연 시간은 3배 낮다.

### GB300 NVL72 확장 효율

- **DeepSeek-R1 오프라인 시나리오**: 랙 1대(GPU 72개)에서 4대(GPU 288개)로 늘렸을 때 **확장 효율 99%**. 처리량이 추가한 하드웨어에 거의 비례해 늘었다.
- **WAN 2.2 텍스트-투-비디오**: 720p 영상을 초당 0.65편, 편당 5.7초에 생성했다. 단일 노드보다 처리량은 9배 높고 지연 시간은 7.5배 낮다.
- NVIDIA가 꼽은 확장 비결은 세 가지다. 랙 안의 고대역폭·저지연 스케일업 인터커넥트, 랙 사이를 잇는 고대역폭 네트워킹, 노드 전반의 요청 오케스트레이션이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA의 DeepSeek-R1(DSR1) 제출 결과는 GB300 NVL72 랙 한 대(GPU 72개)에서 네 대(GPU 288개)까지 확장하며 오프라인 시나리오에서 99%의 확장 효율을 달성했습니다. 처리량이 추가한 하드웨어에 거의 비례해 늘어난 것이죠."</blockquote></details>

### 소프트웨어 최적화로 올린 성능

- **v6.0 → v6.1**: GB300 NVL72의 Qwen3-VL 성능이 **최대 1.6배** 올랐다. KV 캐시 정밀도를 낮추고, 커널 퓨전을 추가하고, 커널 자체를 손보고, vLLM·Dynamo로 분리형 서빙을 적용한 결과다.
- **v6.1 제출 마감 이후**: GPT-OSS-120B와 DLRMv3에서 추가 향상이 나왔다. 마감 뒤 측정한 수치라 MLCommons 검증은 아직 거치지 않았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"v6.1에서 GB300 NVL72의 Qwen3-VL 성능은 v6.0 결과 대비 최대 1.6배 향상됐습니다. KV 캐시 정밀도를 낮추고, 커널 퓨전을 추가하고, 커널 자체를 개선하고, vLLM과 NVIDIA Dynamo로 분리형 서빙을 적용한 것이 성과로 이어졌죠."</blockquote></details>

### 에이전틱 벤치마크와 엣지

- **SemiAnalysis AgentX**: 프리뷰 테스트에서 Vera Rubin NVL72가 GB300 NVL72보다 **30배** 높은 성능을 냈다. 여러 단계에 걸쳐 추론하고 계획하고 행동하는 에이전트 워크로드를 재도록 설계된 벤치마크다. AgentX 자체는 [와트당 에이전틱 AI 성능을 다룬 이전 글]({{site.baseurl}}/dev/2026/08/29/nvidia_vera_rubin_blackwell_agentx_perf_per_watt.html)에서 자세히 정리했다.
- **MLPerf Endpoints**: 곧 공개될 벤치마크로, 기존 처리량 벤치마크가 담지 못하던 에이전틱 추론 워크로드를 표준화된 방식으로 측정한다.
- **Edge-Agentic (Qwen3.6-27B 기반)**: 새로 생긴 이 벤치마크에 TensorRT Edge-LLM을 쓴 Jetson AGX Thor 결과를 냈다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"이런 변화를 담아내도록 설계된 SemiAnalysis AgentX 같은 벤치마크에서 Vera Rubin NVL72는 프리뷰 테스트 기준 GB300 NVL72보다 30배 뛰어난 성능을 기록했습니다."</blockquote></details>

### 파트너 생태계

- 파트너 **19곳**이 결과를 냈고 그중 **8곳**은 멀티 노드 Blackwell NVL72 시스템으로 제출했다.
- 참여사: ASUS, Azure, Cisco, CoreWeave, Crusoe, Dell Technologies, Fujitsu, Giga Computing, HPE, Inventec, Lambda, MiTAC Computing, Nebius, Oracle Cloud Infrastructure, Quanta Cloud Technology, Red Hat, ScitiX, Supermicro, Wiwynn
- Nebius는 Vera Rubin NVL72 프리뷰 결과도 따로 제출했다.

## 3. 심화 분석

### 추론 경제성의 세 축

원문은 AI 추론 경제성을 좌우하는 지렛대로 시스템 성능, 확장 효율, 지속적인 소프트웨어 최적화를 들고 세 가지 모두에 이번 결과를 근거로 댄다. 랙 한 대가 더 많은 토큰을 만들면 같은 랙으로 더 많은 사용자를 받을 수 있고 토큰당 비용은 내려간다는 논리다. 99%라는 확장 효율은 랙을 늘린 만큼 처리량이 거의 그대로 따라온다는 뜻이다. 원문 표현대로, GPU를 두 배로 늘렸는데 처리량이 몇 퍼센트만 오른다면 인프라 비용이 성능 이득을 앞질러 버린다.

### 하드웨어를 바꾸지 않고 오른 성능

같은 GB300 NVL72에서 v6.0과 v6.1 사이에 Qwen3-VL 성능이 최대 1.6배 올랐다는 점이 눈에 띈다. KV 캐시 정밀도, 커널 퓨전, 분리형 서빙은 모두 소프트웨어 쪽 변경이다. 제출 마감 뒤에도 GPT-OSS-120B와 DLRMv3에서 향상이 이어졌다고 하니, 이미 깔아 둔 클러스터도 소프트웨어 업데이트만으로 성능이 오를 여지가 있다. 다만 마감 이후 수치는 MLCommons 검증 전이다.

### 에이전틱 워크로드로 옮겨 가는 벤치마크

AgentX, MLPerf Endpoints, Edge-Agentic처럼 에이전트 워크로드를 겨냥한 벤치마크가 늘고 있다. 단순 처리량 벤치마크(최대 3.7배)와 AgentX 프리뷰(30배)의 격차가 이만큼 벌어진다면, 앞으로 인프라를 비교할 때 어떤 벤치마크를 기준으로 삼느냐가 결론을 크게 바꿀 수 있다. AgentX 30배는 MLPerf 공식 결과가 아닌 프리뷰 테스트 수치라는 점도 함께 봐야 한다.

## 4. 참고 자료

- 원문: [MLPerf Inference v6.1 첫 출전에서 최고 성능을 기록한 NVIDIA Vera Rubin NVL72](https://blogs.nvidia.co.kr/blog/vera-rubin-nvl72-mlperf-inference/){:target="_blank"}
- [MLCommons](https://mlcommons.org/){:target="_blank"} — MLPerf Inference v6.1 공식 결과
- [NVIDIA Vera Rubin 플랫폼 소개 (NVIDIA 한국 블로그)](https://blogs.nvidia.co.kr/blog/vera-rubin-full-production-agentic-ai-factory/){:target="_blank"}
- [Nebius MLPerf Inference v6.1 결과](https://nebius.com/blog/posts/mlperf-inference-v6-1-results){:target="_blank"}
- [TensorRT Edge-LLM, Jetson AGX Thor MLPerf Edge-Agentic 결과 (NVIDIA Developer)](https://developer.nvidia.com/blog/tensorrt-edge-llm-completes-the-mlperf-edge-agentic-benchmark-6-4x-faster-on-jetson-agx-thor/){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/VHmBX7FnXw0){:target="_blank"}
