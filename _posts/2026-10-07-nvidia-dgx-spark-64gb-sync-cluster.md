---
layout: post
comments: true
title: "DGX Spark 64GB — Sync Cluster로 두 대를 묶는 로컬 AI 확장"
description: "DGX Spark에 64GB 구성이 추가됐다. 128GB 모델과 같은 칩·OS·소프트웨어 스택에 4,999달러부터 공급되고, ConnectX-7 + Sync Cluster Assistant로 두 대를 묶으면 메모리 128GB, 성능 최대 1.7배."
img: nvidia_dgx_spark_64gb_title.webp
date: 2026-10-07 19:00:00 +0900
last_modified_at: 2026-10-08 23:10:39 +0900
tags: [nvidia, dgx-spark, local-ai, llm-serving, sync-cluster-assistant, connectx-7, hermes-agent]
related: local-ai
categories: [nvidia-analysis, llm-serving]
source_url: https://blogs.nvidia.co.kr/blog/local-ai-dgx-spark-64gb-sync/
source_date: 2026-10-06
---

NVIDIA DGX Spark에 64GB 통합 메모리 구성이 추가된다. 10월 23일부터 Acer·ASUS·Dell·Gigabyte·HP·MSI를 통해 4,999달러부터 공급된다. 128GB 모델과 같은 GB10 Grace Blackwell 슈퍼칩·DGX OS·NVIDIA AI 소프트웨어 스택을 그대로 두고 가격 문턱만 낮췄다. 워크로드가 커지면 NVIDIA Sync Cluster Assistant로 두 대를 한 클러스터로 묶으면 된다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** DGX Spark 64GB가 10월 23일부터 공급된다. 한 대로 최대 1,000억 파라미터 모델을 기기 안에서 돌릴 수 있다. ConnectX-7 포트끼리 QSFP 케이블로 두 대를 연결해 Sync Cluster Assistant를 돌리면 메모리가 128GB로 모여 최대 2,000억 파라미터까지 올라간다. NVIDIA 자체 테스트(Qwen 3.8 27B)에서 두 대 구성은 한 대보다 최대 1.7배 빨랐다. 이달 말 나오는 NVIDIA Sync Model Launcher는 Qwen3.8 27B 다운로드·실행과 OpenCode 연결을 버튼 몇 번으로 줄인다.

## 1. 64GB 구성: 128GB 모델과 같은 스택, 낮아진 가격

DGX Spark는 Grace Blackwell 컴퓨팅, 통합 메모리, ConnectX-7 네트워킹, CUDA 가속 소프트웨어 스택을 한 시스템에 담은 개인용 AI 슈퍼컴퓨터다. 새 64GB 구성도 GB10 슈퍼칩과 DGX OS, NVIDIA AI 소프트웨어 스택 전체는 128GB 모델과 똑같다. 달라진 건 메모리 용량과 가격이다.

- 최대 **1,000억 파라미터** 모델과 그 위의 에이전틱 애플리케이션을 기기 안에서 구동
- 제조 파트너(Acer·ASUS·Dell·Gigabyte·HP·MSI)를 통해서만 공급
- **4,999달러**부터, 10월 23일(금) 공급 시작
- DGX OS와 NVIDIA AI 스택을 설치한 상태로 출하

<details class="evidence"><summary>원문 근거</summary><blockquote>"제조 파트너를 통해서만 공급되는 새 64GB 구성은 GB10 Grace Blackwell 슈퍼칩과 DGX OS, 전체 NVIDIA AI 소프트웨어 스택을 128GB 모델과 똑같이 유지하면서 가격 문턱을 낮췄습니다. 최대 1,000억 파라미터 모델과 그 위에 올린 에이전틱 애플리케이션을 온전히 기기 안에서 구동할 수 있죠."</blockquote></details>

## 2. Sync Cluster Assistant로 두 대 확장

DGX Spark는 모두 ConnectX-7 NIC를 내장하고 출하된다. 두 대를 QSFP 케이블로 직접 연결하면 이렇게 된다.

- 메모리가 **128GB**로 모여 최대 **2,000억 파라미터** 모델까지 지원
- 메모리 대역폭 **두 배**, 성능 **최대 1.7배**
- NVIDIA Sync 앱의 클러스터 어시스턴트가 연결된 장치를 찾아 구성을 검증하고 ConnectX-7 네트워크까지 설정
- 모든 노드가 같은 소프트웨어 스택으로 돌아가니 한 대에서 두 대로 늘릴 때 다시 설정할 게 없다

1.7배라는 수치는 NVIDIA가 **Qwen 3.8 27B**로 직접 돌린 테스트에서 나왔다. 64GB 두 대를 클러스터로 묶었을 때 한 대 대비 최대치다. 독립 벤치마크로 확인된 값은 아니라는 점을 감안해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA가 진행한 Qwen 3.8 27B 테스트에서 64GB 시스템 두 대를 클러스터로 구성하자 한 대일 때보다 최대 1.7배 높은 성능이 나왔고, 워크로드가 요구하는 만큼 더 확장할 여지도 남아 있었습니다."</blockquote></details>

<details class="evidence"><summary>원문 근거</summary><blockquote>"여기에 두 대를 QSFP 케이블로 직접 연결하면 메모리를 128GB로 모아 최대 2,000억 파라미터 모델까지 지원하면서, 메모리 대역폭은 두 배, 성능은 최대 1.7배까지 끌어올릴 수 있죠."</blockquote></details>

## 3. 출하 시점 소프트웨어 스택

전원을 켜면 바로 에이전트 개발을 시작할 수 있는 상태로 나온다.

- **NVIDIA Agent Toolkit**, CUDA-X AI 라이브러리, **Nemotron** 오픈 모델
- 기본 지원 런타임: **Ollama, vLLM**, CUDA를 지원하는 **PyTorch**
- 원문의 시작 안내에는 지원 추론 프레임워크로 **llama.cpp, Ollama, vLLM, LM Studio**가 올라 있다 — 이 중 하나를 내려받고 워크플로우에 맞는 권장 로컬 모델을 받는 순서다
- 창작 쪽에서는 Blender가 첫 주요 지원 공급사 중 하나로, 미리 빌드된 설치 파일을 곧 내놓는다

<details class="evidence"><summary>원문 근거</summary><blockquote>"DGX Spark는 첫날부터 에이전트 개발에 바로 쓸 수 있는 상태로 출하됩니다. NVIDIA Agent Toolkit과 CUDA-X AI 라이브러리, Nemotron 오픈 모델은 물론 Ollama와 vLLM, CUDA를 지원하는 PyTorch 같은 인기 런타임까지 기본으로 지원하죠."</blockquote></details>

이달 말에는 **NVIDIA Sync Model Launcher**가 나온다. DGX Spark 한 대나 클러스터에 [**Qwen3.8 27B**](https://huggingface.co/Qwen/Qwen3.8-27B){:target="_blank"}를 내려받아 실행하고 NVIDIA Sync가 연결된 장치 전체에 모델을 구성해 노트북에서도 쓸 수 있게 해 준다. **OpenCode**가 그 모델을 쓰도록 설정까지 해 주므로 브라우저에서 곧바로 코딩을 시작할 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"이달 말에는 NVIDIA Sync Model Launcher가 나와 로컬 AI 실행을 버튼 몇 번 누르는 일로 바꿔 놓습니다. 개발자는 DGX Spark 한 대 또는 클러스터에 Qwen3.8 27B를 내려받아 실행할 수 있고, NVIDIA Sync가 연결된 장치 전반에 걸쳐 모델을 구성해 노트북에서도 접근할 수 있게 해 주죠."</blockquote></details>

## 4. 워크플로우 예시 세 가지

원문은 쓰임새를 세 가지로 소개한다.

- **AI 에이전트를 24시간 돌리기:** 코딩·리서치 에이전트를 계속 띄워 두고 코드 리뷰, 문서 분석, 다단계 작업을 맡긴다. 클러스터로 묶으면 더 큰 모델, 더 긴 컨텍스트 창, 여러 에이전트 동시 실행을 감당할 여유가 생긴다.
- **평소 쓰는 PC에서 AI 앱 구동하기:** 언어 모델·이미지 생성 모델 추론은 DGX Spark가 맡고 에이전트나 창작 애플리케이션은 노트북·데스크톱에서 쓴다. PC는 다른 작업에 쓸 수 있다.
- **작업이 커지면 확장하기:** 64GB 두 대를 200GbE 패브릭으로 이어 메모리를 128GB로 모은다. 한 대에서 돌던 워크플로우를 소프트웨어 환경 재구성 없이 두 대로 넓힌다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"더 큰 모델이나 더 긴 컨텍스트 창, 동시에 들어오는 에이전트 요청처럼 한 대로 감당하기 어려워지면, DGX Spark 64GB 두 대를 200GbE 패브릭으로 NVIDIA Sync Cluster Assistant를 통해 연결해 메모리를 128GB로 모읍니다."</blockquote></details>

## 5. 에이전틱 AI 플레이북

DGX Spark용 에이전틱 AI 플레이북은 build.nvidia.com의 [NemoClaw](https://build.nvidia.com/spark/nemoclaw){:target="_blank"}, [OpenClaw](https://build.nvidia.com/spark/openclaw){:target="_blank"}, [Hermes Agent](https://build.nvidia.com/spark/hermes-agent){:target="_blank"}, [OpenShell](https://build.nvidia.com/spark/openshell){:target="_blank"} 페이지에 있다. 다음 플레이북은 곧 64GB 기기용으로도 나온다.

- vLLM으로 LLM 서빙하기
- 로컬 LLM으로 OpenClaw 실행하기
- 분산 워크로드를 위해 여러 DGX Spark 연결하기

Hermes Agent는 앞서 다룬 [IFA 2026 발표]({{site.baseurl}}/nvidia-analysis/llm-serving/2026/10/04/nvidia-ifa-local-ai-rtx-spark.html)에서 RTX PC 원클릭 로컬 모델 설정 사례로 등장했는데 이번에는 DGX Spark 플레이북 목록에 올랐다.

## 6. 시사점

- **같은 스택, 다른 진입 가격:** 64GB 구성은 칩·OS·소프트웨어를 128GB 모델과 공유한다. 한 대로 시작한 코드와 환경을 나중에 두 대 클러스터로 옮겨도 바꿀 게 없다는 게 NVIDIA의 설명이다.
- **확장의 축은 메모리다:** 두 대를 묶어 얻는 건 메모리 128GB, 대역폭 2배, 성능 최대 1.7배다. 2,000억 파라미터급 모델이나 긴 컨텍스트를 다룰 때 의미가 있다.
- **멀티 노드 설정 부담:** Sync Cluster Assistant가 장치 감지·검증·네트워크 설정을 자동으로 처리한다. 두 대 클러스터를 꾸리는 일이 케이블 연결과 앱 실행 수준으로 내려온다.
- **런처:** Sync Model Launcher와 OpenCode 연동으로 "모델 받기 → 코딩 에이전트 연결"까지가 버튼 몇 번이 된다. 로컬 코딩 에이전트를 띄우는 절차가 짧아진다.

다만 성능 수치는 NVIDIA 발표 기준이다. "1,000억 파라미터 구동"도 최대치라 실제로는 런타임과 양자화 방식에 따라 달라진다. 128GB 모델과의 가격 차이는 원문에 나오지 않는다.

## 7. 참고 자료

- 원문: [NVIDIA DGX Spark 64GB, 로컬 AI를 구축하고 확장하는 더 많은 길을 열다](https://blogs.nvidia.co.kr/blog/local-ai-dgx-spark-64gb-sync/){:target="_blank"} (NVIDIA 블로그 코리아, 2026-10-06)
- DGX Spark 제품 페이지: [nvidia.com/ko-kr/products/workstations/dgx-spark/](https://www.nvidia.com/ko-kr/products/workstations/dgx-spark/){:target="_blank"}
- NVIDIA Sync 문서: [docs.nvidia.com/sync/latest/index.html](https://docs.nvidia.com/sync/latest/index.html){:target="_blank"}
- 에이전틱 AI 플레이북: [build.nvidia.com/spark](https://build.nvidia.com/spark){:target="_blank"}
- 관련 영상: [How to Connect Two DGX Sparks with NVIDIA Sync](https://www.youtube.com/watch?v=MehBUQtb9qM){:target="_blank"} (NVIDIA Developer)
- 이전 글: [NVIDIA, IFA 2026서 로컬 AI 가속하는 신기술·생태계 공개]({{site.baseurl}}/nvidia-analysis/llm-serving/2026/10/04/nvidia-ifa-local-ai-rtx-spark.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/_MauPmUJJ08){:target="_blank"}
