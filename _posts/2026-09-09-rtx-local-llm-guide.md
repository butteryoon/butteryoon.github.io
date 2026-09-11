---
layout: post
comments: true
title: "RTX PC에서 로컬 LLM 시작하기 — Ollama·LM Studio·AnythingLLM"
description: "NVIDIA RTX AI Garage 가이드 해설. Ollama, LM Studio, AnythingLLM으로 로컬 LLM 환경을 구축하는 방법과 Flash Attention·TensorRT 최적화 수치를 정리한다."
img: rtx_llm_title.webp
date: 2026-09-09 14:01:00 +0900
last_modified_at: 2026-09-11 21:10:00 +0900
tags: [nvidia, local-llm, ollama, lm-studio, anythingllm, rtx, gpu, windows-ml, llm] # add tag
related: llm
categories: dev
---
NVIDIA 블로그의 RTX AI Garage 시리즈 [「NVIDIA RTX PC에서 거대 언어 모델(LLM) 시작하기」](https://blogs.nvidia.co.kr/blog/rtx-ai-garage-how-to-get-started-with-llms/){:target="_blank"}(2025-11-06)를 정리했다. 발행된 지 좀 된 글이지만 로컬 LLM 입문 스택(Ollama·LM Studio·AnythingLLM)의 구도는 지금도 그대로라 정리해둘 가치가 있다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** RTX GPU에서 로컬 LLM을 시작하는 3층 스택 — 실행 프레임워크(**Ollama**, **LM Studio**), 응용 레이어(**AnythingLLM**), 시스템 통합(**Windows ML + TensorRT**, 최대 50% 빠른 추론). LM Studio는 Flash Attention 기본 활성화로 최대 20%, CUDA 커널 최적화로 최대 9% 성능 향상. 개인 정보가 밖으로 나가지 않는 것이 로컬 실행의 근본 장점이다.

## 1. 실행 프레임워크 — Ollama와 LM Studio

- **Ollama**: OpenAI **gpt-oss-20B**, Google **Gemma 3** 모델의 GeForce RTX 성능 개선이 반영됐고, **Gemma 3 270M**과 **EmbeddingGemma** 조합으로 초경량 RAG를 지원한다. 메모리 사용률을 극대화·정확히 보고하도록 **모델 스케줄링 시스템이 개선**됐고 다중 GPU 처리 성능도 좋아졌다.
- **LM Studio** (llama.cpp 기반): 새로운 **hybrid-mamba 아키텍처** 기반의 NVIDIA **Nemotron Nano v2 9B** 지원, **Flash Attention 기본 활성화로 최대 20%** 성능 향상, RMS Norm·빠른 나눗셈 기반 **CUDA 커널 최적화로 추가 최대 9%** 향상.

지원 모델 폭도 넓어졌다 — OpenAI gpt-oss, Alibaba **Qwen 3**, NVIDIA Nemotron Nano v2 등 최신 오픈 웨이트 모델이 RTX 가속으로 돌아간다.

## 2. 응용 레이어 — AnythingLLM

로컬 문서(PDF 등)로 지식 기반을 만들고 맞춤형 AI 챗봇/에이전트를 구성하는 RAG 도구다. 원문은 교육 유스케이스를 구체적으로 든다: 강의 슬라이드 기반 플래시카드 생성, 자료 기반 문맥 질문, 퀴즈 생성. 학습이나 사내 문서에 쓸 때는 문서가 클라우드로 나가지 않는다는 게 결정적이다.

## 3. 시스템 통합 — Windows ML과 G-Assist

- **Windows ML + TensorRT**: Windows 11 PC에서 추론이 **최대 50% 빨라진다**.
- **Project G-Assist**: 음성/텍스트 명령으로 게이밍 PC 설정을 조정·최적화하는 실험적 어시스턴트. 노트북 앱 프로필, BatteryBoost, WhisperMode 제어가 추가됐고 [플러그인 생태계](https://github.com/NVIDIA/g-assist){:target="_blank"}로 확장된다.

## 4. 읽으면서 든 생각

- vLLM·SGLang 같은 서버급 스택과 별개로, 엔드유저가 바로 쓰는 Ollama/LM Studio 계열도 Flash Attention·커널 최적화가 기본값이 되는 흐름이다. 개발 장비에서 추론 지연을 줄이는 가장 싼 방법은 이 기본값들을 최신으로 유지하는 것이다.
- 소형 고효율 모델(Nemotron Nano v2 9B, Gemma 3 270M)과 특화 임베딩 모델(EmbeddingGemma)을 조합하는 RAG 구성이 사실상의 권장 패턴으로 자리 잡았다.
- 유의점: Flash Attention·FP8 활용은 최신 Tensor Core 탑재 RTX가 전제고 Windows ML 경로는 Windows 11 + 최신 드라이버(NVIDIA App) 세팅이 필요하다.

## 참고

- [원문: RTX PC에서 LLM 시작하기](https://blogs.nvidia.co.kr/blog/rtx-ai-garage-how-to-get-started-with-llms/){:target="_blank"}
- [Ollama](https://ollama.com/){:target="_blank"} · [LM Studio](https://lmstudio.ai/){:target="_blank"}
- [AnythingLLM과 NIM](https://blogs.nvidia.co.kr/blog/rtx-ai-garage-anythingllm-nim/){:target="_blank"}
- [Project G-Assist GitHub](https://github.com/NVIDIA/g-assist){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/3GUW88tRmv8){:target="_blank"}
