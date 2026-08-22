---
layout: post
comments: true
title: "NAVER Cloud의 Megatron Bridge 멀티모달 사전 학습 최적화 사례 해설"
description: "NVIDIA 테크니컬 블로그의 NAVER Cloud 사례 분석. HyperCLOVA X SEED Omni와 30B MoE VLM의 멀티모달 사전 학습에서 스토리지 I/O, 시퀀스 패킹, 비전 인코더, Context Parallelism 4개 병목을 최적화해 throughput을 baseline 대비 150.2%로 끌어올린 과정을 정리한다."
img: ai_abstract_title.jpg
date: 2026-08-22 20:20:00 +0900
last_modified_at: 2026-08-22 20:20:00 +0900
tags: [nvidia, megatron-bridge, energon, multimodal, vlm, hyperclova-x, moe, context-parallelism, naver-cloud] # add tag
related: llm
categories: tools
---

[NVIDIA AI Factory 설계 가이드]({{site.baseurl}}/dev/2026/07/30/nvidia_ai_factory_design_guide.html)에서 인프라 레이어를 봤다면, 이번에는 그 위에서 돌아가는 **대규모 멀티모달 학습 파이프라인**의 실전 사례다. NVIDIA 테크니컬 블로그에 올라온 NAVER Cloud의 [Megatron Bridge 멀티모달 사전 학습 가속 사례](https://developer.nvidia.com/ko-kr/blog/how-naver-cloud-accelerates-multimodal-pre-training-with-nvidia-megatron-bridge/){:target="_blank"}(2026-08-18, 김윤식·송가연·박찬우·이지현·소우진)를 해설한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 대조·정정 후 발행했다.)

<!--more-->

> **TL;DR:** NAVER Cloud는 HyperCLOVA X SEED Omni(8B/4B)와 실험용 30B MoE VLM의 멀티모달 사전 학습(MMPT)에서 **스토리지 I/O, 시퀀스 패킹, 비전 인코더, 분산 학습** 4개 병목을 순차적으로 최적화해, 30B MoE VLM 기준 vanilla Megatron Bridge 대비 **누적 throughput 150.2%**를 달성했다. 데이터는 100개 이상 소스, 2T+ 토큰 규모의 프로덕션 데이터이고, 클러스터는 B200(EP=8, TP/PP/CP=1, 시퀀스 8K, gradient accumulation 128)이다. 수정 사항 일부(PR #4784)는 업스트림에 반영됐다.

## 1. 배경 — 멀티모달 사전 학습의 병목은 LLM과 다르다

텍스트 전용 LLM 학습과 달리 VLM의 MMPT는 이미지 개수와 해상도 분포가 극단적으로 갈리는 데이터를 다룬다. 샘플마다 비전 인코더 연산량이 크게 달라서 GPU 간 부하 불균형이 생기고, 데이터 로딩·패킹·분산 전략 전반이 텍스트 기준 최적화와 어긋난다. NAVER Cloud는 이 문제를 [Megatron Bridge](https://github.com/NVIDIA-NeMo/Megatron-Bridge){:target="_blank"}(NeMo의 후속 학습 프레임워크)와 데이터 로더 [Megatron Energon](https://github.com/NVIDIA/Megatron-Energon){:target="_blank"} 위에서 네 단계로 풀었다.

## 2. 4단계 최적화

### 스토리지/I/O — +2.5%

JSONL offset indexing과 zip 패키징, 비동기 prefetching으로 데이터 로딩 경로를 정리해 2.5%를 얻었다. 별도로 CPU 메모리 누수를 수정해 data loader worker 수를 늘릴 수 있게 했다.

### 시퀀스 패킹 — +13.3%

Greedy bin-packing으로 패킹 효율 99.4%를 달성한 뒤, DP worker 간 **pack reordering**(비동기 all-to-all)으로 이미지 토큰이 몰린 pack을 재배치해 비전 인코더 연산을 worker 간에 균형화했다. 재배치 로직은 학습 루프를 건드리지 않는 얇은 wrapper로 구현했고, 통신은 background thread와 전용 CUDA stream으로 오버랩해 오버헤드를 숨겼다.

### 비전 인코더 — +23.9% (TP=1/EP=8 기준)

먼저 Hugging Face 인코더에 activation recomputation을 적용해 시퀀스 길이를 8K에서 16K로 늘렸고, 이어서 인코더를 **네이티브 Megatron 블록으로 재구현**해 Megatron의 최적화 커널과 병렬화 전략을 그대로 상속받았다. 개선 폭은 병렬 구성에 따라 TP=1/EP=8에서 23.9%, TP=4/EP=2에서 50.3%였다.

### 분산 학습 — CP 확장 시 +20%

긴 시퀀스를 위해 Context Parallelism을 켜면 이미지 토큰 분배가 다시 틀어진다. 이를 **Vision DP when CP**(CP 구간에서 비전 인코더만 DP로 처리)로 풀어 CP=2(24K 시퀀스) 확장 시 속도를 20% 회복했다. 이 과정에서 발견한 packing된 멀티모달 시퀀스의 CP gradient backpropagation 버그 수정은 PR #4784로 업스트림에 반영됐다.

## 3. 시사점

- **패킹 효율과 부하 균형은 별개 문제다.** 99.4% 패킹 효율을 달성해도 이미지 토큰 분포가 기울면 GPU가 논다. pack reordering 같은 부하 재분배가 따로 필요하다.
- **프레임워크 네이티브 재구현의 이득이 크다.** Hugging Face 인코더를 감싸 쓰는 것과 Megatron 블록으로 재구현하는 것의 차이가 구성에 따라 20~50%p에 달했다.
- **최적화가 학습 루프 밖의 wrapper로 격리**되어 있어 프레임워크 업그레이드 추적이 쉽다는 점도 프로덕션 관점에서 참고할 만하다.

100개 이상 소스, 2T+ 토큰이라는 프로덕션 규모에서 검증된 수치라는 점이 이 사례의 무게다. 멀티모달 학습·서빙 파이프라인을 설계한다면 [Megatron-LM](https://github.com/NVIDIA/Megatron-LM){:target="_blank"} 계열의 병렬화 문서와 함께 원문을 정독할 가치가 있다.
