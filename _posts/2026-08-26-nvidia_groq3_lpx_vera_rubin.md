---
layout: post
comments: true
title: "NVIDIA Groq 3 LPX — Vera Rubin에서 긴 컨텍스트 초고속 인터랙티비티 구현"
description: "NVIDIA 테크니컬 블로그 해설. Groq 3 LPX가 컴파일러 사전 스케줄링과 C2C 통신으로 첫 비트 지연을 없애 Gemma 4 31B를 100K 컨텍스트에서 초당 3,431 토큰으로 서빙하는 원리와, Vera Rubin NVL72와의 결합 구성을 정리한다."
img: ai_abstract_title.jpg
date: 2026-08-26 18:20:00 +0900
last_modified_at: 2026-08-26 18:20:00 +0900
tags: [nvidia, groq-3-lpx, vera-rubin, long-context, inference, interactivity, agentic-ai, tensor-parallelism] # add tag
related: llm
categories: dev
---
어제 다룬 [Vera 스토리지 벤치마크]({{site.baseurl}}/dev/2026/08/25/nvidia_vera_storage_benchmark.html)가 AI 팩토리의 스토리지 계층 이야기였다면, 이번에는 추론 계층이다. NVIDIA 테크니컬 블로그의 [Groq 3 LPX 해설 글](https://developer.nvidia.com/ko-kr/blog/how-nvidia-groq-3-lpx-unlocks-ultrafast-interactivity-at-long-context-on-nvidia-vera-rubin/){:target="_blank"}(2026-08-19)을 읽고 정리했다. 긴 컨텍스트에서 왜 인터랙티비티(사용자당 토큰 속도)가 무너지는지, Groq 3 LPX는 그걸 어떻게 피하는지가 골자다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Groq 3 LPX는 256개의 LP30 LPU(Local Processing Unit), 총 128GB SRAM, 칩당 112Gbps로 동작하는 96개 C2C 링크로 구성된 추론 가속기다. 컴파일러가 실행 전에 클록 사이클 단위까지 통신 일정을 확정해 텐서 병렬화의 첫 비트 지연을 제거하고 Artificial Analysis 벤치마크 기준 Gemma 4 31B(밀집형)를 100K 컨텍스트에서 **초당 3,431 토큰**(중앙값)으로 서빙한다. Vera Rubin NVL72와 결합하면 프리필-디코드 분리 등으로 2조 파라미터급 모델까지 지원한다.

## 1. 문제 — 긴 컨텍스트에서 텐서 병렬화가 느려지는 이유

에이전틱 워크로드는 턴이 거듭될수록 컨텍스트가 누적된다. 컨텍스트가 100K를 넘어가면 디코드 단계에서 KV 캐시 접근과 칩 간 통신이 병목이 되고 사용자당 토큰 속도가 떨어진다. 텐서 병렬화(TP)로 칩을 늘리면 연산은 나눠지지만 칩 간 데이터 전송을 시작할 때마다 드는 고정 조정 비용 — 원문 표현으로 첫 비트 지연(first-bit latency) — 이 배치가 작은 인터랙티브 구간에서 이득을 갉아먹는다.

## 2. 해법 — 결정론적 실행과 컴파일러 사전 스케줄링

Groq 3 LPX의 접근은 런타임 중재를 없애는 것이다.

- **컴파일러 사전 스케줄링**: 워크로드 실행 전에 컴파일러가 클록 사이클 단위까지 실행 스케줄을 정확히 생성한다. C2C 전송 시점이 미리 확정되므로 실시간 전송 중재가 필요 없고 첫 비트 지연이 최소화된다.
- **세밀한 컴퓨팅-통신 중첩**: 320바이트 벡터 수준에서 컴퓨팅 유닛과 통신 유닛의 워크로드를 스케줄링한다. 행렬 전체가 끝나기를 기다리지 않고 결과가 준비되는 대로 전송을 시작하는 파이프라인이다.
- **LPU 라우팅**: 각 LPU가 프로세서이자 라우터 역할을 겸해 필요하면 다른 LPU를 경유해 데이터를 라우팅한다.
- **SRAM 기반 메모리**: 총 128GB SRAM에 가중치와 KV 캐시를 올려 메모리 접근 지연을 줄인다.

컨텍스트 길이가 변해도 지연 변동이 작다는 점도 결정론적 아키텍처의 부산물이다.

## 3. 벤치마크 수치

제3자 평가 기관 [Artificial Analysis](https://artificialanalysis.ai/){:target="_blank"}의 측정 결과다. 대상 모델은 Gemma 4 31B 밀집형.

| 조건 | 출력 속도 |
|---|---|
| 100K 컨텍스트 | 초당 3,431 토큰 (중앙값) |
| 10K 컨텍스트 | 초당 3,382 토큰 (중앙값) |
| SPEED-Bench 코딩 | 중앙값 4,767 토큰/초, P80 5,520 토큰/초 |

원문이 드는 체감 예시가 직관적이다. 에이전틱 코딩에서 5,000토큰을 디코드할 때 초당 100토큰이면 50초가 걸리지만 이 속도면 약 1.5초에 끝난다. 에이전트 루프처럼 디코드가 수십 번 반복되는 워크로드일수록 격차가 누적된다.

## 4. Vera Rubin NVL72와의 결합

Groq 3 LPX는 단독 제품이 아니라 [Vera Rubin 플랫폼](https://www.nvidia.com/ko-kr/data-center/technologies/rubin/){:target="_blank"}의 추론 계층으로 설계됐다. [Vera Rubin NVL72](https://www.nvidia.com/ko-kr/data-center/vera-rubin-nvl72/){:target="_blank"}와 결합하는 서빙 구성으로 원문은 세 가지를 든다.

- **프리필-디코드 분리**: 컨텍스트 처리(프리필)와 토큰 생성(디코드)을 서로 다른 하드웨어에 배치
- **어텐션-FFN 분리**: 레이어 구성 요소 단위로 역할을 나누는 구성
- **외부 드래프터 추측 디코딩**: 드래프트 모델을 별도 하드웨어에 두는 speculative decoding

이 조합으로 2조 파라미터로 확장된 GPT-OSS 모델 같은 초대형 멀티에이전트 시스템까지 커버한다는 게 원문의 주장이다. 아키텍처를 더 깊이 보고 싶다면 자매 글인 [Inside NVIDIA Groq 3 LPX](https://developer.nvidia.com/blog/inside-nvidia-groq-3-lpx-the-low-latency-inference-accelerator-for-the-nvidia-vera-rubin-platform/){:target="_blank"}(영문)가 있다.

## 5. 도입 관점 체크포인트

- **전용 하드웨어**: LPU 기반 독점 스택이라 기존 x86+GPU 클러스터에 드롭인할 수 없다. 랙 단위 공동 설계가 전제다.
- **모델 검증 범위**: 공개 수치는 Gemma 4 31B 기준이다. 다른 모델 계열에서 성능이 어떤지는 별도 확인이 필요하다.
- **비용 미공개**: 랙 구성 비용이 공개되지 않아 TCO 비교는 아직 어렵다. 전력·냉각·랙 스페이스를 포함한 전수명주기 기준으로 따져야 한다.

일반 GPU 환경에서도 방향성 자체는 참고할 만하다. 프리필-디코드 분리는 vLLM의 disaggregated prefill로 이미 실험할 수 있고 긴 컨텍스트 에이전트 서빙에서 디코드 지연이 병목이라는 문제의식은 하드웨어를 가리지 않는다.

