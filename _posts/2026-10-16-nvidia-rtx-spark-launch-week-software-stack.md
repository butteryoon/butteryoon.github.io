---
layout: post
comments: true
title: "RTX Spark 출시 주간의 소프트웨어 스택 — Nemotron 3.5 Bolt 74B, Switchyard, MXC"
description: "NVIDIA가 RTX Spark 노트북 출시(10/16)에 맞춰 정리한 소프트웨어 로드맵 해설. 에이전트용 MoE 모델 Nemotron 3.5 Bolt 74B-A9B(2비트 QAT로 24GB), 로컬·클라우드 하이브리드 라우팅 Switchyard와 PAIR, MXC·OpenShell 샌드박싱, Windows on Arm 개발 도구 현황을 날짜별로 정리한다."
img: nvidia_rtx_spark_software_stack_title.webp
date: 2026-10-16 19:30:00 +0900
last_modified_at: 2026-10-16 19:30:00 +0900
tags: [nvidia, rtx-spark, local-ai, nemotron, moe, quantization, hermes-agent, windows-on-arm, llm, llm-serving]
related: llm-serving
categories: [nvidia-analysis, llm-serving]
source_url: https://blogs.nvidia.co.kr/blog/local-ai-rtx-spark-laptops-software-updates/
source_date: 2026-10-14
---

엔비디아 한국 블로그가 [RTX Spark 노트북 출시 주간의 소프트웨어 로드맵](https://blogs.nvidia.co.kr/blog/local-ai-rtx-spark-laptops-software-updates/){:target="_blank"}(2026-10-14 발행)을 한 편에 모았다. 하드웨어 스펙을 다시 소개하는 글이 아니다. 10월 16일 노트북이 나올 때 어떤 앱과 모델, 도구가 함께 도착하는지가 중심이고 에이전트 앱·모델·보안·개발 도구·게임 소식을 날짜별로 늘어놓는다. 분량은 많지 않지만 로컬 에이전트 스택이 어디로 가는지가 몇 개 키워드에 담겨 있어서 로컬 LLM을 돌리는 개발 환경 기준으로 추려 본다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 일정은 10/14 PAIR 정식 출시 → 10/15 Bolt 74B-A9B 24GB 공개 → 10/16 노트북 출시와 DLSS 5 고속 모델 → 10/23 Bolt E4B 4GB와 RTX Remix 1.6 → 11월 Blender DLSS 순이다. 에이전트용 모델 Nemotron 3.5 Bolt 74B-A9B는 전체 74B, 활성 9B인 MoE에 양자화 인식 학습과 2비트 양자화를 적용해 24GB에 담았고 출시 시점부터 llama.cpp·vLLM·Ollama·Unsloth·LM Studio에서 돌아간다. 눈여겨볼 대목은 "로컬 단독"이 아니다. 로컬과 클라우드를 오가는 하이브리드 라우팅(Switchyard·PAIR·HydraFusion)과 OS 수준 에이전트 샌드박싱(MXC·OpenShell)이 기본 조합으로 굳어 가는 흐름이다.

## 1. 출시 주간 타임라인

원문 곳곳에 흩어진 날짜를 모으면 아래와 같다. 모두 원문에 적힌 날짜다(원문 발행일 10/14의 "오늘"은 10/14로 옮겼다).

- **10/14(수)** — NVIDIA PAIR가 베타를 벗어나 정식 릴리스. DLSS 프레임 생성의 새 퍼포먼스 모드도 이날부터 NVIDIA 앱 베타에서 쓸 수 있다. llama.cpp의 Windows 네이티브 앱 Llama Windows도 같은 날 공개.
- **10/15(목)** — Nemotron 3.5 Bolt 74B-A9B 24GB 공개. RTX Spark의 로컬 에이전트용으로 설계한 LLM이다.
- **10/16(금)** — RTX Spark 노트북 출시. 2배 빨라진 DLSS 5 AI 모델이 NBA 2K27에서 처음 선보인다. RTX Spark용 [AI 플레이북](https://build.nvidia.com/rtx-spark){:target="_blank"}도 이날부터 제공된다.
- **10/23(금)** — 게임 캐릭터용 SLM Nemotron 3.5 Bolt E4B 4GB 공개. Skyrim용 AI NPC 모딩 플랫폼 SkyrimNET에서 바로 써 볼 수 있다. RTX Remix 1.6(ComfyUI 버전 2 포함)과 Remix의 DLSS 5 지원도 같은 날이다.
- **11월** — Blender 5.3에 DLSS 레이 리컨스트럭션 지원. DLSS 5 고속 모델은 11/5 Hogwarts Legacy, 11월 말 Where Winds Meet로 넓어진다.

## 2. Nemotron 3.5 Bolt 74B-A9B — 2비트 QAT로 24GB에 담은 에이전트 MoE

기술적으로 가장 무게가 실린 대목이다. Bolt 74B는 전체 파라미터 74B, 활성 파라미터 9B인 MoE 모델이다. 품질을 지키는 양자화 인식 학습(QAT)과 2비트 양자화로 24GB 안에 담았다. 원문은 "속도와 지능 사이의 최적점"을 노렸다며 널리 쓰이는 대부분의 밀집 모델보다 빠르고 비슷한 메모리를 쓰는 대부분의 MoE 모델보다 똑똑하다고 주장한다. 다만 이를 뒷받침할 벤치마크 수치는 내놓지 않았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"이 모델은 74B 파라미터에 활성 파라미터가 9B인 MoE(전문가 혼합) 모델로, 품질을 지켜 주는 고급 양자화 인식 학습 기법과 2비트 양자화를 적용해 24GB 안에 담아냈습니다."</blockquote></details>

실행 엔진으로는 출시 시점부터 llama.cpp, vLLM, Ollama, Unsloth, LM Studio가 꼽혔고 Hermes와 OpenClaw 같은 에이전트에서도 바로 쓸 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"사용자는 출시 시점부터 llama.cpp와 vLLM, Ollama, Unsloth, LM Studio에서 Nemotron 3.5 Bolt 74B를 구동할 수 있습니다. Hermes와 OpenClaw 같은 에이전트에서도 바로 쓸 수 있고요."</blockquote></details>

학습 데이터와 가중치, 레시피까지 공개한 완전한 오픈 소스라는 점은 다른 Nemotron 모델과 같다.

RTX Spark는 통합 메모리가 최대 128GB이고 AI 연산 성능이 1페타플롭이다. 여기서 활성 9B짜리 MoE를 돌리는 구도이니 [9월에 정리한 RTX 로컬 LLM 입문 스택]({{site.baseurl}}/dev/2026/09/09/rtx-local-llm-guide.html)보다 한 단계 큰, "노트북 한 대에 에이전트 전용 대형 MoE"를 올리는 경우가 생겼다.

같은 계열의 소형 모델 Bolt E4B 4GB는 인게임 AI 에이전트용 기술 모음 NVIDIA ACE의 3세대 SLM이다. 자연스러운 대화에 멀티턴 도구 사용, 지시 수행, 애플리케이션 상태 기반 그라운딩을 묶었다. 적은 메모리와 빠른 첫 토큰 생성 시간을 노리고 설계했고 새 대화 모드는 응답을 짧고 음성에 맞게 다듬는다. 리즈닝은 개발자가 필요에 따라 켜고 끌 수 있다.

## 3. 로컬·클라우드 하이브리드 — Switchyard, PAIR, HydraFusion

원문 전체에 깔린 주제는 로컬 단독 실행이 아니다. 마음껏 반복 실험할 수 있는 로컬 모델과 프런티어급 클라우드 모델을 함께 쓰는 라우팅이다.

- **NVIDIA Switchyard** — 오픈 소스 모델 라우터 라이브러리. 에이전트가 질의를 로컬 LLM과 클라우드 LLM에 나눠 보내 클라우드 사용량을 줄인다. 원문에 따르면 Hermes는 10월 중 Switchyard를 붙일 예정이고 Perplexity Computer와 GitHub Copilot은 자체 라우팅 솔루션을 이미 통합했다.
- **NVIDIA PAIR** — 로컬 네트워크에 연결된 호환 PC들의 AI 연산 자원을 모아 쓰는 무료 오픈 소스 도구. 정식 릴리스에서 llama.cpp 지원, Hugging Face 모델 직접 다운로드, 추론 엔진 설정 커스터마이징, RTX Spark 지원이 추가됐다. Windows·Linux·macOS에서 동작한다. 처음 공개되고 한 달 만에 커뮤니티가 기능과 수정 사항을 10건 넘게 보탰다. 원문에 실린 데모 영상 제목이 "NVIDIA PAIR: Hermes 5-Subagent Demo"인 걸 보면 서브에이전트 워크로드를 여러 기기에 나눠 돌리는 시나리오를 염두에 둔 기능이다.
- **GitHub Copilot HydraFusion** — GitHub Copilot의 자체 모델 라우터다. 여기에 로컬 모델 지원을 더해 로컬 하드웨어로 토큰 소비를 줄이면서도 프런티어급 지능을 끌어다 쓰게 하겠다는 방향이다. Microsoft는 MAI 1.1 Coding Flash 같은 코딩 모델을 로컬에서 쓸 수 있게 하고, Windows ML에 llama.cpp를 백엔드로 추가한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"오픈 소스 모델 라우터 라이브러리인 NVIDIA Switchyard는 에이전트가 질의를 로컬 LLM과 클라우드 LLM 사이로 똑똑하게 분배하도록 해, 클라우드 사용량은 최소화하면서도 클라우드에서만 돌릴 때와 비슷한 결과를 얻게 해 줍니다."</blockquote></details>

묶어 보면 라우터 계층(Switchyard·HydraFusion)과 실행 계층(PAIR·llama.cpp·Windows ML)이 갈라지는 모양새다. "로컬 모델 + 클라우드 폴백"이 앱마다 알아서 하던 임시방편에서 플랫폼 기능으로 옮겨 가고 있다.

## 4. 에이전트 보안 — MXC와 OpenShell

에이전트가 파일 시스템과 셸을 직접 만지게 되면서 샌드박싱이 두 계층으로 나뉘었다. 운영체제 수준 보안은 MXC(Microsoft Execution Containers)가 맡는다. NVIDIA OpenShell은 에이전트가 MXC를 쉽게 도입하도록 돕고 자격 증명 보호와 보안 모니터링을 얹는다. 원문 기준으로 OpenClaw와 GitHub Copilot은 이미 MXC를 지원하고 Hermes Agent는 이달 안에 세분화된 OpenShell 제어와 MXC를 통합할 예정이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"MXC가 운영체제 수준의 보안을 제공하고, OpenShell은 에이전트가 MXC를 더 쉽게 도입하도록 도우면서 자격 증명 보호와 보안 모니터링 같은 고급 기능을 더해 주죠. Hermes Agent는 이달 안에 세분화된 OpenShell 제어와 MXC를 통합할 예정이며, OpenClaw와 GitHub Copilot은 이미 MXC를 지원합니다."</blockquote></details>

설치 과정도 손봤다. RTX Spark를 처음 켜면 에이전트 앱을 고르고 설치하는 '시작하기' 화면이 뜬다. GitHub Copilot, Hermes Agent, OpenClaw, Perplexity, Pi, WorkBuddy는 RTX Spark에 맞는 로컬 모델과 설정값을 추천하고 몇 번의 클릭으로 구성까지 끝낸다. NVIDIA는 NVIDIA 앱·Broadcast·PAIR·Omniverse·RTX Remix용 MCP와 [CUDA-X Skills](https://github.com/NVIDIA/skills){:target="_blank"} 라이브러리도 공개했다. 에이전트가 NVIDIA 도구를 직접 다룰 수 있는 접점이 그만큼 늘었다.

## 5. 개발 도구와 Windows on Arm

RTX Spark는 Windows on Arm 기반이라 개발 도구가 얼마나 준비됐느냐가 관건이었다. 원문이 꼽은 커뮤니티 성과는 다음과 같다.

- Windows on Arm용 최초의 PyTorch-CUDA 빌드
- Windows on Arm에서 네이티브로 동작하는 최초의 Python 빌드(Python 3.15)
- Conda 패키지 2,000종 이상을 갖춘 Windows on Arm 최초의 네이티브 Conda 인스톨러
- 상위 1만 5,000개 Python 휠 중 97% 준비 완료(남은 휠도 커뮤니티와 함께 준비할 계획)

작업 환경은 두 갈래다. 주요 개발 도구를 한꺼번에 설치해 주는 Windows 개발자 구성(Windows Developer Config)으로 Windows에서 바로 작업하거나 클라우드 배포 때와 같은 패키지를 쓸 수 있는 WSL을 쓰면 된다.

llama.cpp는 디시전 모델(decision model) 지원을 추가했다. 원문은 긴 리즈닝 루프 없이 요청 분류나 다음 행동 선택 같은 판단을 기존 에이전트보다 최대 200배 빠르게 처리한다고 주장한다. llama.cpp는 Windows 네이티브 동반 앱 [Llama Windows](https://github.com/ggml-org/Llama-Windows){:target="_blank"}도 내놓았다. [Hugging Face 블로그 안내](https://huggingface.co/blog/huggingface/llama-windows){:target="_blank"}대로 Windows 11 시스템 트레이에서 실행되고 로컬 모델과 Hugging Face Hub 추천 모델을 클릭 한 번으로 불러온다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"요청을 분류하는 일부터 다음 행동을 고르는 일까지, 기존 에이전트보다 최대 200배 빠르게 처리할 수 있습니다. 디시전 모델은 이제 RTX Spark와 GeForce RTX GPU의 llama.cpp, Unsloth에서 지원됩니다."</blockquote></details>

## 6. 크리에이티브·게이밍 쪽에서 챙길 것

에이전트 주제와 바로 겹치지는 않지만 원문의 절반을 차지하는 부분이라 요점만 남긴다.

- **Adobe** — Photoshop·Premiere·Substance 3D를 RTX Spark에 맞춰 최적화해 AI와 편집, 색 보정, 효과를 최대 2배 빠르게 처리한다(원문 주장). Photoshop·Premiere의 AI 기능 최적화는 베타에서 이미 쓸 수 있다.
- **Broadcast 2.3** — RTX Spark에서 네이티브로 돈다. 비디오 슈퍼 레졸루션·프레임 생성·오프라인 파일 처리에 더해 에이전트가 Broadcast 효과를 직접 조정하는 MCP 서버와 스킬이 들어갔다.
- **ComfyUI** — RTX 비디오 프레임 생성과 HDR 노드, EXR 내보내기를 지원한다. LTX 2.5와 엮어 1080p 30fps SDR 클립을 60fps HDR로 바꾸는 워크플로가 예시로 나온다.
- **DLSS 5** — 3D 가이드 뉴럴 렌더링으로 조명과 재질을 끌어올린다. 2배 빨라진 AI 모델은 프레임 레이트를 최대 1.4배 높인다(원문 수치). RTX Remix에는 10/23 적용된다.

## 7. 개발 환경 관점의 메모

읽고 나면 세 가지가 남는다.

첫째, 이번 글의 알맹이는 노트북 발표가 아니라 **에이전트 소프트웨어 스택의 표준화**다. 모델(Bolt 74B/E4B), 라우터(Switchyard·PAIR), 보안(MXC·OpenShell), 도구 접점(MCP·CUDA-X Skills)이 한 묶음으로 움직인다. 로컬 에이전트를 직접 돌리는 입장이라면 어느 한 조각보다 이 조합 전체를 지켜봐야 한다.

둘째, 하이브리드 라우팅에 기준점이 생겼다. 모든 질의를 로컬에서 처리하는 대신 라우터가 판단해 로컬과 클라우드로 나누는 구도가 Copilot·Perplexity·Hermes에 똑같이 들어온다. [DGX Spark 두 대를 Sync Cluster로 묶어 메모리를 늘리는 접근]({{site.baseurl}}/nvidia-analysis/llm-serving/2026/10/07/nvidia-dgx-spark-64gb-sync-cluster.html)과 이번 소프트웨어 라우팅은 같은 문제를 반대쪽에서 푼다. 하드웨어로 늘릴 것이냐, 라우팅으로 아낄 것이냐의 차이다.

셋째, 따져 볼 부분이 남아 있다. Bolt 74B의 품질 주장과 디시전 모델의 "최대 200배"는 원문이 수치 근거를 대지 않은 벤더 주장이다. llama.cpp나 Ollama에서 직접 돌려 보기 전에는 판단을 미루는 게 맞다.

로컬 LLM 다운로드가 이미 지난해 전체의 5배에 이른다는 통계와, Qwen 3.8-Flash-Next가 몇 달 전 나온 프런티어 클라우드 모델을 따라잡았다는 서술도 원문의 주장일 뿐 출처 링크가 없다. 흐름을 읽는 참고로만 쓰는 편이 안전하다.

## 참고 자료

- [NVIDIA Korea, 「모든 준비 완료: 전 세계 PC 앱 생태계, NVIDIA RTX Spark를 맞이하다」 (2026-10-14)](https://blogs.nvidia.co.kr/blog/local-ai-rtx-spark-laptops-software-updates/){:target="_blank"}
- [RTX Spark용 AI 플레이북](https://build.nvidia.com/rtx-spark){:target="_blank"}
- [NVIDIA PAIR (GitHub, Personal-AI-Router)](https://github.com/NVIDIA/Personal-AI-Router){:target="_blank"}
- [NVIDIA Skills (GitHub)](https://github.com/NVIDIA/skills){:target="_blank"}
- [Llama Windows (GitHub)](https://github.com/ggml-org/Llama-Windows){:target="_blank"}
- [Hugging Face Blog — Llama Windows 소개](https://huggingface.co/blog/huggingface/llama-windows){:target="_blank"}
- [GitHub Blog — Project HydraFusion](https://github.blog/ai-and-ml/github-copilot/project-hydrafusion-frontier-quality-via-multi-model-orchestration/){:target="_blank"}
- [RTX PC에서 로컬 LLM 시작하기 — Ollama·LM Studio·AnythingLLM]({{site.baseurl}}/dev/2026/09/09/rtx-local-llm-guide.html)
- [DGX Spark 64GB — Sync Cluster로 두 대를 묶는 로컬 AI 확장]({{site.baseurl}}/nvidia-analysis/llm-serving/2026/10/07/nvidia-dgx-spark-64gb-sync-cluster.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/w69Z8K-HGQU){:target="_blank"}
