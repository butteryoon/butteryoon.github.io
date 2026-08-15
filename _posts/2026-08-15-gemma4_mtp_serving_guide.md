---
layout: post
comments: true
title: "Gemma 4 MTP 서빙 가이드 해설 — 페어드 드래프터·MIG 슬라이싱·vLLM 설정"
description: "Google Cloud의 Gemma 4 MTP(Multi-Token Prediction) 서빙 가이드 핵심 정리. 페어드 assistant 드래프터와 공유 KV-cache 구조, k=2에서 TPOT 14% 단축 벤치마크, G4(RTX PRO 6000 Blackwell) MIG 슬라이싱, vLLM speculative-config 배포와 운영 메트릭까지."
img: tools_title.jpg
date: 2026-08-15 22:56:00 +0900
last_modified_at: 2026-08-15 23:55:00 +0900
tags: [gemma, mtp, multi-token-prediction, vllm, mig, speculative-decoding, llm-serving, google-cloud] # add tag
related: llm
categories: tools
---

LLM 디코딩 지연의 근본 병목은 연산이 아니라 **GPU 메모리 대역폭**이다 — 토큰 하나를 만들 때마다 수십억 파라미터 가중치를 HBM에서 통째로 읽어야 한다. Google이 Developer 포럼에 공개한 **Gemma 4 MTP(Multi-Token Prediction) 서빙 가이드**는 이 병목을 드래프터 기반 투기적 디코딩으로 우회하는 실무 레시피다. 이 글은 그 가이드를 온프레미스 서빙 관점에서 재구성한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 대조·사실 정정 후 발행했다.)

<!--more-->

> **TL;DR:** Gemma 4(2026년 5월 공개: E2B/E4B/26B-MoE/31B)는 모델별로 **전용 MTP assistant 체크포인트**(~78M–156M, 4-레이어 드래프터)가 페어로 제공된다. 타겟의 top-layer hidden state·임베딩 테이블·**KV-cache를 공유**해 별도 드래프트 모델 방식의 VRAM 이중 부담이 없다. Google 실측(G4 Blackwell)에서는 **k=2가 최적** — 스텝당 수락 토큰 1.71배, **TPOT(토큰당 시간) 14% 단축**. 운영 목표치는 전체 acceptance 34~36%, 경보 임계 25% 미만.

## 1. 배경 — 왜 MTP인가

기존 투기적 디코딩(speculative decoding)은 별도 소형 드래프트 모델을 함께 띄워야 했다. 체크포인트 2벌 운영, 드래프터 전용 VRAM, 그리고 드래프터가 프롬프트를 처음부터 다시 인코딩하는 중복 연산이 따라온다. Gemma 4의 MTP assistant는 이 오버헤드를 구조적으로 제거한다.

| 항목 | 외부 드래프트 모델 방식 | Gemma 4 MTP assistant |
|------|---------------------------|-------------|
| 드래프터 | 독립 아키텍처 별도 운영 | 타겟별 페어 제공되는 4-레이어 드래프터 헤드 (~78M–156M) |
| 프롬프트 처리 | 드래프터가 처음부터 재인코딩 | 타겟의 top-layer hidden state(h_t)를 바로 받아 예측 |
| KV-cache | 드래프터·타겟 각각 유지 | **타겟 KV-cache 풀 공유** — 중복 캐시 0 |
| 임베딩 | 각자 보유 | **입력 임베딩 테이블 공유** (256k 어휘 = 수백 MB 절약) |

엣지 모델(E2B/E4B)의 assistant는 256k 어휘 로짓 프로젝션 비용을 줄이는 **clustered embedder**(어휘를 계층 클러스터로 묶어 계산량 축소)까지 얹었다.

MTP 계보를 짚어두면: 투기적 디코딩은 Google의 [Leviathan et al. 2022](https://arxiv.org/abs/2211.17192){:target="_blank"}, MTP 학습 목표는 Meta FAIR의 [Gloeckle et al. 2024](https://arxiv.org/abs/2404.19737){:target="_blank"}, 프런티어 규모 실증은 DeepSeek-V3/R1, 그리고 Gemma 4가 오픈 모델에 페어드 assistant 형태로 가져온 것이다.

## 2. 실측 벤치마크 — no MTP vs k=1 vs k=2

Google Cloud **G4 인스턴스(NVIDIA RTX PRO 6000 Blackwell)** 실측 결과다.

| 설정 | 스텝당 평균 수락 토큰 | 토큰 수율 | TPOT 개선 | E2E 지연 |
|------|-------------------|-----------|-----------|-----------|
| no MTP (k=0, 기준선) | 1.00 tok/step | — | 기준선 | 기준선 |
| k=1 | ~1.45 tok/step | +45% | 7% 빠름 | 7% 감소 |
| **k=2 (최적)** | **~1.71 tok/step** | **+71%** | **14% 빠름** | **14% 감소** |

- 수락률은 위치가 깊어질수록 지수적으로 감소한다 — position 0 ~47%, position 1 ~24%. 이 워크로드에서 k≥3은 검증 연산 낭비가 이득을 갉아먹어 제외됐다.
- 가이드의 권고: 자기 하드웨어에서 k를 올려가며 **MAL(Mean Acceptance Length)이 정체되는 지점에서 멈춰라**.
- 참고로 acceptance는 워크로드 의존성이 크다 — 일반 혼합 프롬프트 기준 34~36%가 참조치이고, 코드 생성·JSON 출력·반복적 파싱에서는 50%를 넘길 수 있다.

## 3. 하드웨어 슬라이싱 — G4 + MIG/vGPU와 에이전틱 추론

멀티 에이전트 마이크로서비스가 GPU를 나눠 쓰는 워크로드에서는 소프트웨어 워커 격리로는 L2 캐시·대역폭 경합(noisy neighbor)을 못 막는다. G4의 RTX PRO 6000 Blackwell(96GB)은 **카드당 최대 4개 MIG 슬라이스**(슬라이스당 ~24GB)와 vGPU 분할을 지원한다.

- **하드웨어 격리**: MIG/vGPU는 드라이버·하드웨어 수준에서 메모리 대역폭과 VRAM을 전용 할당 — 동시 에이전트 워크플로의 테일 레이턴시 보장
- **TCO**: MTP 워커 4개를 카드 하나에 격리 배치하면 카드당 QPS 4배, 필요 GPU 수 75% 절감 (가이드 주장)

이 구도는 온프레미스 Hopper GPU(H100/H200)에도 그대로 적용된다. 예컨대 H200 141GB는 NVIDIA MIG 가이드 기준 `1g.18gb`×7부터 `7g.141gb`×1까지 파티션할 수 있어, 대형 타겟 모델은 GPU를 독점시키고 임베딩·리랭커 같은 소형 모델을 작은 슬라이스에 격리 배치하는 식으로 밀도를 올릴 수 있다.

## 4. Gemma 4 패밀리와 페어드 assistant

2026년 5월 공개된 Gemma 4는 **gemma-4-E2B-it, gemma-4-E4B-it, gemma-4-26B-MoE-it(26B-A4B), gemma-4-31B-it** 네 타겟 크기로 나왔고, **각 타겟마다 전용 MTP assistant 체크포인트**(예: `google/gemma-4-E4B-it-assistant`)가 동봉된다.

페어링이 필요한 이유: 타겟 크기마다 내부 레이어 차원·어휘 구조가 다르고, assistant는 그 차원에 정확히 맞춰져야 타겟이 드래프트 토큰들을 **단일 포워드 패스로 병렬 검증**할 수 있다.

실행 흐름(k=2 예):

1. **Forward + Drafting (스텝 t)**: 타겟이 시퀀스를 처리해 top-layer 출력 생성 → 4-레이어 드래프터가 공유 임베딩·공유 KV-cache로 `t+1, t+2` 후보를 체인으로 예측
2. **병렬 검증 (스텝 t+1)**: 타겟이 드래프트 시퀀스 전체를 한 번의 포워드로 자기 분포와 대조
3. **수락/기각**: 첫 기각 토큰부터 나머지 드래프트는 폐기하고 타겟이 직접 생성. 디코딩이 대역폭 병목이라 여러 토큰 병렬 검증은 토큰 하나 생성과 비슷한 시간 — 단, 기각이 많으면 TPOT가 오히려 나빠진다

## 5. 파인튜닝 주의 — 드래프터 정렬 보존

도메인 파인튜닝(SQL 생성, 고객 지원, 구조화 파싱 등)에서 흔한 실패 모드: **타겟만 튜닝하고 assistant를 방치**하는 것. assistant는 계속 일반 도메인 토큰을 드래프트하므로 기각률이 치솟고, 전체 acceptance가 20% 아래로 떨어지면 MTP 검증 비용만 내고 가속은 못 받는 상태가 된다. 도메인 적응 시 assistant 재정렬(co-fine-tuning)을 파이프라인에 포함해야 한다.

## 6. 운영 관측성 — 모니터링 체크리스트

가이드가 제시하는 4개 텔레메트리(일반 혼합 워크로드 기준 참조치):

| 메트릭 | 참조 목표 | 경보/조치 |
|--------|-----------|--------|
| 전체 Speculative Acceptance Rate | 34~36% | **25% 미만 지속** → 도메인 드리프트 신호, 드래프터 재정렬 |
| MAL (스텝당 평균 수락 토큰) | ~1.70–1.72 (k=2) | 정체 지점에서 k 고정 |
| 위치별 수락 감쇠 | pos0 ~47% / pos1 ~24% | pos0 정상인데 pos1 < 15% → k를 1로 축소 |
| 순 TPOT/ITL 개선 | 10~15% 유지 | 미달 시 MTP 설정 재검토 |

## 7. vLLM 배포 — 실제 커맨드

가이드의 G4 MIG 슬라이스(1g.24gb) 기준 표준 레시피다. 포인트는 구식 `--speculative-model` 플래그가 아니라 **`--speculative-config` JSON에 `"method": "mtp"`**를 지정하는 것.

```bash
# G4 MIG 슬라이스(카드당 4개 중 1개) 안에서 실행
export FLASHINFER_ENABLE_SAMPLING=1
export CUDA_VISIBLE_DEVICES=0

python3 -m vllm.entrypoints.openai.api_server \
  --host 0.0.0.0 --port 9190 \
  --model /models/gemma4_e4b \
  --served-model-name gemma4_e4b \
  --tensor-parallel-size 1 \
  --max-model-len 16384 \
  --gpu-memory-utilization 0.90 \
  --enable-prefix-caching \
  --speculative-config '{"method": "mtp", "model": "/models/gemma-4-E4B-it-assistant", "num_speculative_tokens": 2}'
```

| 설정 | 이유 |
|------|------|
| `--gpu-memory-utilization 0.90` | MIG 슬라이스 자체가 하드웨어 격리를 보장하므로 슬라이스 메모리의 90%까지 안전하게 사용 |
| `FLASHINFER_ENABLE_SAMPLING=1` | Gemma 4는 이종 어텐션 헤드 차원(local 256/global 512)이라 어텐션은 TRITON_ATTN 강제 — FlashInfer는 샘플링 파이프라인만 가속 |
| `--speculative-config` | 페어드 assistant 경로 + `num_speculative_tokens: 2` → TPOT 14% 단축 |
| `--enable-prefix-caching` | 시스템 프롬프트·툴 스키마 재사용 — [Prompt Caching 글]({{site.baseurl}}/dev/2026/08/05/prompt-caching-guide.html)의 vLLM APC와 같은 메커니즘 |

## 마무리

Gemma 4 MTP의 요점은 "별도 드래프트 모델 없이, KV-cache·임베딩을 공유하는 페어드 assistant로, 검증 한 번에 여러 토큰"이다. 실무 순서는:

1. **k=2로 시작**하되 자기 워크로드에서 MAL 정체 지점 확인 (Google 실측 최적점이 k=2)
2. **acceptance 34~36%를 참조선**으로 — 25% 미만 지속이면 드래프터 정렬부터 의심
3. 도메인 파인튜닝 시 **assistant 동시 재정렬** 누락 금지
4. 멀티 에이전트 서빙이면 **MIG 슬라이스 격리**로 테일 레이턴시 확보

에이전트 파이프라인 쪽 맥락은 [Agent Skills 워크플로우]({{site.baseurl}}/tools/2026/08/11/agent_skills_engineering_workflows.html) 글과, GPU 인프라 쪽 맥락은 [NVIDIA AI Factory 해설]({{site.baseurl}}/dev/2026/07/30/nvidia_ai_factory_design_guide.html)과 이어진다.

## 참고

- [How to effectively serve MTP-based Gemma 4 models (Google Developer 포럼 원문)](https://discuss.google.dev/t/how-to-effectively-serve-mtp-based-gemma-4-models-for-inference-performance/389951){:target="_blank"}
- [vLLM Speculative Decoding 문서](https://docs.vllm.ai/en/latest/features/spec_decode.html){:target="_blank"}
- [Fast Inference from Transformers via Speculative Decoding (Leviathan et al.)](https://arxiv.org/abs/2211.17192){:target="_blank"}
- [Better & Faster LLMs via Multi-token Prediction (Gloeckle et al.)](https://arxiv.org/abs/2404.19737){:target="_blank"}
- [NVIDIA MIG 사용자 가이드](https://docs.nvidia.com/datacenter/tesla/mig-user-guide/){:target="_blank"}
