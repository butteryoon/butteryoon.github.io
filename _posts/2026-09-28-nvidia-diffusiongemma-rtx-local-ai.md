---
layout: post
comments: true
title: "DiffusionGemma를 RTX·DGX Spark에서 가속 — 단일 사용자 추론 약 4배"
description: "DiffusionGemma는 자기회귀 대신 디퓨전 방식으로 단계마다 최대 256개 토큰을 병렬로 생성한다. NVIDIA는 H100에서 초당 1,000토큰, DGX Spark에서 150토큰, DGX Station에서 최대 2,000토큰을 제시하며 동급 자기회귀 모델 대비 약 4배."
img: diffusiongemma_rtx_title.webp
date: 2026-09-28 20:20:00 +0900
last_modified_at: 2026-09-28 20:20:00 +0900
tags: [nvidia, diffusiongemma, google-deepmind, rtx, dgx-spark, inference-optimization, llm-serving, local-ai]
related: local-ai
categories: [nvidia-blog, llm-serving, inference-optimization]
source_url: https://blogs.nvidia.co.kr/blog/rtx-ai-garage-local-gemma-diffusion/
source_date: 2026-06-12
---

NVIDIA 코리아 블로그의 RTX AI Garage 글 [「NVIDIA 기술로 더 빨라지는 Google DeepMind DiffusionGemma 기반 로컬 AI」](https://blogs.nvidia.co.kr/blog/rtx-ai-garage-local-gemma-diffusion/){:target="_blank"}(2026-06-12)를 읽고 정리했다. 토큰을 하나씩 뽑지 않고 블록 단위로 한꺼번에 다듬는 디퓨전 언어 모델이 로컬 GPU에서 얼마나 빨라지는지가 핵심이다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Google DeepMind의 DiffusionGemma는 Gemma 4 기반 26B MoE(단계당 3.8B 활성) 모델에 디퓨전 헤드를 붙여, 단계마다 최대 256개 토큰의 노이즈를 병렬로 걷어내며 텍스트를 만든다. Apache 2.0 오픈 웨이트다. NVIDIA에 따르면 단일 H100에서 초당 1,000토큰, DGX Spark에서 150토큰, DGX Station에서 최대 2,000토큰이 나온다. 같은 단일 사용자 환경의 동급 자기회귀 모델보다 약 4배 빠르다. Hugging Face Transformers·vLLM·Unsloth가 기본 지원하며, GeForce RTX용 llama.cpp 지원은 추가될 예정이다.

## 1. DiffusionGemma — 자기회귀를 벗어난 텍스트 생성

지금 쓰이는 LLM 대부분은 자기회귀(autoregressive) 방식이다. 새 토큰이 앞 토큰에 의존하니 한 번에 하나씩 순서대로 만들 수밖에 없고, 대화형 AI가 글자를 타이핑하듯 답하는 것도 그래서다. DiffusionGemma는 이미지 디퓨전 모델처럼 노이즈에서 출발해 텍스트 블록 전체를 한꺼번에 정제한다.

- **병렬 생성**: 토큰을 하나씩 예측하지 않고, 단계마다 최대 256개 토큰에서 노이즈를 제거한다.
- **Gemma 4 기반**: 260억 파라미터 전문가 혼합(MoE) 모델로 단계마다 38억 파라미터를 활성화하며, Gemma 4 아키텍처에 디퓨전 헤드를 결합했다.
- **오픈 로컬 실행**: Apache 2.0 오픈 웨이트라 RTX와 DGX Spark에서 완전히 로컬로 돌릴 수 있다. 클라우드도 토큰당 비용도 필요 없다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"디퓨전젬마는 젬마 4를 기반으로 구축됐는데요. 이는 260억 개의 파라미터를 갖춘 전문가 혼합(MoE) 모델로 단계마다 38억 개의 파라미터를 활성화하며, 구글의 젬마 4 아키텍처에 디퓨전 헤드를 결합합니다."</blockquote></details>

원문은 이 구조가 병목의 성격을 바꾼다고 설명한다. 토큰을 하나씩 생성하면 GPU는 연산보다 메모리 대역폭을 기다리는 데 시간을 더 쓴다. 256개 토큰 블록을 트랜스포머에 병렬로 통과시키면 워크로드가 컴퓨팅 성능에 좌우되고, 이 지점에서 Tensor 코어의 대규모 병렬 연산이 힘을 쓴다. CUDA 스택은 별도 튜닝 없이 출시 시점부터 이 모델을 효율적으로 실행한다고 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"토큰을 한 번에 하나씩 생성하는 방식은 본질적으로 메모리 병목형 문제인데요. 기존 LLM은 대부분의 시간을 연산 수행이 아닌 메모리 대역폭 대기에 사용하기 때문에 컴퓨팅 자원을 충분히 활용하지 못합니다."</blockquote></details>

## 2. NVIDIA 플랫폼별 성능

원문 본문에 나온 수치는 세 가지다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"디퓨전젬마는 단일 NVIDIA H100 Tensor 코어 GPU에서 초당 1,000개 토큰, NVIDIA DGX Spark에서 초당 150개 토큰, NVIDIA DGX Station에서 최대 초당 2,000개 토큰의 성능을 제공합니다. 이는 동일한 단일 사용자 환경에서 실행되는 동급 자기회귀 모델 대비 약 4배 빠른 수준이죠."</blockquote></details>

| 플랫폼 | 원문 설명 | 초당 토큰(단일 사용자) |
|--------|----------|----------------------|
| DGX Station | 748GB 통합(coherent) 메모리 | 최대 약 2,000 |
| H100 (단일 GPU) | H100 Tensor 코어 GPU | 약 1,000 |
| DGX Spark | GB10 Grace Blackwell 슈퍼칩, 128GB 통합 메모리 | 약 150 |
| RTX PRO 6000 워크스테이션 | 로컬 저지연 생성·에이전틱 루프를 돌릴 성능 여유 | 수치 없음 |
| GeForce RTX | llama.cpp 지원 추가 예정 | 수치 없음 |

원문은 이런 병렬 처리가 대화형 채팅, 에이전틱 루프, 계획과 실행을 오가는 온디바이스 어시스턴트처럼 지연 시간에 민감한 단일 사용자 작업에 잘 맞는다고 본다.

## 3. 로컬에서 시작하는 경로

| 도구 | 용도 | 참고 |
|------|------|------|
| Hugging Face Transformers | 가장 빠른 테스트·프로토타이핑 | GeForce RTX 5090·DGX Spark에서 별도 설정 없이 실행, 모델 `nvidia/diffusiongemma-26B-A4B-it-NVFP4` |
| vLLM | 더 높은 처리량의 추론 | [DGX Spark](https://build.nvidia.com/spark/vllm){:target="_blank"}·[RTX PRO](https://build.nvidia.com/rtx/vllm){:target="_blank"}·[DGX Station](https://build.nvidia.com/station/vllm){:target="_blank"}용 플레이북 |
| Unsloth, NVIDIA NeMo | 작업·도메인 맞춤 파인튜닝 | — |
| llama.cpp | GeForce RTX GPU 실행 | 지원 추가 예정 |

사전 구성된 DGX Spark 플레이북도 있다. 설치 없이 먼저 써 보고 싶다면 [build.nvidia.com](https://build.nvidia.com/){:target="_blank"}에서 NVIDIA가 호스팅하는 API로 무료 테스트할 수 있다.

## 4. 함께 소개된 RTX AI Garage 소식

- **SANA-WM**: NVIDIA 연구진이 공개한 오픈소스 월드 모델이다. 이미지 한 장과 카메라 경로만으로 정밀한 6-DoF 제어가 되는 720p·1분 길이 비디오를 만든다. 26억 파라미터 증류 버전은 NVFP4 포맷으로 GeForce RTX 5090 한 장에서 60초 분량을 34초에 생성하고, 비슷한 오픈 모델보다 처리량이 최대 36배 높다. ([논문](https://arxiv.org/pdf/2605.15178){:target="_blank"})
- **Windows 에이전트 환경**: NVIDIA와 Microsoft가 기본 Windows에서 쓰는 턴키 에이전트 샌드박싱을 공개했다. Microsoft 실행 컨테이너(eXecution Containers)와 NVIDIA OpenShell 런타임을 제공한다. 에이전틱 추론 속도는 최대 2배 높아졌고, Hermes Agent의 기본 Windows 지원도 추가됐다.
- **DGX Spark**: 간소화된 NVIDIA NemoClaw 설치로 로컬 에이전트를 바로 쓸 수 있고, Qwen3.6-35B는 vLLM에서 최대 2.6배 빨라졌다. NVIDIA Sync의 새 클러스터 어시스턴트는 DGX Spark를 최대 4대까지 512GB 풀 하나로 묶어 약 4,000억(400B) 파라미터 모델을 돌린다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA Sync의 새로운 클러스터 어시스턴트는 최대 4대의 DGX Spark를 하나의 512GB 풀로 연결해 약 4,000억 개 파라미터 규모의 모델을 실행할 수 있습니다."</blockquote></details>

## 5. 참고 자료

- [원문: NVIDIA 기술로 더 빨라지는 Google DeepMind DiffusionGemma 기반 로컬 AI](https://blogs.nvidia.co.kr/blog/rtx-ai-garage-local-gemma-diffusion/){:target="_blank"}
- [NVIDIA 테크니컬 블로그 — 아키텍처·로컬 배포 상세](https://developer.nvidia.com/blog/?p=118305){:target="_blank"}
- [Google DeepMind 발표 — DiffusionGemma](https://blog.google/innovation-and-ai/technology/developers-tools/diffusion-gemma-faster-text-generation/){:target="_blank"}
- [Hugging Face 모델 카드 — nvidia/diffusiongemma-26B-A4B-it-NVFP4](https://huggingface.co/nvidia/diffusiongemma-26B-A4B-it-NVFP4){:target="_blank"}
- [build.nvidia.com](https://build.nvidia.com/){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/TErYPw4o1KM){:target="_blank"}
