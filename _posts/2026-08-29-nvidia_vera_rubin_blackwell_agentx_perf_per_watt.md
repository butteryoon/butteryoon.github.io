---
layout: post
comments: true
title: "NVIDIA Vera Rubin·Blackwell — AgentX 벤치마크로 본 와트당 에이전틱 AI 성능의 새 기준"
description: "NVIDIA 테크니컬 블로그 해설. SemiAnalysis AgentX 벤치마크에서 Vera Rubin NVL72가 GB300 NVL72 대비 메가와트당 30배, GB300 NVL72가 H200 NVL8 대비 15~80배 처리량 우위를 보인 결과와 Dynamo·NVLink·MoE 런타임 최적화가 만든 효율성 격차를 정리한다."
img: command-title.webp
date: 2026-08-29 18:01:11 +0900
last_modified_at: 2026-08-29 18:01:11 +0900
tags: [nvidia, vera-rubin, blackwell, agentic-ai, agentx, semi-analysis, inference, performance-per-watt, gb300, dynamo, nvlink, moe] # add tag
related: llm
categories: dev
---
NVIDIA 테크니컬 블로그에 8월 28일 올라온 [「NVIDIA Vera Rubin·Blackwell, 와트당 에이전틱 AI 성능의 새로운 기준을 제시하다」](https://developer.nvidia.com/ko-kr/blog/nvidia-vera-rubin-and-blackwell-set-a-new-standard-for-agentic-ai-performance-per-watt/){:target="_blank"}를 읽고 핵심만 추려 정리했다. 에이전틱 워크로드 특화 벤치마크인 **AgentX(SemiAnalysis)** 실측치로 Vera Rubin NVL72와 GB300 NVL72의 전력 효율 격차를 정량화한 드문 사례다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** SemiAnalysis AgentX(에이전틱 코딩 재현 벤치마크)상에서 **Vera Rubin NVL72는 GB300 NVL72 대비 메가와트당 AI 팩토리 처리량 최대 30배**, **GB300 NVL72는 H200 NVL8 대비 최대 15배(DeepSeek V4 Pro) ~ 80배(Kimi K3 2.8T)** 우위. 핵심은 ① MoE 서빙 런타임(SGLang·TensorRT-LLM·vLLM)의 와이드 엑스퍼트 패러렐리즘·DeepEP, ② DeepGEMM·MXFP4/8 커널과 통신-연산 오버랩, ③ **NVIDIA Dynamo**의 프리필/디코드 분리·세션 인식·KV 캐시 라우팅, ④ **NVLink 스케일업 패브릭** 72-GPU 단일 도메인 결합. 동일 전력 예산으로 더 많은 동시 에이전트 세션·더 긴 컨텍스트·더 낮은 토큰 비용을 달성.

## 1. 배경 — 에이전틱 워크로드가 바꾼 벤치마크 요구사항

기존 LLM 서빙 벤치마크(고정 ISL/OSL)는 **에이전트 세션의 가변성·긴 컨텍스트 누적·도구 호출 간격·KV 캐시 재사용**을 반영하지 못한다. OpenRouter 실사용 100조 토큰 분석에서도 요청당 평균 프롬프트 토큰이 4×로 늘었다. 에이전트 요청은 일반 채팅 대비 15× 토큰을 썼다.

이에 SemiAnalysis가 **InferenceX** 제품군에 **AgentX**를 추가했다. 특징:
- **사전 녹화된 Claude Code 세션**을 AIPerf 클라이언트로 턴 단위 재현
- 컨텍스트 길이·입출력 시퀀스·추론 시간·도구 호출 지연을 원본 타이밍 그대로 유지
- 동시 실행 수준을 조절해 **처리량 ↔ 인터랙티브 성능 트레이드오프** 곡선 측정
- 지표: **메가와트당 토큰 수**(핵심), E2E 정규화 인터랙티브 성능, 표준 인터랙티브 성능, E2E 지연 시간, TTFT

## 2. Vera Rubin NVL72 결과 — 30× 효율 도약

| 워크로드 | 비교 대상 | 메가와트당 처리량 비율 | 인터랙티브 성능 기준 |
|----------|-----------|------------------------|---------------------|
| DeepSeek V4 Pro | GB300 NVL72 | **최대 30×** | 사용자당 160 tok/s |

- 동일 응답성(160 tok/s/user) 유지하면서 **에이전틱 추론 용량 30배 확대**
- NVIDIA Rubin GPU 아키텍처(에이전틱 AI 시대 설계)가 기여: [Inside NVIDIA Rubin GPU Architecture](https://developer.nvidia.com/blog/inside-nvidia-rubin-gpu-architecture-powering-the-era-of-agentic-ai/){:target="_blank"} 참조

## 3. GB300 NVL72 결과 — H200 대비 15~80×

| 워크로드(모델) | 비교 대상 | 메가와트당 처리량 비율 | 비용 절감(1M 토큰당) |
|----------------|-----------|------------------------|---------------------|
| DeepSeek V4 Pro 1.6T | H200 NVL8 | **최대 15×** | **최대 1/10** |
| Kimi K3 2.8T | H200 NVL8 | **약 80×** | 인터랙티브 성능 215 tok/s까지 확장(H200 도달 불가 영역) |

- **단위 경제성 직결**: 동일 전력·인프라 예산으로 더 많은 에이전트 용량, 또는 동일 용량을 1/10 비용으로 운영
- 모델 규모가 커질수록 격차 확대 → MoE 전문가 병렬·KV 캐시·스케일업 패브릭 효율이 핵심이라는 신호

## 4. 효율성 격차를 만든 4대 시스템 최적화

### 4.1 MoE 서빙 런타임 — 와이드 엑스퍼트 패러렐리즘 & DeepEP
- **SGLang, TensorRT-LLM, vLLM**이 NVL72 72-GPU 도메인 전체에 전문가 분산
- **Wide Expert Parallelism**: 더 많은 GPU에 전문가 워크로드 균등 분산 → 유효 배치 크기 확대
- **DeepEP**: 전문가 간 올투올 통신 최적화로 동시 에이전트 요청 처리량 증대

### 4.2 MoE 커널 & 통신-연산 오버랩
- **DeepGEMM 기반 커널** + **MXFP4/MXFP8** 혼합 정밀도
- **융합 MoE 실행 경로**: 전문가 단계 간 데이터 이동 지연 최소화
- **전문가 병렬 통신 ⊗ 텐서 코어 연산 중첩** → 토큰 처리량 향상

### 4.3 NVIDIA Dynamo — 세션 인식 서빙 & KV 캐시 라우팅
- **프리필/디코드 독립적 워커 풀**로 분리 확장
- **세션 ID**로 관련 모델 호출·도구 실행·트레이스 연계 → 세션 인식 서빙
- **KV 캐시 인식 라우터**: 캐시 중복도·워커 부하 기반 라우팅 → 불필요 프리필 감소·멀티턴 응답성 유지

### 4.4 NVLink 스케일업 패브릭 — 72-GPU 단일 도메인
- GB300/Vera Rubin NVL72: **72개 GPU를 고대역폭 NVLink로 단일 스케일업 도메인 구성**
- 대규모 MoE 모델을 **하나의 통합 랙-스케일 시스템**으로 서빙 가능
- 컴퓨팅·메모리·전문가 병렬 통신·KV 캐시 이동을 랙 내부에서 완결

## 5. 도입 관점 체크포인트

| 항목 | 고려 사항 |
|------|-----------|
| **하드웨어 전제** | NVL72 랙 단위 공동 설계 필요(기존 x86+GPU 클러스터 드롭인 불가) |
| **모델 검증 범위** | 공개 수치는 DeepSeek V4 Pro / Kimi K3 기준. 타 모델 계열 별도 검증 필요 |
| **소프트웨어 스택** | SGLang·TensorRT-LLM·vLLM·Dynamo 최신 버전 동기화 필수 |
| **비용·전력·냉각** | 랙 구성 TCO 미공개 → 전수명주기 기준 비교 필요 |
| **일반 GPU 환경 적용성** | 프리필-디코드 분리(vLLM disaggregated prefill), KV 캐시 라우팅, 세션 인식 서빙은 하드웨어 무관하게 도입 가능 |

## 6. 관련 글

- [NVIDIA Groq 3 LPX — Vera Rubin에서 긴 컨텍스트 초고속 인터랙티비티 구현]({{site.baseurl}}/dev/2026/08/26/nvidia_groq3_lpx_vera_rubin.html) — 추론 가속기 계층
- [NVIDIA AVO, ARC-AGI-3 100% 달성]({{site.baseurl}}/dev/2026/08/25/nvidia_avo_arc_agi3.html) — 에이전트 아키텍처 계층
- [NVIDIA Vera 스토리지 벤치마크]({{site.baseurl}}/dev/2026/08/25/nvidia_vera_storage_benchmark.html) — 스토리지 계층

