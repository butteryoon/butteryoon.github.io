---
layout: post
comments: true
title: "NVIDIA BioNeMo Agent Toolkit — GPU 워크플로우를 에이전트의 도구로 넘기다"
description: "Claude Science와 통합된 NVIDIA BioNeMo Agent Toolkit이 Parabricks·RAPIDS-singlecell·nvMolKit 같은 가속 도구를 에이전트 스킬로 묶은 방식을 정리한다."
img: bionemo-agent-toolkit_title.webp
date: 2026-09-15 20:10:00 +0900
last_modified_at: 2026-09-15 20:10:00 +0900
tags: [nvidia, bionemo, claude-science, ai-agents, drug-discovery, gpu, nim, llm]
related: llm
categories: dev
---
NVIDIA 블로그의 [「BioNeMo Agent Toolkit, Claude Science에서 생명과학 연구자들에게 가속화된 AI를 제공하다」](https://blogs.nvidia.co.kr/blog/claude-science-bionemo-agent-toolkit/){:target="_blank"}를 읽고 정리했다. 요점은 도구의 목록이 아니라 도구의 급이 달라졌다는 데 있다. 지금까지 LLM 에이전트가 손에 쥔 건 웹 검색이나 간단한 API 호출 정도였는데, 여기서는 유전체 정렬과 분자 시뮬레이션 같은 GPU 워크플로우가 통째로 에이전트의 호출 대상이 된다. 앞서 다룬 [OpenAI의 1만 에이전트 나비에-스토크스 글]({{site.baseurl}}/dev/2026/09/11/openai-navier-stokes.html)이 규모로 과학을 밀어붙인 사례라면, 이쪽은 계산 자원을 에이전트에 물리는 배관 작업에 가깝다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** NVIDIA가 앤스로픽의 과학 연구용 AI 워크벤치 Claude Science에 BioNeMo Agent Toolkit을 통합했다. 연구자가 자연어로 요청하면 에이전트가 Parabricks·RAPIDS-singlecell·nvMolKit 같은 가속 도구를 스킬로 골라 GPU에서 실행한다. RAPIDS-singlecell은 130만 개 세포 전처리·클러스터링을 52분에서 25초로 줄였다.

## 무엇이 붙었나

Claude Science는 과학 연구를 위한 AI 워크벤치다. 연구자가 자연어로 에이전트와 대화하면서 연구 전 과정을 처음부터 끝까지 끌고 갈 수 있는 환경을 목표로 한다. 여기에 NVIDIA가 붙인 게 BioNeMo Agent Toolkit으로, 가속 라이브러리와 NIM 마이크로서비스를 에이전트가 고를 수 있는 스킬 형태로 포장한 것이다.

흐름은 단순하다. 자연어 요청이 들어오면 에이전트가 계획을 세우고, 필요한 가속 도구를 골라 실행하고, 결과를 해석해 다음 질문을 다듬는다. 여기서 실행 단계가 GPU 위에서 돌아간다는 점이 차이를 만든다.

## 스킬로 묶인 가속 도구들

- **NVIDIA Parabricks** — 유전체 분석 시간을 수 시간에서 수 분으로 줄인다.
- **RAPIDS-singlecell** — 130만 개 세포의 전처리·클러스터링 워크플로우를 52분에서 25초로 압축한다.

    <details class="evidence"><summary>원문 근거</summary><blockquote>"130만 개 세포의 전처리·클러스터링 워크플로우를 52분에서 25초로 압축"</blockquote></details>

- **nvMolKit** — 유사도 검색과 컨포머 생성 같은 화학정보학 연산을 최대 3,000배까지 끌어올린다.

    <details class="evidence"><summary>원문 근거</summary><blockquote>"유사도 검색과 컨포머 생성 등 화학정보학 연산을 최대 3,000배 가속"</blockquote></details>

- **BioNeMo 오픈 모델과 NIM 마이크로서비스** — Evo 2, Boltz-2, OpenFold3 같은 모델을 컨테이너 기반 마이크로서비스로 감싸, 에이전트가 표준 API로 고성능 추론 엔드포인트에 닿게 한다.

## 왜 52분이 25초가 되는 게 중요한가

절대 시간만 보면 52분도 못 기다릴 만큼 긴 시간은 아니다. 중요한 건 그 시간이 루프 안에 들어올 수 있느냐다.

전처리에 52분이 걸리면 그건 배치 작업이다. 조건을 바꿔 다시 돌리려면 사람이 결과를 확인하고 다음 실행을 걸어야 한다. 25초가 되면 이야기가 달라진다. 에이전트가 파라미터를 바꿔가며 수십 번 돌려보고 그중 쓸 만한 것만 들고 오는 게 가능해진다. 전처리가 결과물이 아니라 추론의 한 단계가 되는 셈이다.

nvMolKit의 3,000배도 같은 맥락에서 읽힌다. 화합물 라이브러리를 훑는 작업이 "밤새 돌려놓고 아침에 확인"에서 "물어보고 기다리는" 수준으로 내려오면, 가설을 세우고 검증하는 주기 자체가 달라진다.

## 남는 질문

툴킷 자체는 특정 프레임워크에 묶이지 않는 오픈 소스로 공개돼 다른 연구 플랫폼에도 붙일 수 있다. 다만 블로그 글이 다루는 범위는 여기까지다. 실제로 에이전트가 도구를 얼마나 정확하게 고르는지, 잘못된 파라미터로 GPU 시간을 태우는 경우를 어떻게 걸러내는지는 이 글에 나오지 않는다. 자율적으로 가설을 검증하는 그림이 실제로 성립하려면 결국 그 선택의 정확도가 관건일 텐데, 그 부분은 아직 확인할 자료가 없다.

## 참고

- [NVIDIA BioNeMo Agent Toolkit, Claude Science에서 생명과학 연구자들에게 가속화된 AI를 제공하다](https://blogs.nvidia.co.kr/blog/claude-science-bionemo-agent-toolkit/){:target="_blank"}
- [NVIDIA BioNeMo GitHub](https://github.com/NVIDIA-BioNeMo){:target="_blank"}
- [Claude Science — Anthropic](https://www.anthropic.com/news/claude-science-ai-workbench){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/-qycBqByWIY){:target="_blank"}
