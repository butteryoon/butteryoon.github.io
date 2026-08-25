---
layout: post
comments: true
title: "NVIDIA Vera 스토리지 벤치마크 — AI 네이티브 스토리지의 암호화·압축·무결성 검사 가속"
description: "NVIDIA 테크니컬 블로그 해설. Vera CPU(Olympus 코어 88개) 기반 BlueField-4 STX 스토리지 프로세서가 x86 대비 암호화 1.43배, Reed-Solomon 복구 3.26배, CRC32C 3.67배, 압축 3.29배의 처리량을 내는 벤치마크 결과와 방법론을 정리한다."
img: ai_abstract_title.jpg
date: 2026-08-25 18:25:00 +0900
last_modified_at: 2026-08-25 18:25:00 +0900
tags: [nvidia, vera-cpu, bluefield-4, stx, storage, olympus-cores, encryption, compression, reed-solomon] # add tag
related: llm
categories: dev
---

[NVIDIA AI Factory 설계 가이드]({{site.baseurl}}/dev/2026/07/30/nvidia_ai_factory_design_guide.html)에서 다룬 인프라 스택 중 스토리지 계층의 후속 소식이다. NVIDIA 테크니컬 블로그의 [Vera 스토리지 벤치마크 글](https://developer.nvidia.com/blog/nvidia-vera-storage-benchmarks-faster-encryption-compression-integrity-checking-and-recovery-for-ai-native-storage/){:target="_blank"}(2026-08-03, Jason Hardy·Ronil Prasad)을 해설한다. 스토리지 시스템이 매일 수행하는 암호화·압축·무결성 검사·복구 작업을 x86 대신 Vera CPU로 돌렸을 때의 처리량 비교다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** NVIDIA 설계 Olympus 코어 88개를 탑재한 Vera CPU가 BlueField-4 STX 스토리지 프로세서에 실려 스토리지 데이터 경로에 들어온다. x86 대비 처리량은 AES-128 암호화 최대 **1.43배**, 복호화 **1.29배**, Reed-Solomon 복구 **3.26배**, CRC32C 무결성 검사 **3.67배**, 압축 **3.29배**, 압축 해제 **1.72배**, 압축+암호화 2단계 파이프라인 **3.21배**다. 에이전틱 AI 워크로드에서 스토리지 CPU가 병목이 되는 구간을 겨냥한 결과다.

## 1. 하드웨어 — Vera CPU와 Olympus 코어

[Vera CPU](https://www.nvidia.com/ko-kr/data-center/vera-cpu/){:target="_blank"}는 NVIDIA가 직접 설계한 Olympus 코어 88개(Spatial Multithreading으로 176 스레드), Armv9.2 호환 프로세서다. SCF(Scalable Coherency Fabric)가 최대 3.4 TB/s의 이분 대역폭을 제공하고 164 MB 통합 L3 캐시와 SOCAMM2 LPDDR5X 메모리로 최대 1.2 TB/s(코어당 최대 14 GB/s)의 메모리 대역폭을 낸다.

설계 의도는 두 가지 상반된 요구를 한 코어에서 충족하는 것이다. 스토리지 소프트웨어의 제어 집약적 코드는 높은 단일 스레드 성능을, 데이터 처리 코드는 높은 대역폭과 예측 가능한 레이턴시를 요구한다. 통합 L3와 SCF, Spatial Multithreading의 조합으로 동시 스트림을 처리하면서 스레드 간 간섭을 줄였다는 게 아키텍처 쪽 설명이다. 코어 설계 자체가 궁금하다면 자매 글인 [Olympus 코어 해설](https://developer.nvidia.com/ko-kr/blog/inside-nvidia-vera-cpu-olympus-cores-built-for-maximum-single-threaded-performance-in-agentic-ai/){:target="_blank"}(2026-07-24)을 함께 보면 좋다.

## 2. 벤치마크 결과와 방법론

x86 대비 처리량 향상 폭은 다음과 같다.

| 작업 | x86 대비 |
|---|---|
| AES-128 암호화 | 최대 1.43배 |
| AES-128 복호화 | 최대 1.29배 |
| Reed-Solomon 복구(erasure coding) | 최대 3.26배 |
| CRC32C 무결성 검사 | 최대 3.67배 |
| 압축(Zstandard/LZ4) | 최대 3.29배 |
| 압축 해제 | 최대 1.72배 (동시성 증가 시 우위 확대) |
| 압축+암호화 2단계 파이프라인 | 최대 3.21배 |

방법론을 읽을 때 유의할 조건이 있다. 각 테스트는 메모리에 이미 올라온 데이터를 단일 프로세스에서 처리하는 방식으로 파일 I/O·디스크·네트워크 병목은 배제됐다. OpenSSL, Zstandard, LZ4 등 표준 라이브러리에 네이티브 명령어 최적화를 적용한 구현이고 소스코드·스크립트·고정 소프트웨어 버전으로 재현 가능하다고 명시되어 있으나 공개 표준 벤치마크는 아니다. 즉 이 수치는 CPU 연산 구간의 상한이며 엔드투엔드 스토리지 성능은 SSD·네트워크를 포함한 별도 검증이 필요하다.

## 3. 왜 지금 스토리지 CPU인가

NVIDIA가 이 벤치마크를 에이전틱 AI 맥락에 놓는 이유가 있다. 기업 지식 검색, 에이전트의 영구 메모리, KV 캐시 재사용, 툴 실행 같은 워크플로우에서 스토리지는 단순 보관소가 아니라 추론 루프에 데이터를 공급하고 상태를 보존하는 역할을 한다. 동시 에이전트 수와 컨텍스트 윈도우가 커질수록 암호화·압축·무결성 검사 같은 CPU 측 작업이 병목이 된다.

도입 관점의 체크포인트는 이렇다.

- **하드웨어 종속**: Vera CPU는 BlueField-4 STX 스토리지 프로세서에 탑재되며 기존 x86 스토리지 서버의 교체 또는 신규 도입이 전제된다. SOCAMM2 메모리는 현장 교체가 가능하지만 전용 폼팩터다.
- **소프트웨어 포팅**: NVIDIA DOCA 스토리지 프레임워크와 [STX 파운데이션](https://www.nvidia.com/ko-kr/data-center/ai-storage/stx/){:target="_blank"} 위에서 동작하며 기존 x86 기반 SDS·분산 파일시스템은 Armv9.2 네이티브 최적화를 포함한 포팅 작업이 필요하다.
- **TCO 계산**: 압축 처리량 3.29배는 스토리지 용량·대역폭 요구 감소로, 와트당 성능 향상은 전력·냉각 비용 절감으로 이어질 수 있다. 다만 BlueField-4 STX 단가는 공개되지 않았다.

스토리지 계층까지 Arm 기반 자사 실리콘으로 채우려는 NVIDIA의 방향성이 잘 드러나는 글이다. 벤치마크 조건의 한계를 감안하더라도 AI 인프라에서 스토리지 CPU 선택이 설계 변수가 되기 시작했다는 신호로 읽을 만하다.
