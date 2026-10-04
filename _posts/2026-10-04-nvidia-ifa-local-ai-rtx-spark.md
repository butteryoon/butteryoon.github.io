---
layout: post
comments: true
title: "NVIDIA, IFA 2026서 로컬 AI 가속하는 신기술·생태계 공개"
description: "Hermes Agent·OpenClaw·Perplexity가 원클릭 로컬 모델 설정 제공, PAIR로 유휴 PC 자원 활용, RTX Spark 10월 출시"
img: nvidia_ifa_local_ai_title.webp
date: 2026-10-04 18:00:00 +0900
last_modified_at: 2026-10-04 18:00:00 +0900
tags: [nvidia, ifa-2026, local-ai, rtx-spark, pair, hermes-agent, llm-serving]
related: llm-serving
categories: [nvidia-analysis, llm-serving]
source_url: https://blogs.nvidia.co.kr/blog/local-ai-ifa-next-gen-agents-nv-pair-rtx-spark/
source_date: 2026-09-08
---

NVIDIA가 IFA 2026에서 프론티어 인텔리전스를 로컬 환경으로 넓히는 신기술과 생태계를 공개했다. 핵심은 세 가지다. Hermes Agent·OpenClaw·Perplexity Portable Computer가 Windows RTX PC에서 원클릭 로컬 모델 설정을 제공하고, NVIDIA PAIR(Personal AI Router)가 로컬 네트워크의 유휴 PC 자원을 AI 추론에 활용하며, 1페타플롭 RTX Blackwell GPU를 얹은 RTX Spark Windows PC가 10월 출시된다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<details class="evidence"><summary>원문 근거</summary><blockquote>"프론티어 인텔리전스가 로컬 환경으로 확장되는 가운데, NVIDIA는 Microsoft를 비롯한 파트너사와 함께 NVIDIA 하드웨어에서 에이전트를 보다 쉽게 설정하고 로컬에서 실행할 수 있도록 빠른 추론 성능과 새로운 도구를 제공하기 위해 협력하고 있습니다."</blockquote></details>

<!--more-->

> **TL;DR:** NVIDIA가 IFA 2026에서 로컬 AI 생태계의 세 축을 내놨다. (1) Hermes Agent·OpenClaw·Perplexity의 원클릭 로컬 모델 설정, (2) PAIR로 로컬 네트워크의 유휴 PC 자원 분산 활용, (3) 1페타플롭 RTX Blackwell GPU, 최대 128GB 통합 메모리, 20코어 Grace CPU를 갖춘 RTX Spark의 10월 출시. llama.cpp는 최대 1.9배, vLLM은 최대 1.4배 빨라져 로컬 에이전트의 반응성을 확보했다.

## 1. 원클릭 로컬 에이전트 설정: Hermes Agent·OpenClaw·Perplexity

로컬 모델로 에이전트를 돌리려면 모델 선택, 호환 추론 서버 탐색, 양자화 설정 조정, 구성 요소 최신화까지 손이 많이 갔다. NVIDIA는 이 과정을 Windows에서 세 가지 주요 에이전트 앱으로 단순화했다.

- **Hermes Agent** (Nous Research 개발): 모델과 제공업체에 구애받지 않는 범용 에이전트다. RTX PC·RTX PRO Workstation·DGX Spark에서 쓸 수 있고, 원클릭 설정을 제공한다. NVIDIA GPU를 자동으로 감지해 알맞은 모델과 구성을 고른 뒤, NVIDIA 추론 최적화가 적용된 통합 llama.cpp로 실행한다. Linux 지원은 추후 제공된다.
- **OpenClaw:** GitHub에서 38만 개 이상의 스타를 받은 최대 규모의 오픈 에이전트 프로젝트다. NVIDIA·Microsoft와 협력해 Windows 앱에서 최적화된 로컬 모델 구축 과정을 간소화했으며, 최소 24GB VRAM을 갖춘 RTX GPU를 지원한다.
- **Perplexity Portable Computer:** 모델·오케스트레이션·도구를 하나로 묶은 앱 환경이다. 최소 24GB VRAM의 NVIDIA RTX GPU가 있는 Linux에서 먼저 제공되고, Windows 지원은 추후 제공된다. 클라우드의 15개 이상 프론티어 모델로 작업을 확장할 수 있으며, 콘텐츠를 클라우드로 보내기 전에 사용자 권한을 요청한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"가장 널리 사용되는 에이전트 앱 3종이 Windows에서 간소화된 로컬 모델 설정을 제공할 예정입니다. 각 앱은 llama.cpp를 기반으로 구축됐으며, NVIDIA의 최신 추론 최적화를 활용합니다."</blockquote></details>

## 2. NVIDIA PAIR: 로컬 네트워크의 유휴 PC를 추론 풀로

원문에 따르면 미국 가정의 절반 이상이 PC를 두 대 이상 갖고 있지만, 상당한 컴퓨팅 성능이 하루 대부분 놀고 있다. PAIR는 이 자원을 로컬 AI 추론에 쓰게 해 주는 무료 오픈소스 도구다.

- 로컬 네트워크에서 호환 PC를 자동으로 찾고, 여유 용량이 있는 시스템으로 개별 추론 요청을 넘긴다.
- Ollama와 LM Studio에서 쓸 수 있고, 기기가 네트워크에 붙거나 떨어지는 상황에도 대응한다.
- 에이전트 워크플로가 복잡한 작업을 병렬 작업으로 쪼개는 특성과 잘 맞는다. 예컨대 Hermes가 복잡한 메일함 정리를 여러 하위 에이전트로 나누면, PAIR가 모든 작업을 한 GPU에 몰지 않고 여러 PC에 분산한다.
- 메인 시스템이 게이밍이나 콘텐츠 제작에 쓰이는 동안 AI 워크로드를 다른 PC로 옮길 수 있다.
- 베타로 Windows·macOS·Linux를 지원한다. 지원 하드웨어는 GeForce RTX 20 시리즈 이상, Turing 이후 아키텍처의 RTX PRO Workstation GPU, DGX Spark, Apple M4 이상이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"PAIR는 로컬 네트워크에서 호환되는 PC를 자동으로 검색하고, 여유 용량이 있는 시스템으로 개별적인 추론 요청을 전달합니다. 이는 Ollama와 LM Studio에서 활용 가능하며, 기기가 네트워크에 연결되거나 연결 해제되는 상황에도 대응할 수 있습니다."</blockquote></details>

## 3. 추론 가속: llama.cpp 최대 1.9배, vLLM 최대 1.4배

로컬 에이전트가 굼뜨지 않으려면 추론 성능이 받쳐 줘야 한다. NVIDIA는 오픈소스 llama.cpp·vLLM 커뮤니티와 협력해 RTX 플랫폼 전반에서 에이전트 워크로드를 가속했다.

- **llama.cpp:** GeForce RTX 5090에서 커널 최적화, 향상된 추측 디코딩, 더 빠른 프리필로 최대 **1.9배** 처리량을 낸다.
- **vLLM:** RTX PRO 6000 Blackwell Workstation Edition에서 **1.2배**, 2개 DGX Spark 클러스터에서 최대 **1.4배** 성능을 낸다. FlashInfer의 새 XQA 어텐션 커널과 백엔드 최적화가 두 플랫폼의 추론을 가속한다.
- LM Studio와 Ollama 애플리케이션에서도 같은 최적화를 경험할 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"llama.cpp는 GeForce RTX 5090에서 커널 최적화와 향상된 추측 디코딩 기법, 더 빠른 프리필을 통해 최대 1.9배 높은 처리량을 제공합니다. vLLM은 RTX PRO 6000 Blackwell Workstation Edition에서 1.2배, 2개의 DGX Spark 클러스터에서 최대 1.4배의 성능을 제공합니다."</blockquote></details>

## 4. RTX Spark: 10월 출시

NVIDIA RTX Spark는 Lenovo·Acer의 Windows PC 신제품과 함께 10월에 출시된다. IFA에서 Lenovo는 Yoga Pro 9n과 Yoga 9n 2-in-1을, Acer는 소형 데스크톱 RTX Spark 콘셉트를 공개했다.

- **1페타플롭 RTX Blackwell GPU**
- **최대 128GB 통합 메모리**
- **고효율 20코어 Grace CPU**
- 새로운 Windows Agent 프레임워크와 결합하면 운영체제 수준의 제어 아래에서 에이전트를 안전하게 백그라운드로 실행할 수 있다.
- 종일 쓰는 배터리의 슬림 노트북부터 상시 실행 에이전트용 소형 데스크톱까지 여러 폼팩터를 지원한다.
- 지난주 Gamescom에서 Electronic Arts·Embark·Ubisoft가 RTX Spark 지원 게임사 대열에 합류했다. 지난 5월 COMPUTEX에서는 KRAFTON·NetEase·Riot Games·XBOX가 지원을 발표했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"강력한 1페타플롭 RTX Blackwell GPU와 최대 128GB의 통합 메모리, 고효율 20코어 Grace CPU를 탑재한 RTX Spark는 뛰어난 성능과 효율성을 제공하는데요."</blockquote></details>

## 5. 로컬 실행에 최적화된 오픈 모델

원문은 지난 8월 소개된 로컬 AI 소식으로 아래 모델들을 꼽는다.

| 모델 | 규모 | 특징 | 실행 환경 |
|------|------|------|-----------|
| Nemotron 3.5 Lightning | 30B | 현재 사용 가능 | RTX PC, RTX PRO Workstation, DGX Spark, Jetson |
| Qwen3.8-27B | 27B | 로컬 에이전트·코딩 워크로드에 최적화 | NVIDIA GPU |
| Qwen 8-Flash-Next | 멀티모달 MoE | 오픈 웨이트 | DGX Spark, DGX Station |
| LTX 2.5 | - | 오픈 월드 비디오 생성, NVFP4·FastVideo·ComfyUI 개선 | RTX GPU, DGX Spark, DGX Station |
| MiniMax-H3 | - | 동기화된 오디오를 지원하는 오픈 웨이트 비디오 생성. 4단계 증류 버전 FastH3는 최대 7배 성능 향상 | NVIDIA GPU (ComfyUI) |
| Muse Glimmer (Meta) | 30B | 코딩·에이전트 워크로드, DGX Spark용 NVFP4 양자화 공개 | GeForce RTX PC, DGX Spark, DGX Station, Jetson |
| DeepSeek V4 Flash | 총 2,840억, 활성 130억 (MoE) | 2개 DGX Spark 클러스터에서 실행 | DGX Spark, DGX Station |
| GLM-5.3-Flash | 멀티모달 MoE | 에이전트 AI를 DGX Station으로 확장 | DGX Station |

## 6. 창작 워크플로: PhotoDirector AI PC 모드

CyberLink PhotoDirector 365의 새 AI PC 모드는 RTX Spark에 최적화돼 10월에 출시된다. 디퓨전 모델을 창작 소프트웨어에 통합해 아티스트가 쉽게 쓰게 한 최초 사례 가운데 하나라고 한다. 생성형 편집, 이미지 향상, 객체 제거, 배경 제거·교체, 인물 보정을 제공하며, 작업에 따라 로컬이나 클라우드 처리를 고를 수 있다. 로컬 AI 가속에는 TensorRT-RTX와 FP8을 쓴다.

## 7. 시사점

이번 발표에서 눈에 띄는 점은 모델·런타임·하드웨어·네트워킹이 한꺼번에 움직였다는 것이다.

- **런타임:** llama.cpp·vLLM 최적화가 RTX·DGX 하드웨어로 들어왔다.
- **오케스트레이션:** Hermes·OpenClaw·Perplexity가 원클릭 설정으로 진입 장벽을 낮췄다.
- **네트워킹:** PAIR가 단일 머신의 한계를 로컬 네트워크 분산으로 넘으려 한다.
- **하드웨어:** RTX Spark가 1페타플롭 GPU와 128GB 통합 메모리를 일반 PC 폼팩터에 담는다.
- **모델:** 주요 오픈 모델이 NVFP4 양자화, FastVideo, ComfyUI 지원 같은 최적화와 함께 공개됐다.

성능 수치는 모두 NVIDIA가 밝힌 것이라 독립적인 벤치마크로 확인된 값이 아니다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA는 오픈소스 llama.cpp, vLLM 커뮤니티와 지속적으로 협력하며 로컬 NVIDIA 플랫폼 전반에서 에이전틱 워크로드를 가속하고 있습니다."</blockquote></details>

## 8. 참고 자료

- 원문: [NVIDIA, IFA 2026서 로컬 AI 가속하는 신기술·생태계 공개](https://blogs.nvidia.co.kr/blog/local-ai-ifa-next-gen-agents-nv-pair-rtx-spark/){:target="_blank"} (NVIDIA 블로그 코리아, 2026-09-08)
- NVIDIA PAIR 기술 블로그: [developer.nvidia.com/blog/nvidia-pair-virtual-inference-router-expands-available-compute-on-your-local-network](https://developer.nvidia.com/blog/nvidia-pair-virtual-inference-router-expands-available-compute-on-your-local-network){:target="_blank"}
- RTX Spark 제품 페이지: [nvidia.com/ko-kr/products/rtx-spark/](https://www.nvidia.com/ko-kr/products/rtx-spark/){:target="_blank"}
- Hermes Agent: [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/){:target="_blank"}
- OpenClaw: [openclaw.ai](https://openclaw.ai/){:target="_blank"}
- Perplexity Portable Computer: [perplexity.ai/hub/products/portable-computer](https://www.perplexity.ai/hub/products/portable-computer){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/O_UDOY0gM1Q){:target="_blank"}
