---
layout: post
comments: true
title: "NVIDIA Blackwell, 에이전틱 AI 인프라 벤치마크 AgentPerf에서 선두"
description: "Artificial Analysis의 에이전틱 AI 전용 벤치마크 AgentPerf에서 GB300 NVL72가 H200 대비 메가와트당 최대 20배의 에이전트 처리량을 기록한 결과를 정리한다."
img: command-title.webp
date: 2026-09-09 14:00:00 +0900
last_modified_at: 2026-09-09 14:00:00 +0900
tags: [nvidia, blackwell, agentic-ai, agentperf, artificial-analysis, gb300, inference, gpu] # add tag
related: llm
categories: dev
---
NVIDIA 블로그의 [「NVIDIA Blackwell, 업계 최초 에이전틱 AI 인프라 벤치마크에서 선두 기록」](https://blogs.nvidia.co.kr/blog/nvidia-blackwell-agentperf-artificial-analysis/){:target="_blank"}(2026-06-15)을 읽고 핵심만 정리했다. 앞서 다룬 [SemiAnalysis AgentX 벤치마크 글]({{site.baseurl}}/dev/2026/08/29/nvidia_vera_rubin_blackwell_agentx_perf_per_watt.html)과 짝을 이루는 내용으로, 이번엔 **Artificial Analysis**의 **AgentPerf**다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 단일 LLM 호출 측정에서 벗어나 연쇄 호출·툴 사용이 반복되는 에이전틱 워크로드 전용 벤치마크 **AgentPerf**(Artificial Analysis)가 나왔다. **GB300 NVL72**는 DeepSeek V4 Pro 기준 **HGX H200 대비 메가와트당 최대 20배** 많은 동시 에이전트를 처리했다. 랙 스케일 NVLink, 통신-연산 중첩 커널, TensorRT-LLM의 프리필/디코드 분리 최적화가 격차를 만든다.

## 1. 왜 에이전트 전용 벤치마크인가

단순 챗봇은 요청 한 번에 응답 한 번이지만, 에이전틱 워크로드는 목표 분해 → 추론 → 툴 호출 → 관찰 → 재추론의 루프가 이어지는 릴레이 방식이다. AgentPerf는 실제 코딩 에이전트의 궤적(파일 읽기, 코드 수정, 명령 실행)을 그대로 본떠 설계했고, 응답성과 출력 토큰 비율(SLO)을 지키면서 **동시에 굴릴 수 있는 에이전트 수**를 잰다.

## 2. 결과 — 메가와트당 20배

GB300 NVL72는 DeepSeek V4 Pro를 돌릴 때 HGX H200 시스템보다 **메가와트당 최대 20배** 많은 동시 에이전트를 감당했다. 격차는 세 겹에서 나온다.

- **하드웨어:** GPU 72개를 NVLink로 랙 하나에 묶어 대규모 MoE 모델의 분산 실행을 효율화
- **커널:** 통신과 연산을 중첩(overlap)해 전문가(expert) 간 조율 비용을 지연 없이 흡수
- **소프트웨어:** TensorRT-LLM이 입력 처리(prefill)와 출력 생성(decode)을 따로 최적화해 세션이 늘어도 효율 유지

실제 적용 사례로는 Baseten, Together AI(Cursor 에이전틱 코딩 플랫폼의 실시간 추론), DeepInfra(자동차 대리점 AI 인력 플랫폼 Pam.ai)를 든다.

## 3. AgentX와 뭐가 다른가

같은 "에이전틱 벤치마크"지만 주체와 방법이 다르다. [AgentX(SemiAnalysis)]({{site.baseurl}}/dev/2026/08/29/nvidia_vera_rubin_blackwell_agentx_perf_per_watt.html)는 미리 녹화해둔 Claude Code 세션을 턴 단위로 재현하고, AgentPerf(Artificial Analysis)는 코딩 에이전트 궤적을 놓고 SLO를 충족하는 동시 에이전트 수를 센다. 그래도 결론은 한곳으로 모인다 — 에이전트 시대의 인프라 지표는 절대 처리량이 아니라 **전력당 동시 에이전트 수**다.

## 4. 읽으면서 든 생각

- 에이전트 시스템에서 중요한 건 단건 응답 속도가 아니라, 툴 호출 루프가 반복될 때 시스템 전체가 내는 처리량과 전력 효율이다.
- 다만 원문도 벤치마크의 한계를 짚는다. AgentPerf는 툴 호출 시간을 실제로 실행하지 않고 **대표적인 CPU 처리 시간으로 시뮬레이션**한다. 실환경에서는 외부 API 지연이나 DB 쿼리 같은 외부 I/O가 먼저 병목이 되기 쉬우니, 이 수치를 그대로 용량 계획에 옮기면 곤란하다.
- 전력 효율이 20배라 해도 GB300 NVL72의 도입 비용과 MW 단위 전력 요구는 그대로 남는다. 예상 동시 에이전트 수를 기준으로 TCO부터 따져봐야 한다.

## 참고

- [원문: NVIDIA Blackwell AgentPerf 결과](https://blogs.nvidia.co.kr/blog/nvidia-blackwell-agentperf-artificial-analysis/){:target="_blank"}
- [NVIDIA 개발자 블로그](https://developer.nvidia.com/blog/){:target="_blank"}
- [이전 글: AgentX 벤치마크로 본 와트당 에이전틱 AI 성능]({{site.baseurl}}/dev/2026/08/29/nvidia_vera_rubin_blackwell_agentx_perf_per_watt.html)
