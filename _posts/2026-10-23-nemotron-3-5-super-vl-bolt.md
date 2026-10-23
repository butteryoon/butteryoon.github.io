---
layout: post
comments: true
title: "Nemotron 3.5 Super VL·Bolt 공개 — 데이터센터부터 엣지까지 아우르는 개방형 에이전트 모델군"
description: "NVIDIA가 120B 파라미터 멀티모달 Super VL과 엣지용 74B/4B Bolt 모델을 오픈 가중치로 공개했다. Hugging Face·Ollama·vLLM에서 바로 내려받아 로컬·클라우드 어디서든 에이전트 워크로드에 쓸 수 있다."
img: nemotron35_super_vl_bolt_title.webp
date: 2026-10-23 19:30:00 +0900
last_modified_at: 2026-10-23 19:30:00 +0900
tags: [nvidia, nemotron, llm-model, llm, open-model, agent, multimodal]
related: llm-model
categories: [nvidia, llm-model]
source_url: https://blogs.nvidia.co.kr/blog/nemotron-3-5-super-vl/
source_date: 2026-10-22
---
NVIDIA 한국 블로그에 10월 22일 올라온 [「데이터센터부터 데스크톱까지 개방되다: 최고 효율의 Super VL 및 Bolt 모델로 Nemotron 제품군을 넓히는 NVIDIA」](https://blogs.nvidia.co.kr/blog/nemotron-3-5-super-vl/)를 읽고 정리했다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** NVIDIA가 Nemotron 3.5 제품군에 120B 파라미터 멀티모달 **Super VL**과 엣지·로컬용 **Bolt 74B-A9B(24GB)**, **Bolt E4B(4GB)**를 추가로 공개했다. 세 모델 모두 **오픈 가중치**로 Hugging Face에서 즉시 내려받을 수 있고, llama.cpp·Ollama·LM Studio·Unsloth·vLLM을 지원한다. Super VL은 프런티어급 지능을 1/5 크기로 구현해 대량 에이전트 워크플로우에, Bolt 계열은 RTX/DGX Spark·Jetson 같은 온디바이스 환경에 최적화됐다. NeMo·NIM·공개 데이터셋으로 포스트트레이닝·커스터마이징 파이프라인도 함께 제공돼, 조직이 자체 데이터·보안 요건에 맞춰 모델을 직접 손보고 배포할 수 있다.

## 원문 정보
- **원문 제목**: 데이터센터부터 데스크톱까지 개방되다: 최고 효율의 Super VL 및 Bolt 모델로 Nemotron 제품군을 넓히는 NVIDIA
- **원문 링크**: https://blogs.nvidia.co.kr/blog/nemotron-3-5-super-vl/
- **발행일**: 2026-10-22 (KST)
- **작성자**: NVIDIA 한국 블로그팀
- **핵심 링크**:
  - Super VL 허깅페이스: https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-NVFP4
  - Bolt 74B-A9B 24GB 허깅페이스: https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Bolt-74B-A9B-24GB-GGUF
  - Bolt 74B-A9B vLLM용: https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Bolt-74B-A9B-24GB
  - 기술 블로그(상세 아키텍처): https://developer.nvidia.com/blog/nvidia-nemotron-3-5-super-vl-powers-specialized-agents-with-multimodal-reasoning/
  - NIM 마이크로서비스: https://www.nvidia.com/ko-kr/ai-data-science/products/nim-microservices/
  - 공개 데이터셋 컬렉션: https://huggingface.co/collections/nvidia/nemotron-pre-training-datasets

## 한 줄 요약
NVIDIA가 멀티모달 에이전트 작업에 최적화된 120B Super VL과 온디바이스 배포용 74B/4B Bolt 모델을 **오픈 가중치로 즉시 사용 가능하게 공개**했으며, NeMo·NIM·공개 데이터셋으로 커스터마이징·운영 파이프라인까지 함께 제공한다.

## 핵심 기술 내용

### Nemotron 3.5 Super VL — 프런티어급 지능을 1/5 크기로 압축
- **파라미터**: 120B (NVFP4 양자화), NVIDIA 최대 리즈닝 모델의 약 1/5 크기
- **아키텍처**: 멀티모달(비전-언어) — 이미지·웹페이지·PDF·표·차트·OCR 이해 후 추론·행동까지 단일 모델에서 처리
- **학습**: 약 26조 토큰 사전학습 + 약 380만 샘플·수십 환경 강화학습 포스트트레이닝
- **벤치마크(원문 차트 기준)**:
  - Artificial Analysis Intelligence Index: 동급 최고 수준 프런티어급 지능
  - 처리량: 동급 오픈 모델 대비 **최대 5배 높음**(그다음 빠른 모델의 5배)
  - PinchBench: 동급 개방형 모델 1위
  - BrowseComp: 개방형 모델 2위

<details class="evidence"><summary>원문 근거</summary><blockquote>"Nemotron 3.5 Super VL은 대량 에이전틱 워크플로우를 위해 동급 최고 수준의 프런티어급 지능을 제공합니다. 동급에서 가장 빠른 오픈 에이전틱 모델로 그다음으로 빠른 모델보다 5배 빠른데요."</blockquote></details>

### Nemotron 3.5 Bolt — 엣지·로컬 배포 최적화
| 모델 | 파라미터 | 양자화 | 타깃 메모리 | 타깃 하드웨어 | 공개일 |
|------|----------|--------|-------------|---------------|--------|
| Bolt 74B-A9B 24GB | 74B (A9B MoE) | 네이티브 2비트 | 24GB | RTX/DGX Spark, DGX Station, RTX PRO, GeForce RTX PC | 즉시 (10/16 출시) |
| Bolt E4B 4GB | 원문 미공개 | - | 4GB | GeForce RTX, Jetson, RTX/DGX Spark | 2026-10-23 |

- Bolt 74B-A9B: **PinchBench 동급 개방형 1위**, BrowseComp 개방형 2위. 웹 리서치·글쓰기·코드 디버깅 등 로컬 에이전트 하네스 워크로드를 클라우드 없이 처리 가능.
- Bolt E4B: 피지컬·대화형 에이전트(음성 어시스턴트, 인터랙티브 캐릭터, 임베디드 인터페이스)용.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Super VL과 Bolt 74B-A9B 24GB는 지금 바로 쓸 수 있고, Bolt E4B는 10월 23일 공개됩니다. 이들 세 가지 오픈 모델은 대규모 엔터프라이즈 서버부터 엣지 기기와 로컬 하드웨어까지 에이전트 배포의 전 영역을 아우릅니다."</blockquote></details>

### 커스터마이징·운영 파이프라인 (NeMo·NIM·공개 자산)
- **NeMo**: Python 기반 파이프라인으로 포스트트레이닝 지원. 공개 데이터셋(금융, 지시 수행, 롱 컨텍스트, 공정성, 비전-언어 등)과 NeMo Gym RL 환경 제공.
- **NIM**: 프로덕션 즉시 사용 마이크로서비스로 패키징. 검증된 최적화 구성 + 엔터프라이즈 지원 + 2주 주기 보안 패치·성능 개선.
- **Dynamo**: 추론 서빙 시 지연 시간 **85% 감소**, 동시 처리 사용자 **최대 8배 증가**.
- **공개 자산**: 허깅페이스 컬렉션에 사전학습 데이터셋, 포스트트레이닝 데이터셋, 강화학습 환경 모두 공개.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA NeMo와 Python 기반 파이프라인, 공개 데이터셋이 맞춤 구성을 뒷받침합니다. NVIDIA Dynamo에서 이 모델을 구동하면 지연 시간을 85% 낮추면서 최대 8배 많은 사용자에게 서비스할 수 있습니다."</blockquote></details>

### 실전 포스트트레이닝 사례 (원문 나열 기업)
- **Scale AI**: Skydance 공급업체 여신 금융 에이전트, EY M&A 실사 에이전트
- **AfterQuery**: Hebbia 신용 계약 분석
- **AI21, Alice**: 사이버 공격 대응력 강화
- **Mercor**: APEX-SWE 벤치마크로 코딩 에이전트(계획·작성·디버깅) 포스트트레이닝
- **ModelScope/ms-swift**: 특화 멀티모달 애플리케이션 파인튜닝 프레임워크 제공
- **Ramp, Dream, Lila Sciences, R1, TrainLoop/Nolla** 등 추가 사례
- **플랫폼 통합**: Atlassian(Jira/Confluence 에이전틱 검색), CrowdStrike(블루팀 운영), Dassault Systèmes(설계·시뮬레이션), Perplexity(검색·Computer), Salesforce(Missionforce Operations), Synopsys(반도체 설계 검증)

## 심화 분석

### Super VL — "크기 대비 지능"을 내세운다

원문은 Super VL을 **NVIDIA 최대 리즈닝 모델의 약 1/5 크기로 그에 견줄 만한 프런티어급 지능**이라고 표현한다. 속도 쪽 주장은 두 가지다 — 동급에서 **가장 빠른 오픈 에이전틱 모델**이며 그다음으로 빠른 모델보다 5배 빠르고, 차트 설명에서는 동급 대비 **최대 5배 높은 처리량**을 든다.

다만 **원문은 Artificial Analysis Intelligence Index 차트를 싣기만 하고 점수를 본문에 적지 않았다.** 어떤 모델과 몇 점 차이인지는 이 글에서 확인할 수 없다. 숫자로 비교하려면 지표 사이트나 영어 기술 블로그를 따로 봐야 한다.

### Bolt 계열이 노리는 "로컬 에이전트" 자리

Bolt 74B-A9B 24GB는 **네이티브 2비트 양자화**로 24GB 메모리에 올라간다. 원문이 든 타깃은 GeForce RTX PC, DGX Spark, DGX Station, RTX PRO 워크스테이션, 그리고 10월 16일 나온 RTX Spark 노트북이다.

의미는 분명하다 — **클라우드 API 없이 로컬에서 에이전트 워크플로를 끝내는** 쪽을 겨냥한다. 프라이버시·지연 시간·비용 때문에 로컬을 원하던 수요다. llama.cpp·Ollama·LM Studio·Unsloth·vLLM 지원이 공개 시점부터 갖춰진 것도 진입 장벽을 낮춘다. 다만 **원문에 특정 지포스 모델명이나 VRAM 조합은 나오지 않으므로**, 손에 있는 장비에서 실제로 돌지는 직접 확인해야 한다.

### 커스터마이징 파이프라인이 닫힌 모델과의 차별점
가중치가 닫힌 모델은 공급자의 정책과 데이터 소재지 제약을 따라야 한다. Nemotron 3.5 제품군은 **개방형 가중치·공개 데이터셋·학습 레시피·NeMo 파이프라인·NIM 패키징**을 모두 제공해 조직이 **자체 인프라·자체 데이터·자체 보안 정책** 아래 모델을 직접 손보고 배포할 수 있게 한다. 원문 표현대로 "통제권은 두 방향으로 작동한다": 배포 위치 자유 + 모델 직접 수정 자유.

### 우리 개발 환경에서의 적용 포인트
- **로컬 개발·테스트**: 24GB 메모리를 확보할 수 있는 장비(원문이 든 예: GeForce RTX PC, DGX Spark, RTX PRO 워크스테이션)라면 Bolt 74B-A9B 24GB를 Ollama·LM Studio로 띄워 에이전트 프롬프트·툴체인을 검증해 볼 수 있다. 특정 지포스 모델에서의 동작은 원문에 없으니 직접 확인해야 한다.
- **프로덕션 서빙**: NIM으로 Super VL·Bolt를 컨테이너화해 쿠버네티스·Dynamo 위에 올리면 2주 주기 패치·엔터프라이즈 지원이 따라옴.
- **도메인 특화**: 금융·법률·의료·사이버보안 등 자체 기밀 데이터로 포스트트레이닝해 전문 에이전트 구축 가능(원문 사례 참고).
- **멀티모달 파이프라인**: Super VL 단일 모델로 OCR·표/차트 이해·문서 QA·웹페이지 추출을 한 번에 처리 → 별도 비전 모델·텍스트 모델 파이프라인 단순화.

## 참고 자료
- **원문(한국어)**: https://blogs.nvidia.co.kr/blog/nemotron-3-5-super-vl/
- **기술 블로그(영어·상세 아키텍처)**: https://developer.nvidia.com/blog/nvidia-nemotron-3-5-super-vl-powers-specialized-agents-with-multimodal-reasoning/
- **허깅페이스 모델 카드(Super VL)**: https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-NVFP4
- **허깅페이스 모델 카드(Bolt 74B-A9B GGUF)**: https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Bolt-74B-A9B-24GB-GGUF
- **사전학습 데이터셋 컬렉션**: https://huggingface.co/collections/nvidia/nemotron-pre-training-datasets
- **NIM 마이크로서비스 제품 페이지**: https://www.nvidia.com/ko-kr/ai-data-science/products/nim-microservices/
- **Dynamo 추론 서빙**: https://github.com/ai-dynamo/dynamo
- **관련 기발행 글**: [NVIDIA Blackwell AgentPerf 벤치마크]({{site.baseurl}}/dev/2026/08/29/nvidia_vera_rubin_blackwell_agentx_perf_per_watt.html), [RTX 로컬 LLM 가이드]({{site.baseurl}}/dev/2026/09/09/rtx-local-llm-guide.html)