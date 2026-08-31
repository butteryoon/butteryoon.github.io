---
layout: post
comments: true
title: "NVIDIA 양자화 기술과 TensorRT-LLM으로 MoE 모델 서빙 최적화 — Kakao Kanana-Flex FP8 사례"
description: "NVIDIA 테크니컬 블로그 해설. TensorRT-LLM과 TensorRT-Model-Optimizer로 Hopper(H200) 위에서 MoE 모델(Kanana-Flex)을 FP8 tensor-wise 양자화해 처리량 43%↑, TTFT 34%↓ 달성한 엔드투엔드 최적화 파이프라인을 정리한다."
img: nvidia-tensorrt-fp8-moe.webp
date: 2026-08-30 18:01:17 +0900
last_modified_at: 2026-08-30 21:25:00 +0900
tags: [nvidia, tensorrt-llm, fp8, quantization, moe, tensorrt-model-optimizer, hopper, h200, kanana-flex, llm-serving] # add tag
related: llm
categories: dev
---

NVIDIA 테크니컬 블로그에 올라온 [「NVIDIA 양자화 기술과 TensorRT-LLM을 이용한 서비스 최적화」](https://developer.nvidia.com/ko-kr/blog/nvidia-quantization-technology-and-tensorrt-llmbased-service-optimization/){:target="_blank"}를 읽고 핵심 기술 스택과 실측 수치를 정리했다. Kakao가 개발한 Kanana-Flex MoE 모델을 H200 8-GPU에서 FP8 tensor-wise로 양자화·서빙하며 TensorRT-LLM이 SGLang·vLLM 대비 처리량과 지연 모두에서 우위를 보인 드문 실증 사례다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고 Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Kakao Kanana-Flex MoE 모델을 TensorRT-Model-Optimizer로 **FP8 tensor-wise 양자화**한 뒤 TensorRT-LLM `trtllm-serve`로 H200 8-GPU(TP=8, EP=8)에서 서빙하면 **BF16 대비 처리량 43% 향상(10.95 req/s), p99 TTFT 34% 단축(5.7s)**을 얻는다. 동일 하드웨어에서 SGLang·vLLM보다 처리량·TTFT·ITL 전 지표 우위. 핵심은 **Hopper FP8 tensor-wise 네이티브 지원 + Grouped GEMM + TMA/Warp Specialization + XQA 커널 + CUDA Graph + Overlap Scheduler** 조합이다.

## 1. 배경 — MoE 서빙의 병목과 ROI 문제

최신 LLM 서빙에서 가장 큰 고민은 투자 대비 수익(ROI)이다. 학습 땐 고성능 GPU를 쓰지만 서빙 단계에선 효율이 급락한다. 특히 MoE(Mixture-of-Experts) 모델은 전문가 네트워크 간 통신 병목, 동적 라우팅 처리에 따른 지연, 가변적인 expert 활성화로 인한 GPU 활용률 저하가 겹친다.

이 때문에 H200 같은 최신 GPU를 써도 처리량(throughput)과 지연 시간(latency) 요구를 동시에 맞추기 어렵다. NVIDIA는 이 문제를 **TensorRT-LLM**의 MoE 전용 최적화 커널과 **FP8 양자화**로 풀었다.

## 2. NVIDIA 기술 스택 — 4대 핵심 컴포넌트

### 2.1 NVIDIA TensorRT-LLM

오픈소스 추론 라이브러리다. 원문이 꼽는 특징은 이렇다.

- TensorRT 컴파일러 + 고성능 GPU 커널 + 메모리 최적화로 기존 대비 최대 8배까지 속도 향상
- 멀티 GPU/노드 지원(TP, PP, EP, DEP 병렬화 전략)
- BF16, FP16, INT8, FP8, MXFP4 다중 정밀도 양자화
- Python API로 CUDA/C++ 없이 모델 최적화·배포 가능
- `trtllm-serve`로 OpenAI API 호환 서빙 즉시 제공

### 2.2 MoE 아키텍처 최적화 (3대 축)

| 최적화 축 | 핵심 기술 | 효과 |
|-----------|-----------|------|
| **라우팅** | 병렬 리덕션(Parallel Reduction) + 공유 메모리 최적화 | Top-K expert 분배 저지연, 토큰 수별 싱글/클러스터/멀티패스 변형 |
| **Grouped GEMM** | Expert 연산 묶음 병렬 실행 + **Finalize Fusion** | 메모리 접근 감소, **TMA(Tensor Memory Accelerator)** + **Warp Specialization**으로 연산-메모리 오버랩 |
| **병렬화** | TP + EP + **DEP(Data & Expert Parallelism)** | 대규모 모델 확장성 확보 |

### 2.3 XQA (eXtended Query Attention) 커널

기존 MQA/GQA 구현의 한계(L2 대역폭 병목, FP8→FP32 변환 오버헤드, Flash Attention 미사용)를 이렇게 해결한다.

- K/V 헤드 기준으로 워크로드를 분할해 K/V 데이터를 1회만 로드 (Q 헤드당 반복 로드 제거)
- Tensor 코어 수학 연산으로 계산 성능을 끌어올림
- DRAM 대역폭 극대화, 빠른 JIT 컴파일, 자원 효율성 향상

### 2.4 TensorRT-Model-Optimizer (FP8 양자화 도구)

- PTQ(Post-Training Quantization): INT8, FP8, INT4 + 희소성
- PyTorch/ONNX 모델을 TensorRT-LLM/TensorRT 배포 가능 체크포인트로 변환
- **FP8 tensor-wise 양자화**: 각 텐서별 독립 스케일링 팩터로 정확도 손실 최소화
- Blackwell, Hopper, Ampere, Ada Lovelace 지원

## 3. Kakao Kanana-Flex FP8 양자화·서빙 파이프라인

### 3.1 1단계: FP8 양자화 변환

```bash
# TensorRT-Model-Optimizer 클론
git clone https://github.com/NVIDIA/TensorRT-Model-Optimizer.git

# 패키지 설치
pip install -U "nvidia-modelopt[all]"

# BF16 MoE → FP8 tensor-wise 양자화
cd TensorRT-Model-Optimizer/examples/llm_ptq
./scripts/huggingface_example.sh --model ./Kanana-Flex --quant fp8 --export_fmt hf
```

핵심은 Hugging Face 기본 **block-wise FP8**을 Hopper 네이티브 **tensor-wise FP8**로 변환해야 최적 성능이 나온다는 점이다.

### 3.2 2단계: 엔진 빌드 및 서빙

```bash
trtllm-serve serve ./Kanana-Flex_fp8 \
  --backend pytorch \
  --max_batch_size 128 \
  --host 0.0.0.0 --port 13121 \
  --max_num_tokens 8192 --max_seq_len 8192 \
  --tp_size 8 --ep_size 8 \
  --kv_cache_free_gpu_memory_fraction 0.8 \
  --num_postprocess_workers 4 \
  --extra_llm_api_options ./extra_config.yaml
```

`extra_config.yaml`로 런치 오버헤드를 제거한다.

```yaml
pytorch_backend_config:
  use_cuda_graph: true
  disable_overlap_scheduler: false
```

- **CUDA Graph**: 커널 발사 오버헤드 제거
- **Overlap Scheduler**: 프리필/디코드 파이프라인 중첩으로 GPU 자원 효율 극대화

## 4. 성능 벤치마크 — 실측 수치로 보는 차이

### 4.1 BF16 베이스라인 (H200 8-GPU)

| 플랫폼 | Throughput (req/s) | p99 TTFT (ms) |
|--------|---------------------|---------------|
| SGLang | 2.98 | 21,179 |
| **TensorRT-LLM** | **8.98** | **7,792** |
| vLLM | 7.54 | 9,631 |

*각 플랫폼 최고 성능 버전 기준(SGLang 0.4.6, TensorRT-LLM 0.21.0rc1, vLLM 0.9.1 — 버전별 상세는 원문 표 참조).*

TensorRT-LLM이 처리량과 TTFT 모두 1위다.

### 4.2 FP8 양자화 적용 시 향상율 (BF16 대비)

| 플랫폼 | 처리량 ↑ | p99 TPOT ↑ | p99 ITL ↑ | TTFT ↑ |
|--------|----------|------------|-----------|--------|
| SGLang | +21.0% | -37.9% | +5.5% | -9.7% |
| **TensorRT-LLM** | **+36.9%** | **+32.2%** | **+12.5%** | **+29.5%** |
| vLLM | +0.6% | +0.8% | +0.3% | +16.3% |

TensorRT-LLM만 전 지표가 개선됐다. SGLang은 TPOT·TTFT가 악화됐고 vLLM은 개선 폭이 미미하다.

### 4.3 Kanana-Flex 모델별 상세 비교

| 모델 (데이터타입) | 처리량 (req/s) | p99 TTFT (ms) | 토큰 처리량 (tokens/s) |
|-------------------|----------------|---------------|------------------------|
| Kanana-Flex (BF16) | 8.98 | 8,619 | 35,062 |
| Kanana-Flex (FP8 block-wise) | 6.16 | 9,769 | 28,207 |
| **Kanana-Flex (FP8 tensor-wise)** | **10.95** | **5,713** | **51,593** |

FP8 tensor-wise가 BF16 대비 처리량 +43%, TTFT -34%를 기록했다. 반면 block-wise는 오히려 성능이 떨어진다 — Hopper가 tensor-wise에 최적화돼 있다는 방증이다.

## 5. FP8 캘리브레이션 — 정확도 지키는 비결

단일 스케일링 팩터로는 텐서별 값 분포 차이 때문에 오버플로우/언더플로우가 생겨 정확도가 무너진다. Tensor-wise quantization은 각 텐서(가중치, 활성화 등)의 고유 값 범위를 분석해 독립 스케일링 팩터를 적용하고 8비트 표현 범위 안에 정보를 최대한 보존한다.

캘리브레이션 예시 (실제 유스케이스 약 1,000문장):

```bash
cd TensorRT-Model-Optimizer/examples/llm_ptq
python hf_ptq.py \
  --pyt_ckpt_path=./Kanana-Flex_fp8 \
  --export_path=./Kanana-Flex_fp8-ptq \
  --qformat=fp8 --export_fmt=hf
```

정확도 검증 결과가 흥미롭다. 한국어/다국어 IF(Instruction Following)·NR(Non-reasoning) 테스트에서 FP8 tensor-wise가 **83.17점**으로 BF16의 **81.67점**보다 오히려 1.5점 높았다. 단순 경량화를 넘어 품질까지 지킨(오히려 개선한) 사례다.

## 6. 직접 적용 시 체크리스트

| 단계 | 필수 조치 | 주의사항 |
|------|-----------|----------|
| **모델 준비** | HF block-wise FP8 → tensor-wise 변환 | `modelopt` PTQ 파이프라인 필수 |
| **엔진 빌드** | `--tp_size --ep_size` GPU 토폴로지에 맞게 | DEP 전략 고려 시 `trtllm-build` 옵션 확인 |
| **서빙 설정** | `use_cuda_graph: true`, `disable_overlap_scheduler: false` | KV cache fraction(`--kv_cache_free_gpu_memory_fraction`) 튜닝 필요 |
| **캘리브레이션** | 도메인 데이터 500~2,000 샘플 | 너무 적으면 정확도 하락, 많으면 시간만 증가 |
| **벤치마크** | ISL/OSL 고정이 아닌 **에이전틱 워크로드 재현**(AgentX 등) | TTFT·TPOT·ITL·Throughput 동시 측정 |

## 7. 결론 — MoE 서빙의 새로운 기준

Kakao 사례가 보여주는 건 **하드웨어(Hopper H200) + 컴파일러(TensorRT-LLM) + 양자화(TensorRT-Model-Optimizer) + 커널(XQA/Grouped GEMM/TMA)**의 풀스택 최적화만이 MoE 서빙의 ROI 문제를 푼다는 점이다.

- 단일 기술로는 부족하다: 양자화만 하면 block-wise 함정에 빠지고 커널만 손대면 메모리 병목이 남는다
- 엔드투엔드 파이프라인이 핵심이다: PTQ → tensor-wise 변환 → TRT-LLM 빌드 → CUDA Graph/Overlap Scheduler 서빙
- 정확도 트레이드오프가 없다: FP8 tensor-wise가 BF16보다 정확도까지 높였다

MoE 모델을 H200/H100급 GPU에서 서빙 중이라면 **TensorRT-LLM + FP8 tensor-wise** 조합을 PoC 1순위로 테스트해볼 만하다. [Model-Optimizer PTQ 예제](https://github.com/NVIDIA/Model-Optimizer/tree/main/examples/hf_ptq){:target="_blank"}와 [trtllm-serve 가이드](https://nvidia.github.io/TensorRT-LLM/commands/trtllm-serve.html){:target="_blank"}가 좋은 출발점이다.
