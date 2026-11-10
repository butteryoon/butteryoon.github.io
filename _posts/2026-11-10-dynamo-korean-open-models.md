---
layout: post
comments: true
title: "NVIDIA Dynamo 레시피로 한국 오픈 모델 4종 B200 최적화 — A.X K2, K-EXAONE 2.0, Solar Open 2, Motif 3 배포 가이드"
description: "NVIDIA가 공개한 Dynamo 레시피 4종을 통해 한국 주요 오픈 모델(SK텔레콤 A.X K2, LG K-EXAONE 2.0, 업스테이지 Solar Open 2, Motif 3)을 B200 GPU에서 최적화 배포하는 구체적인 구성과 벤치마크 결과를 정리한다."
img: dynamo_korean_models_title.webp
date: 2026-11-10 20:00:00 +0900
last_modified_at: 2026-11-10 20:00:00 +0900
tags: [nvidia, dynamo, llm-serving, korean-llm, moe, b200]
related: llm-serving
categories: [nvidia, llm-serving, ai-infra]
source_url: https://developer.nvidia.com/ko-kr/blog/optimizing-koreas-leading-open-models-with-new-nvidia-dynamo-recipes/
source_date: 2026-11-09
---

국내 오픈 모델을 받아다 직접 서빙해 보면, 모델을 고르는 것보다 "이 모델을 몇 장에 어떤 병렬화로 올려야 하는가"에서 시간이 더 많이 녹는다. NVIDIA 기술 블로그가 11월 9일 올린 [「새로운 NVIDIA Dynamo 레시피로 한국의 주요 오픈 모델 최적화하기」](https://developer.nvidia.com/ko-kr/blog/optimizing-koreas-leading-open-models-with-new-nvidia-dynamo-recipes/){:target="_blank"}는 그 탐색을 네 모델에 대해 대신 해 놓은 글이다. 체크포인트·런타임 이미지·TP 크기·KV 캐시 정밀도·라우팅·speculative decoding 설정을 한 덩어리로 고정한 배포 레시피를 Kubernetes 매니페스트까지 붙여 공개했다. B200 8대가 없더라도 읽을 값은 남는다 — 어떤 선택이 어떤 제약에서 나왔는지가 모델별로 적혀 있다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** NVIDIA가 한국 오픈 모델 4종(A.X K2 688B, K-EXAONE 2.0 750B, Solar Open 2 250B, Motif 3 314B)의 B200 배포 레시피를 공개했다. NVFP4 가중치 + FP8 KV 캐시 + speculative decoding(EAGLE3/MTP) + 모델별 MoE 커널을 묶어, **사용자별 생성 속도 50 tok/s 이상·p50 TTFT 5초 미만이라는 SLA 아래에서** GPU당 출력 처리량이 baseline 대비 약 1.43~3.67배가 됐다. Kubernetes 매니페스트·체크포인트 다운로드 작업·스모크 테스트·벤치마크 절차가 함께 있어 그대로 재현한 뒤 한 항목씩 바꿔 가며 튜닝할 수 있다.

## 무엇을 공개했나

NVIDIA는 한국의 대표적인 오픈 웨이트 모델 4종(SK텔레콤 A.X K2, LG AI연구원 K-EXAONE 2.0, 업스테이지 Solar Open 2, Motif Technologies Motif 3)을 B200 GPU에서 효율적으로 서빙하기 위한 **NVIDIA Dynamo 최적화 레시피**를 공개했다. 모든 모델이 Mixture-of-Experts(MoE) 아키텍처이며, Dynamo의 분산 추론 서빙 프레임워크 위에서 vLLM과 결합해 모델 실행·요청 라우팅·KV 캐시 전송을 통합 최적화한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA는 한국의 주요 오픈 모델인 SK텔레콤의 A.X K2, LG AI연구원의 K-EXAONE 2.0, 업스테이지의 Solar Open 2, Motif Technologies의 Motif-3를 위한 Dynamo 최적화 배포 레시피를 공개합니다. NVIDIA B200 GPU에 최적화된 이 레시피는 개발자가 모델을 더 빠르게 실제 서비스에 적용할 수 있도록 지원합니다."</blockquote></details>

## 모델 4종과 아키텍처

| 모델 | 개발사 | 전체 / 활성 파라미터 | 문맥 길이 | 어텐션 아키텍처 |
|------|--------|---------------------|-----------|----------------|
| A.X K2 | SK Telecom | 688B / 33B | 256K 토큰 | Sparse MLA (Gated Attention) |
| K-EXAONE 2.0 | LG AI Research | 750B / 37B | 256K 토큰 | 전역 어텐션 + 슬라이딩 윈도 |
| Solar Open 2 | Upstage | 250B / 15B | 1M 토큰 | GQA softmax(12L) + KDA linear attention(36L) |
| Motif 3 | Motif Technologies | 314B / 13.2B | 256K 토큰 | Grouped Differential Latent Attention (GDLA) |

<details class="evidence"><summary>원문 근거</summary><blockquote>"레시피는 연산량과 메모리, 문맥 처리 요구사항이 서로 다른 네 가지 전문가 혼합(Mixture-of-Experts, MoE) 모델을 지원합니다." / "A.X K2는 전체 688B 파라미터 중 토큰당 33B를 활성화하는 MoE 모델입니다. 256개의 routed expert 중 8개를 선택하고 shared expert도 사용합니다. Sparse Gated Attention은 Multi-head Latent Attention(MLA)을 기반으로..." / "K-EXAONE 2.0은 전체 750B급, 토큰당 활성 37B 파라미터를 갖춘 MoE 모델입니다. 전역 어텐션과 슬라이딩 윈도 어텐션을 결합하며..." / "Solar Open 2는 전체 250B 파라미터 중 토큰당 15B를 활성화하는 하이브리드 MoE 모델입니다. 48개 레이어 중 12개는 GQA 기반 softmax attention, 36개는 KDA linear attention을 사용하며..." / "Motif 3는 전체 314B 파라미터 중 토큰당 13.2B를 활성화하는 MoE 모델입니다. 384개의 routed expert 중 8개를 선택하고 shared expert도 사용합니다. Grouped Differential Latent Attention(GDLA)은 grouped differential attention과 Multi-head Latent Attention(MLA)의 압축 KV latent를 결합하며..."</blockquote></details>

## 네 레시피가 공유하는 최적화 스택

모든 레시피에 공통으로 적용되는 최적화 스택:

| 기법 | 설명 | 효과 |
|------|------|------|
| **NVFP4 체크포인트** | 가중치 4비트 양자화(W4A4, 어텐션·라우터·shared expert·dense layer는 FP8, 임베딩·LM head는 BF16) | 모델 메모리 사용량 대폭 감소 |
| **FP8 KV 캐시** | 생성 중 문맥 유지용 KV 캐시를 FP8로 저장 | 캐시 메모리 절감, 긴 문맥 지원 |
| **NVIDIA Dynamo** | 멀티 GPU/노드 추론 조율: 요청 배치, prefill·decode 분리, KV-aware 라우팅, NIXL/UCX 전송 | GPU 자원 효율 활용, 지연 시간 목표 충족 |
| **vLLM 런타임** | 모델 실행을 담당, Dynamo와 연동 | 검증된 추론 엔진, 지속적 업데이트 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"새로운 레시피는 Dynamo에 NVFP4 체크포인트와 vLLM, 모델별 최적화를 결합해 배포 과정의 시행착오를 줄입니다." / "NVFP4 가중치는 모델의 메모리 요구량을 줄이고, FP8 KV 캐시는 생성 중 문맥을 유지하는 데 필요한 메모리를 줄입니다." / "NVIDIA Dynamo는 여러 GPU와 노드에 걸친 추론을 조율합니다. 이번 레시피에서 모델 실행은 vLLM이 담당하고, Dynamo는 요청 배치와 분리된 prefill·decode 워커를 조율합니다."</blockquote></details>

## 모델별 서빙 구성과 측정값

### A.X K2 — TP4 + Sparse MLA + EAGLE3

| 구성 | aggregated | disaggregated |
|------|------------|---------------|
| 워커 구성 | TP4 워커 2개 (GPU 8대) | TP4 prefill 2개 + TP4 decode 1개 (GPU 12대) |
| 주요 최적화 | Sparse MLA (FLASHINFER_MLA_SPARSE), KV-aware 라우팅, EAGLE3 speculative decoding | NIXL/UCX KV 전송, 동일한 TP4 구성 |

**벤치마크 (aggregated, B200 8대, 동시성 16):**
- Baseline: 44.3 tok/s/GPU
- 최적화: 76.5 tok/s/GPU (**1.72배 향상**)
- 사용자 출력 속도 평균: 55.83 tok/s, p50 TTFT: 525ms
- Synthetic EAGLE3 acceptance length: 2.12

<details class="evidence"><summary>원문 근거</summary><blockquote>"A.X K2는 전체 688B 파라미터 중 토큰당 33B를 활성화하는 MoE 모델입니다. 256개의 routed expert 중 8개를 선택하고 shared expert도 사용합니다. Sparse Gated Attention은 Multi-head Latent Attention(MLA)을 기반으로 긴 문맥 처리에 사용할 KV 후보를 선택하며, 최대 256K 토큰의 문맥을 지원합니다." / "레시피는 skt/A.X-K2-NVFP4와 전용 draft 모델인 skt/A.X-K2-EAGLE3를 함께 사용합니다. Routed expert는 NVFP4 W4A4, 어텐션·라우터·shared expert·dense layer는 FP8, 임베딩과 LM head는 BF16을 사용합니다. KV 캐시는 FP8입니다." / "A.X K2: B200 GPU 8대, 동시성 16. Synthetic EAGLE3 acceptance length 2.12에서 76.49 tok/s/GPU를 측정했습니다. 평균 사용자 출력 속도는 55.83 tok/s, p50 TTFT는 525ms였습니다."</blockquote></details>

### K-EXAONE 2.0 — TP4 + MTP + 검증된 MoE 커널

| 구성 | aggregated | disaggregated |
|------|------------|---------------|
| 워커 구성 | TP4 워커 1개 (GPU 4대) | TP4 prefill 1개 + TP4 decode 1개 (GPU 8대) |
| 주요 최적화 | Multi-Token Prediction(MTP) draft length 2, FLASHINFER_CUTLASS 백엔드 고정 | NIXL/UCX KV 전송, speculative 설정 일치 필수 |

**벤치마크 (aggregated, B200 4대, 동시성 7):**
- Baseline: 55.0 tok/s/GPU
- 최적화: 87.0 tok/s/GPU (**1.58배 향상**)
- 사용자별 E2E 처리량: 51.4 tok/s, p50 TTFT: 291ms
- 실측 KV 재사용률: 8.8% (목표 70% 대비 낮음 — working set이 캐시 용량 초과)

<details class="evidence"><summary>원문 근거</summary><blockquote>"K-EXAONE 2.0은 전체 750B급, 토큰당 활성 37B 파라미터를 갖춘 MoE 모델입니다. 전역 어텐션과 슬라이딩 윈도 어텐션을 결합하며 최대 256K 토큰의 문맥을 지원합니다. 레시피는 LGAI-EXAONE/K-EXAONE-2.0-750B-A37B-NVFP4와 FP8 KV 캐시를 사용합니다. 내장 Multi-Token Prediction(MTP)을 draft length 2로 실행해 여러 후보 토큰을 함께 검증합니다." / "권장 aggregated 프로파일은 B200 GPU 4대에서 TP4를 사용합니다. 약 530GB의 가중치는 TP2에 적재되지 않으며, 레시피 테스트에서는 TP8의 추가 통신으로 GPU당 효율이 낮아졌습니다." / "MoE 백엔드는 FLASHINFER_CUTLASS로 고정합니다. 레시피에는 이 체크포인트에서 자동 선택 기능이 FLASHINFER_TRTLLM을 선택할 경우 장문 생성의 정확성 문제가 발생한다고 명시되어 있습니다." / "K-EXAONE 2.0: B200 GPU 4대, 동시성 7. 사용자별 E2E 처리량은 51.4 tok/s, p50 TTFT는 291ms였습니다. 문맥 한도를 초과한 요청 38개는 지표에서 제외했으며, 실측 KV 재사용률은 8.8%였습니다."</blockquote></details>

### Solar Open 2 — TP2/TP4 + KV-aware 라우팅 + 하이브리드 상태 전송

| 구성 | aggregated | disaggregated |
|------|------------|---------------|
| 워커 구성 | TP2 워커 2개 (GPU 4대) | TP4 prefill 1개 + TP2 decode 2개 (GPU 8대) |
| 주요 최적화 | Prefix 캐싱, FP8 KV 캐시, flashinfer_trtllm + autotuning | KDA 상태 전송 (VLLM_SSM_CONV_STATE_LAYOUT=DS), 하이브리드 KV 페이지 |

**벤치마크 (aggregated, B200 4대, 동시성 20):**
- Baseline: 151.6 tok/s/GPU
- 최적화: 216.9 tok/s/GPU (**1.43배 향상**)
- 사용자 출력 속도 p50: 52.02 tok/s, p50 TTFT: 308ms
- Cold-cache 비교 시 워커 재시작 필요 (공개 벤치마크는 서버 측 KV 캐시 초기화 안 함)

<details class="evidence"><summary>원문 근거</summary><blockquote>"Solar Open 2는 전체 250B 파라미터 중 토큰당 15B를 활성화하는 하이브리드 MoE 모델입니다. 48개 레이어 중 12개는 GQA 기반 softmax attention, 36개는 KDA linear attention을 사용하며 최대 1M 토큰의 문맥을 지원합니다. 레시피는 nota-ai/Solar-Open2-250B-Nota-NVFP4를 사용합니다. aggregated 프로파일은 B200 GPU 4대에 TP2 워커 두 개를 배치하고, FP8 KV 캐시와 KV-aware 라우팅을 적용해 프롬프트와 대화 이력을 재사용합니다." / "disaggregated 프로파일은 TP4 prefill 워커 한 개와 TP2 decode 워커 두 개를 사용합니다. Prefill 워커가 하나이므로 프런트엔드는 round-robin 라우팅을 사용합니다. Prefill과 decode 사이에서 KDA 상태를 전달하려면 VLLM_SSM_CONV_STATE_LAYOUT=DS가 필요합니다." / "Solar Open 2: B200 GPU 4대, 동시성 20. 사용자 출력 속도 p50은 52.02 tok/s, p50 TTFT는 308ms였습니다. 실행 전에 서버 측 KV 캐시를 초기화하지 않았으므로 이전 캐시 상태가 결과에 영향을 줄 수 있습니다."</blockquote></details>

### Motif 3 — 전용 vLLM 런타임 + Expert Parallelism + 자체 MTP

| 구성 | aggregated | disaggregated |
|------|------------|---------------|
| 워커 구성 | Expert parallelism TP2 워커 1개 (GPU 2대) | TP2 prefill 3개 + TP2 decode 1개 = 3P1D (GPU 8대) |
| 주요 최적화 | Motif 전용 vLLM 포크 런타임 이미지, FP8 KV 캐시, 내장 MTP head (draft length 2) | KV-aware 라우팅, NIXL/UCX 전송, 3P1D 토폴로지 |

**벤치마크:**
- Aggregated (B200 2대, 동시성 11): 293.96 tok/s/GPU (synthetic MTP acceptance 2.13)
- Disaggregated 3P1D (B200 8대): 319.26 tok/s/GPU (**3.67배 향상** vs 별도 baseline)
- 사용자 출력 속도 p50: 66.27 tok/s, p50 TTFT: 388ms

<details class="evidence"><summary>원문 근거</summary><blockquote>"Motif 3는 전체 314B 파라미터 중 토큰당 13.2B를 활성화하는 MoE 모델입니다. 384개의 routed expert 중 8개를 선택하고 shared expert도 사용합니다. Grouped Differential Latent Attention(GDLA)은 grouped differential attention과 Multi-head Latent Attention(MLA)의 압축 KV latent를 결합하며, 최대 256K(262,144) 토큰의 문맥을 지원합니다." / "이러한 전용 구성요소 때문에 런타임 호환성을 먼저 고려해야 합니다. Motif의 공개 vLLM 포크는 MoE 레이어용 grouped PolyNorm 활성화 커널 등 Motif 전용 CUDA 커널을 컴파일하므로, 모델 코드를 기본 vLLM 설치에 단순 패치하는 방식으로는 적용할 수 없습니다. 레시피는 Motif가 공개한 vLLM 이미지를 기반으로 구축한 런타임 이미지(nvcr.io/nvidia/ai-dynamo/vllm-runtime:1.5.0-motif-3-dev.2)에서 Dynamo 1.5.0을 실행합니다." / "Motif 3: B200 GPU 2대, 동시성 11. Synthetic MTP acceptance length 2.13에서 293.96 tok/s/GPU를 측정했습니다. 사용자 출력 속도 p50은 66.27 tok/s, p50 TTFT는 388ms였습니다." / "Motif 3의 표에 표시한 319.26 output tok/s/GPU는 B200 GPU 8대를 사용하는 3P1D disaggregated 구성의 결과입니다. 최적화된 aggregated 구성의 결과는 293.96 output tok/s/GPU입니다. 각각의 벤치마크 운영점에서 disaggregated의 GPU당 처리량이 약 8.6% 높았으며, 3.67배는 별도 baseline 대비 disaggregated 결과의 개선 배율입니다."</blockquote></details>

## baseline 대비 개선 폭 — 비교 기준부터 본다

| 모델 | 측정 구성 | Baseline | 최적화 | 개선 |
|------|----------|----------|--------|------|
| A.X K2 | aggregated, B200 8대 | 44.3 | 76.5 | 약 1.72배 |
| K-EXAONE 2.0 | aggregated, B200 4대 | 55.0 | 87.0 | 약 1.58배 |
| Solar Open 2 | aggregated, B200 4대 | 151.6 | 216.9 | 약 1.43배 |
| Motif 3 | disaggregated 3P1D, B200 8대 | 86.95 | 319.26 | 약 3.67배 |

단위는 GPU당 출력 tok/s다. 배수만 떼어 읽으면 두 가지를 놓친다.

첫째, 여기서 baseline은 "최적화를 안 한 상태"가 아니다. **같은 NVFP4 체크포인트를 Dynamo와 vLLM에서, 모델 적재에 필요한 최소 TP와 round-robin 라우팅으로** 돌린 구성이다. 실행에 필요한 호환성 패치는 baseline에도 들어가 있고 워커 replica 수도 레시피와 맞춘다. 그러니 이 배수는 양자화로 번 몫이 아니라 병렬화·라우팅·speculative decoding을 조율해서 번 몫이다.

둘째, 개선율은 **사용자별 생성 속도 50 tok/s 이상, p50 TTFT 5초 미만**이라는 SLA를 걸고 잰 값이다. 지연 목표를 떼고 처리량만 비교하면 다른 숫자가 나온다.

Motif 3의 3.67배는 조건이 하나 더 다르다. 나머지 셋은 aggregated인데 Motif 3만 disaggregated 3P1D 결과이고, 기준이 되는 baseline도 별도다. 같은 운영점에서 disaggregated가 aggregated보다 GPU당 8.6% 높았다는 것이 모델 안에서의 비교값이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"B200에서 모델별 baseline 대비 GPU당 출력 처리량은 aggregated 구성의 A.X K2에서 약 1.72배, K-EXAONE 2.0에서 약 1.58배, Solar Open 2에서 약 1.43배, disaggregated 구성의 Motif 3에서 약 3.67배로 증가했습니다." / "Baseline은 같은 모델의 NVFP4 체크포인트를 Dynamo와 vLLM에서 실행하며, 모델 적재에 필요한 최소 TP와 round-robin 라우팅을 사용합니다. 실행에 필요한 호환성 패치는 baseline에도 포함하고, 워커 replica 수는 최적화 레시피에 맞춥니다. 비교 지표는 GPU당 출력 처리량입니다." / "개선율은 사용자별 생성 속도 50 tok/s 이상과 p50 TTFT 5초 미만이라는 SLA를 기준으로 측정한 GPU당 출력 처리량을 나타냅니다. 각 모델의 처리량은 해당 지연 시간 목표와 함께 평가하세요."</blockquote></details>

## 레시피에 들어 있는 것

레시피 하나에는 다음이 함께 들어 있다. 설정 조합만 알려 주고 끝내는 문서가 아니라, 그대로 적용해 돌려 볼 수 있는 묶음이다.

1. **Kubernetes 배포 매니페스트** — NVIDIA Dynamo Operator용 (aggregated/disaggregated 각각)
2. **모델 다운로드 작업** — Hugging Face 체크포인트를 공유 스토리지(ReadWriteMany)에 준비
3. **토폴로지 구성** — TP 크기, replica 수, 캐시/라우팅/speculative decoding 설정
4. **스모크 테스트** — `/v1/chat/completions` 엔드포인트 기능 검증
5. **벤치마크 재현 절차** — AIPerf 또는 지정 trace 사용, 동시성 sweep 및 goodput 분석
6. **알려진 제약·운영 확인 설정** — 모델별 문서에 명시

<details class="evidence"><summary>원문 근거</summary><blockquote>"각 레시피는 모델별 체크포인트와 런타임 이미지, 병렬화 전략, KV 캐시 정밀도, 라우팅, speculative decoding, 파서, 커널을 하나의 조합으로 고정합니다. 배포와 실행 동작을 재현할 수 있도록 다음 항목도 제공합니다."</blockquote></details>

## 클러스터에 올리는 5단계

원문이 제시하는 배포·검증 순서는 이렇다.

**1. 인프라와 접근 권한 준비** — B200 GPU와 Kubernetes 클러스터, Dynamo Operator, 체크포인트 접근용 Hugging Face 토큰. disaggregated로 갈 거면 InfiniBand/RDMA device plugin의 리소스 이름을 먼저 확인한다. A.X K2는 워커당 호스트 메모리 400GiB가 필요하다.

**2. 공유 스토리지와 모델 캐시 구성** — K-EXAONE 2.0 체크포인트가 약 530GB라 최소 700GB를 잡으라고 한다. Solar Open 2는 약 153GB이고 캐시 매니페스트는 1,200Gi를 요청한다. 모델 다운로드 작업을 먼저 돌려 둔다.

**3. 서빙 토폴로지 배포** — aggregated나 disaggregated 매니페스트를 고정 런타임 이미지와 함께 적용한다. K-EXAONE 2.0은 최초 가중치 적재와 autotuning, CUDA Graph capture에 40~120분이 걸리니 타임아웃을 여기 맞춰 둔다.

**4. 기능과 전송 경로 확인** — `/v1/chat/completions` 스모크 테스트를 돌리고, reasoning·도구 호출을 쓴다면 파서 출력까지 본다. disaggregated는 NIXL/UCX가 실제로 InfiniBand를 타는지 확인한다.

**5. 애플리케이션 트래픽으로 성능 검증** — AIPerf에 자기 trace를 물려 동시성을 바꿔 가며 GPU당 처리량·TTFT·ITL·요청 성공률을 기록한다. 입력·출력 길이 분포와 캐시 상태, acceptance 설정을 함께 남겨 둬야 나중에 같은 조건으로 다시 잴 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"다음 다섯 단계로 레시피를 배포하고 검증할 수 있습니다. 1. 인프라와 접근 권한 준비... 2. 공유 스토리지와 모델 캐시 구성... 3. 서빙 토폴로지 배포... 4. 기능과 전송 경로 확인... 5. 애플리케이션 트래픽으로 성능 검증..."</blockquote></details>

## 튜닝은 레시피 재현 다음이다

원문이 강조하는 순서는 레시피 재현 → 한 설정 그룹씩 변경 → 기능·성능 동시 검증이다. 여러 항목을 한꺼번에 바꾸면 무엇이 기여했는지 알 수 없다.

| 모델 | 튜닝 전 필수 고정 사항 | 튜닝 대상 |
|------|----------------------|-----------|
| A.X K2 | 실제 EAGLE3 수락 동작 측정 | TP, replica, 라우팅, 캐시 |
| K-EXAONE 2.0 | 검증된 MoE 백엔드(FLASHINFER_CUTLASS) + MTP 설정 유지 | TP, replica, 라우팅 |
| Solar Open 2 | FP8 KV 캐시 + KDA 상태 레이아웃(DS) 호환성 유지 | TP, replica, 라우팅, 캐시 |
| Motif 3 | Motif vLLM 런타임 이미지에서 실행 필수 | TP, expert parallelism, replica |

<details class="evidence"><summary>원문 근거</summary><blockquote>"튜닝 전에 공개 레시피를 먼저 재현하세요. 모델과 런타임, 평가 요청을 고정한 뒤 TP와 replica, 캐시와 라우팅, speculative decoding 등 설정 그룹을 한 번에 하나씩 바꿉니다. 기능 검증을 통과한 구성만 비교하고, 각 측정은 동일 조건에서 반복해야 합니다." / "모델의 제약은 튜닝 범위를 결정합니다. A.X K2에서는 실제 EAGLE3 수락 동작을 측정하고, K-EXAONE 2.0에서는 검증된 MoE 백엔드와 MTP 설정을 유지해야 합니다. Solar Open 2에서는 FP8 KV 캐시와 KDA 상태 레이아웃의 호환성을 유지하고, Motif 3는 Motif vLLM 런타임 이미지에서 실행해야 합니다."</blockquote></details>

## aggregated로 시작할지, disaggregated로 갈지

| 워크로드 특성 | 권장 시작 구성 |
|--------------|---------------|
| 공개 워크로드와 유사 (채팅·일반 추론) | **Aggregated** — 배포 단순, K-EXAONE 2.0·Solar Open 2는 disaggregated보다 낮은 TTFT 기록 |
| 긴 입력·긴 출력 혼재, 단계별 부하 변화 다름 | **Disaggregated** 평가 — prefill/decode 독립 확장, 추가 GPU·KV 전송 오버헤드 포함 비교 |
| 시스템 프롬프트·도구 정의·대화 이력 반복 | **KV-aware 라우팅 + prefix 캐시** 적중률 측정, 재사용·워커 부하 균형 조정 |
| Disaggregated 배포 시 | RDMA 리소스 이름(rdma/ib 또는 rdma/shared_ib) 클러스터 device plugin과 일치 확인, NIXL/UCX가 실제 InfiniBand 사용 검증 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"워크로드의 병목과 서비스 목표를 바탕으로 초기 구성을 선택하세요. 공개된 워크로드와 유사한 조건에서는 K-EXAONE 2.0과 Solar Open 2가 우선 구성으로 제시하는 aggregated 프로파일로 시작하세요. 배포가 더 단순하며, 두 모델의 공개 측정에서는 disaggregated보다 낮은 TTFT를 보였습니다." / "Prefill과 decode를 독립적으로 확장해야 한다면 disaggregated 프로파일을 평가하세요... disaggregated 배포에서는 레시피의 RDMA 리소스 이름(rdma/ib 또는 rdma/shared_ib)을 클러스터의 device plugin과 일치시키세요. 장치 할당뿐 아니라 NIXL/UCX 전송이 실제로 InfiniBand를 사용하는지도 확인해야 합니다."</blockquote></details>

## 운영 신호로 병목 찾기

Dynamo 지표·구조화 로그·분산 trace 연계 시 다음 순서로 확인:

| 증상 | 1차 확인 지점 | 2차 확인 지점 |
|------|--------------|--------------|
| TTFT 증가 | 대기열 길이, prefill 워커 부하 | 라우터 prefix 중복 추정 vs 엔진 실제 KV 재사용률 격차 |
| ITL 증가 | decode 워커 부하, KV 캐시 사용률 | disaggregated: NIXL 텔레메트리로 KV 전송 지연 확인, UCX 로그로 IB 경로 검증 |
| 요청 성공률 저하 | 워커 준비 상태, OOM 여부 | 모델별 문맥 한도 초과 요청 비율 (K-EXAONE 38개, Motif 3 35개 제외됨) |

<details class="evidence"><summary>원문 근거</summary><blockquote>"Dynamo 지표와 구조화 로그, 분산 trace를 연계하면 요청 처리 경로에서 변화가 발생한 지점을 찾을 수 있습니다. TTFT가 증가하면 대기열과 prefill을, ITL이 증가하면 decode 부하와 KV 캐시 사용률을 확인하세요. 하나의 신호로 원인을 판단하기 전에 같은 부하와 시간 구간을 비교해야 합니다." / "라우터가 추정한 prefix 중복률과 엔진의 실제 KV 재사용률을 구분하세요. disaggregated에서는 선택적 NIXL 텔레메트리로 전송을 살펴볼 수 있으며, UCX 로그로 InfiniBand 경로를 확인할 수 있습니다."</blockquote></details>

## 참고

- [새로운 NVIDIA Dynamo 레시피로 한국의 주요 오픈 모델 최적화하기](https://developer.nvidia.com/ko-kr/blog/optimizing-koreas-leading-open-models-with-new-nvidia-dynamo-recipes/){:target="_blank"} — 원문, NVIDIA 기술 블로그 (2026-11-09, Chanran Kim·Joosung Yoon·Shwetha Krishnamurthy·Anahita Bhiwandiwalla)
- [A.X K2 레시피](https://docs.nvidia.com/dynamo/dev/recipes/ax-k2){:target="_blank"}
- [K-EXAONE 2.0 레시피](https://docs.nvidia.com/dynamo/dev/recipes/k-exaone-2){:target="_blank"}
- [Solar Open 2 레시피](https://docs.nvidia.com/dynamo/dev/recipes/solar-open2-250b){:target="_blank"}
- [Motif 3 레시피](https://docs.nvidia.com/dynamo/dev/recipes/motif-3){:target="_blank"}
- [Dynamo 레시피 카탈로그](https://docs.nvidia.com/dynamo/dev/recipes/browse){:target="_blank"}
- [Dynamo Kubernetes 빠른 시작](https://docs.nvidia.com/dynamo/dev/kubernetes/getting-started/quickstart){:target="_blank"}
- [Dynamo 성능 튜닝 가이드](https://docs.nvidia.com/dynamo/dev/kubernetes/operations/performance-tuning){:target="_blank"}
- [Dynamo 관측성 가이드](https://docs.nvidia.com/dynamo/dev/kubernetes/operations/observability){:target="_blank"}
- [AIPerf로 Kubernetes 배포 벤치마크하기](https://docs.nvidia.com/dynamo/dev/kubernetes/operations/benchmarking-with-ai-perf){:target="_blank"}
- [DynoSim으로 추론 구성 탐색하기](https://developer.nvidia.com/ko-kr/blog/dynosim-simulating-the-pareto-frontier/){:target="_blank"}
- [ai-dynamo/dynamo](https://github.com/ai-dynamo/dynamo){:target="_blank"} — Dynamo 저장소
