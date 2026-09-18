---
layout: post
comments: true
title: "Claude가 생체분자 모델 30개를 4배 빠르게 만든 4주"
description: "앤스로픽이 공개한 생체분자 모델링 가속 결과를 정리한다. FlashPairformer 커널, 단일 노드로 7만 토큰을 다루는 Big 모드, 그리고 150달러로 끝낸 단백질 설계 캠페인."
img: claude-biomolecular-modeling_title.webp
date: 2026-09-18 20:20:00 +0900
last_modified_at: 2026-09-18 20:20:00 +0900
tags: [anthropic, claude, biomolecular-modeling, protein-design, kernel-optimization, gpu, llm]
related: llm
categories: dev
---
앤스로픽 리서치의 [「How Claude is uplifting biomolecular modeling」](https://www.anthropic.com/research/claude-uplifts-biomolecular-modeling){:target="_blank"}을 읽고 정리했다. 눈길이 간 건 4배라는 숫자 자체가 아니라, 그 일을 해낸 인력 구성이다. 커널 엔지니어링 경험이 없는 연구원 두 명이 감독을 맡았고 최적화는 Claude가 했다. 며칠 전 다룬 [NVIDIA BioNeMo Agent Toolkit 글]({{site.baseurl}}/dev/2026/09/15/nvidia-bionemo-agent-toolkit.html)이 이미 빠른 GPU 도구를 에이전트 손에 쥐여주는 이야기였다면, 이번 건은 에이전트가 그 도구 자체를 다시 깎은 쪽에 가깝다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Claude가 4주가 채 안 되는 기간에 오픈소스 생체분자 모델 30여 개를 평균 약 4배 가속했다. 출력이 완전히 같은 조건으로 좁히면 약 1.6배다. 새로 만든 FlashPairformer 커널은 업계 표준 대비 triangle attention에서 2.7~2.9배, triangle multiplication에서 1.7~3.2배 앞섰다. 저메모리 Big 모드는 GPU 노드 하나로 1만 토큰이 넘는 시스템을 정확히 모델링하고 7만 토큰이 넘는 시스템까지 추론을 성공시켰다. 단백질 설계 캠페인은 타깃당 1만 달러 예산을 쓰던 이전 방식과 맞먹는 ipSAE를 H200 한 장 24시간, 약 150달러로 재현했다.

## 병목은 삼각형 연산에 있다

AlphaFold3, OpenFold3, Boltz-2 같은 현대 구조 예측 모델의 계산을 잡아먹는 건 triangle attention과 triangle multiplication이다. 토큰 삼중항 위에서 도는 연산이라 시간과 메모리가 모두 3차로 불어난다. 시스템 크기가 2배면 비용은 8배, 3배면 27배다. 단백질 하나를 접는 것과 리보솜을 접는 것 사이의 간극이 여기서 벌어진다.

이 지점을 파고든 커널 최적화는 이미 있었다. NVIDIA의 cuEquivariance와 BioNeMo Inference Runtime이 사실상 표준 자리를 지켜왔다. 앤스로픽이 내놓은 FlashPairformer는 그 표준을 넘어섰다.

| 연산 | 업계 표준 대비 |
|------|---------------|
| Triangle attention | 2.7~2.9배 |
| Triangle multiplication | 1.7~3.2배 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"outperforming the field standard on average by 2.7-2.9x on triangle attention and 1.7-3.2x on triangle multiplication"</blockquote></details>

전략은 두 갈래다. 여러 모델에 그대로 얹을 수 있는 범용 커널을 하나 만들고 그 위에 모델별 맞춤 최적화를 덧붙였다. 캐싱을 넣거나 실행되지 않는 분기를 상수로 접어버리는 식의 손질이다. 가속된 버전이 다운스트림 정확도를 떨어뜨리지 않는지도 함께 확인했다.

## 30개 모델, 4주, 감독 두 명

대상은 구조 예측만이 아니었다. 단백질 설계, 단백질 언어 모델, 지노믹스까지 30개가 넘는 오픈소스 모델을 건드려 평균 약 4배를 뽑았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"optimized more than 30 of these models in just under four weeks, speeding them up roughly 4x on average"</blockquote></details>

4배라는 수치는 조건을 봐야 한다. 출력이 한 비트도 달라지지 않는 엄격한 기준으로 재면 약 1.6배로 내려간다. 나머지는 수치적으로 동등한 범위 안에서 얻은 여유다. 그래도 모델 하나당 몇 주씩 걸리던 작업을 30개 넘게, 4주 안에 끝냈다는 사실은 남는다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"roughly 1.6x speed-up with identical outputs"</blockquote></details>

감독한 사람은 둘이다. 생체분자 모델링은 알지만 추론 최적화는 해본 적이 없는 직원들이었다. 커널을 깎는 쪽 전문성은 전부 모델이 채웠다는 뜻이다.

## Big 모드: 노드 하나로 리보솜을 접는다

메모리를 아끼는 Big 모드가 이번 발표에서 가장 실용적인 부분이다. 노드 하나로 1만 토큰이 넘는 시스템을 정확히 모델링하고 7만 토큰이 넘는 시스템까지 추론 자체를 성공시켰다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"enables the accurate modeling of systems larger than 10,000 tokens and successful inference on systems larger than 70,000 tokens using just one NVIDIA GPU node"</blockquote></details>

감이 잘 안 오면 비교 대상을 보면 된다. AlphaFold3가 정확히 예측해낸 40S 리보솜이 7,663 토큰이었다. 1만 토큰은 그 위의 영역이다. Big 모드로 접어낸 분자 기계 목록에는 인간 미토콘드리아 복합체 I, TRiC 샤페론 복합체, 프로테아솜, 박테리아 리보솜이 들어간다. 모두 실험으로 결정된 구조와 가깝게 맞았다.

8-GPU B300 노드 한 대를 쓴 실험에서는 바이러스 캡시드와 단백질 구획 전체를 예측했다. 크기는 3만 1천 토큰부터 7만 토큰 이상까지였다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Using a single 8-GPU B300 node, Claude generated predictions of entire viral capsids and protein compartments ranging in size from more than 31,000 to more than 70,000 tokens."</blockquote></details>

종전에는 이 규모를 다루려면 여러 노드에 걸친 분산 추론을 짜야 했다. 노드 하나로 내려왔다는 건 장비가 아니라 진입 장벽 이야기다.

## 단백질 설계: 1만 달러에서 150달러로

비용 쪽 결과가 더 극적이다. 이전 캠페인은 타깃당 1만 달러 예산을 썼다. H100 시간으로 대략 2,500시간이다. 프롬프트는 1만 6천 단어쯤 됐고 서브 에이전트를 여럿 굴렸다.

가속된 모델을 쓴 새 방식은 H200 한 장으로 24시간, 프롬프트는 1,100단어 남짓, 서브 에이전트 없이 돌렸다. ipSAE 결합 점수는 비슷하게 나왔고 GPU와 토큰을 합친 비용은 약 150달러였다. 예산 기준으로 70분의 1 수준이다.

| | 이전 캠페인 | 가속 모델 적용 |
|---|---|---|
| 타깃당 예산 | 1만 달러 (~2,500 H100시간) | H200 1장, 24시간 |
| 프롬프트 | ~16,000 단어 | ~1,100 단어 |
| 서브 에이전트 | 사용 | 미사용 |
| 총 비용 | ~1만 달러 | 약 150달러 |

같은 조건에서 Mythos 5.1, Mythos 5, Opus 5 세 모델을 16개 타깃에 돌렸다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"We ran three Claude models (Mythos 5.1, Mythos 5, and Opus 5) against 16 targets with the accelerated biomolecular models described in this post."</blockquote></details>

프롬프트가 1만 6천 단어에서 1,100단어로 줄고 서브 에이전트가 사라졌다는 대목이 흥미롭다. 모델과 도구가 빨라지면 그걸 감싸던 오케스트레이션 배관도 같이 얇아진다는 신호로 읽힌다.

## 코드와 대회

최적화 코드는 [GitHub](https://github.com/anthropics/uplifting-biomolecular-modeling){:target="_blank"}에 공개됐다. 기존 파이프라인에 커널을 갈아 끼우는 방식이라 모델을 다시 학습시킬 필요는 없다.

여기에 Adaptyv Bio와 [단백질 설계 대회](https://proteinbase.com/competitions/anthropic-adaptyv-2026){:target="_blank"}를 연다. 상금은 최대 100만 달러 규모의 Claude 크레딧과 25만 달러어치 Modal 컴퓨트 크레딧이고 5,000건이 넘는 설계를 습식 실험으로 검증한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"up to $1 million in Claude credits" ... "wet lab validation for over 5,000 designs"</blockquote></details>

생물학 연구자에게 프론티어 모델을 제공하는 Life Sciences Verification Program도 함께 언급됐다.

## 남는 생각

숫자보다 인력 구성이 이 발표의 핵심이라고 본다. 커널 최적화는 오랫동안 소수 전문가의 영역이었고 그래서 고성능 생체분자 모델링도 큰 랩과 빅테크 쪽에 몰려 있었다. 그 둘을 이어주던 병목을 모델이 대신 메울 수 있다면, 작은 랩이 손댈 수 있는 범위가 꽤 넓어진다.

다만 4배와 1.6배의 간극은 기억해둘 만하다. 자기 파이프라인에 얹을 때 어느 쪽 숫자를 기대해야 하는지는 출력 동등성을 어디까지 요구하느냐에 달렸다.

## 참고

- 원문: [How Claude is uplifting biomolecular modeling](https://www.anthropic.com/research/claude-uplifts-biomolecular-modeling){:target="_blank"}
- 기술 리포트 PDF: [Anthropic](https://www-cdn.anthropic.com/c03643714397d9d396fa1ce1794f5f9f7863a82c.pdf){:target="_blank"}
- 소스 코드: [anthropics/uplifting-biomolecular-modeling](https://github.com/anthropics/uplifting-biomolecular-modeling){:target="_blank"}
- 대회: [Anthropic × Adaptyv Bio 2026](https://proteinbase.com/competitions/anthropic-adaptyv-2026){:target="_blank"}
- 이전 연구: [Claude accelerates protein design](https://www.anthropic.com/research/Claude-accelerates-protein-design){:target="_blank"}
- 비교 대상 커널: [cuEquivariance](https://github.com/nvidia/cuequivariance){:target="_blank"}, [BioNeMo Inference Runtime](https://github.com/NVIDIA-BioNeMo/BioNeMo-Inference-Runtime){:target="_blank"}
- ipSAE 메트릭: [bioRxiv](https://www.biorxiv.org/content/10.1101/2025.02.10.637595v2){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/RflgrtzU3Cw){:target="_blank"}
