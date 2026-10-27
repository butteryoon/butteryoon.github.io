---
layout: post
comments: true
title: "에이전트가 장애물 코스를 짓고 로봇이 걷는다 — NVIDIA Microduck 시뮬레이션 워크플로"
description: "NVIDIA 기술 블로그의 Microduck 튜토리얼 정리. Hermes·Nemotron 3.5 Bolt가 계획과 인계 패키지를 만들고, Astra가 FreeCAD·Blender로 에셋을 제작하고, Isaac Lab에서 병렬 환경 4,096개로 정책을 학습한 뒤 RTX Spark의 Isaac Sim에서 검증하는 흐름과 바로 실행해 볼 수 있는 리소스."
img: microduck_ai_agent_sim_title.webp
date: 2026-10-27 18:00:00 +0900
last_modified_at: 2026-10-27 18:00:00 +0900
tags: [nvidia, robotics, isaac-lab, isaac-sim, newton, sim-to-real, rtx-spark, hermes-agent, llm-agent, agent-case]
related: agent-case
categories: [nvidia, robotics, llm-agent]
source_url: https://developer.nvidia.com/ko-kr/blog/using-ai-agents-to-teach-robots-like-microduck-to-walk-in-simulation/
source_date: 2026-10-23
---
로봇 정책을 학습시키려면 그 전에 할 일이 많다. 코스를 설계하고 3D 에셋을 만들고 시뮬레이션 환경을 조립해야 학습을 시작할 수 있다. NVIDIA 기술 블로그가 10월 23일 공개한 [「AI 에이전트로 Microduck 로봇에게 시뮬레이션에서 걷기 가르치기」](https://developer.nvidia.com/ko-kr/blog/using-ai-agents-to-teach-robots-like-microduck-to-walk-in-simulation/){:target="_blank"}는 이 준비 과정의 상당 부분을 에이전트에게 맡긴 튜토리얼이다. 로봇을 다루지 않더라도 계획 에이전트가 실행 에이전트에게 일을 넘기는 방식, 그리고 산출물에 무엇을 담는지는 에이전트 파이프라인을 짜는 사람이 눈여겨볼 만하다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 예제 로봇은 Pollen Robotics의 작은 2족 보행 오리 로봇 [Microduck](https://github.com/pollen-robotics/microduck){:target="_blank"}이다. RTX Spark에서 도는 Hermes 에이전트와 Nemotron 3.5 Bolt가 프롬프트를 6단계 계획으로 바꾸고 결과물을 인계 패키지로 묶어 Astra에게 넘긴다. Astra는 FreeCAD·Blender를 프로그래밍 방식으로 돌려 USD 에셋을 만들고 무료 공개 라이브러리 SimReady Central Library(OpenUSD 에셋 2,000개 이상)도 함께 쓴다. 정책은 클라우드의 Isaac Lab에서 RTX PRO 6000 Blackwell 한 장, 병렬 환경 4,096개로 적응 학습하고 RTX Spark의 Isaac Sim에서 롤아웃으로 검증한다. sim-to-real 대비는 BAM 서보 모델과 도메인 랜덤화로 하며 실제 하드웨어 배포 절차는 원문 범위 밖이다.

## 전체 흐름

원문 워크플로는 로컬과 클라우드를 오간다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA RTX Spark는 AI를 활용한 계획 수립과 정책 검증을 위한 로컬 개발 허브 역할을 합니다. 클라우드에서는 NVIDIA Isaac Lab을 사용해 강화 학습으로 로봇 정책을 학습합니다."</blockquote></details>

| 단계 | 누가 / 무엇으로 | 산출물 |
|---|---|---|
| 1. 장애물 코스 계획 | Hermes + Nemotron 3.5 Bolt (RTX Spark) | 프로젝트 계획, 컨셉 이미지, 에셋 목록, 인계 패키지 |
| 2. 시뮬레이션 에셋 제작 | Astra → FreeCAD, Blender | 편집 가능한 모델 + USD 에셋 |
| 3. 환경 조립 | Astra, Isaac Sim + Newton | 로봇·코스·센서가 들어간 학습 환경 |
| 4. 정책 학습 | Isaac Lab (클라우드) | 경사로·장애물 통과 정책 |
| 5. 정책 검증 | Isaac Sim (RTX Spark) | 완주·넘어짐·충돌·정지 기록, 롤아웃 영상 |
| 6. 실제 로봇 준비 | BAM 서보 모델, 도메인 랜덤화 | sim-to-real 격차를 줄인 시뮬레이션 설정 |

## 1단계: 프롬프트를 계획으로, 계획을 인계 패키지로

먼저 코스 구성과 Microduck이 배울 동작을 프롬프트로 쓴다. 원문이 예시로 든 프롬프트는 단계마다 무엇을 남겨야 하는지까지 지정한다. 일부를 옮기면 이렇다.

```text
항법과 보행 과제를 포함한 Microduck 시뮬레이션 장애물 코스를 구축하고 테스트하라.
사실적인 컨셉 이미지를 만들고, Astra가 이를 구축하고 테스트하게 하라.

워크플로
준비에는 Hermes와 Nemotron 3.5 Bolt를, 에셋 제작과 시뮬레이션에는 Astra를 사용하라.
1. 조사 및 계획 — Microduck 공식 문서를 검토하여 기능, 제약, 미확인 사항을 요약하라.
2. 컨셉 생성 — 검증된 Microduck 참조 이미지로 사실적인 코스 컨셉을 생성하라.
3. 에셋 식별 — 재사용 가능한 고유 에셋을 추출하고, 반복되는 인스턴스를 묶고, 대표 크롭을 선정하라.
4. 인계 준비 — 계획, 로봇 브리프, 컨셉, 에셋 목록, 크롭, 실행 지침을 패키징하라.
5. 에셋 제작 및 검증 — 시뮬레이션 전에 모든 에셋이 완료되었거나 진행 불가임이 입증되었는지, 사용 가능한 모델이 최소 하나 있는지 확인하라.
6. 조립 및 테스트 — 재현 가능한 지침과 근거를 제공하고, 물리 하드웨어는 조작하지 마라.
```

Hermes와 Nemotron은 Microduck 저장소와 문서를 훑어 로봇이 할 수 있는 동작과 추가로 확인할 점을 정리한다. 이어 로컬 이미지 생성 모델로 코스 컨셉 이미지를 만들고 이미지를 분석해 재사용할 에셋 목록을 뽑는다. 에셋마다 식별자와 외형 설명을 달고 해당 부분을 잘라 참조로 붙인다. 이미지 밖으로 잘렸거나 다른 물체에 가려진 부분도 기록해 둔다. 그래야 다음 단계에서 이미지로 확인한 정보와 추정할 부분을 구분할 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Hermes는 이러한 결과물을 하나의 인계 패키지로 묶습니다. 전체 프로젝트 계획, Microduck 조사 내용과 출처 링크, 컨셉 이미지, 에셋 목록, 크롭한 참조 이미지, 다음 단계의 지침이 포함됩니다."</blockquote></details>

패키지가 갖춰지면 Hermes가 Astra에게 작업을 자동으로 위임한다.

## 2단계: Astra의 에셋 제작

큰 컨셉 이미지에서 잘라 낸 크롭은 작고 흐릿하다. Astra는 클라우드 이미지 편집으로 객체를 분리해 선명한 참조 이미지를 만들고 이것을 원본 크롭과 비교해 실루엣·비율·색상이 유지됐는지 확인한다. 원본 크롭은 컨셉 이미지에 실제로 보였던 내용의 기록으로 남긴다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"이미지로 확정할 수 없는 치수와 가려진 면은 모델링 가정으로 기록합니다. Astra가 FreeCAD에 지시하여 기본 형상과 편집 가능한 CAD 구조를 만든 다음, Blender로 세밀한 형상과 머티리얼을 추가합니다. 두 애플리케이션은 모두 프로그래밍 방식 실행을 통해 로컬에서 실행됩니다."</blockquote></details>

결과물은 편집 가능한 모델과 내보낸 USD 에셋 두 벌로 저장되고 검토용 미리보기와 출력 점검 결과가 함께 나온다. 새로 만들 필요가 없는 물체는 [SimReady Central Library](https://huggingface.co/blog/nvidia/simready-central-assets-robotics-ai-factories){:target="_blank"}에서 가져온다. 시뮬레이션에 바로 쓸 수 있는 OpenUSD 에셋 2,000개 이상을 무료로 공개한 라이브러리다.

## 3단계: Isaac Sim과 Newton으로 환경 조립

Astra는 준비된 에셋으로 Isaac Sim에서 코스를 조립한다. Isaac Sim은 Omniverse 라이브러리의 렌더링과 센서 시뮬레이션을 맡고 Newton은 이동과 균형을 포함한 Microduck의 물리 동작을 모델링한다. 로봇 구조는 링크(연결된 물체), 조인트(물체 사이의 상대 운동 제약), 액추에이터(구동)로 정의하는데 원문은 이 구성도 에이전트가 거들 수 있다고 적었다. 이렇게 하면 움직이는 로봇, 상호작용할 코스, 그 결과를 피드백으로 돌려주는 시뮬레이션이 한 환경에 갖춰진다.

## 4단계: Isaac Lab 병렬 학습

Isaac Lab에서는 과제마다 관측, 행동, 보상을 정의하고 API로 구현한다. 오픈소스 GPU 가속 물리 엔진 [Newton](https://developer.nvidia.com/newton-physics){:target="_blank"}이 그 행동에 따라 로봇이 지면·환경과 어떻게 상호작용하는지 계산한다. 루프는 "시뮬레이션 관측값 → 정책 행동 → 시뮬레이션 동작 → 갱신된 관측값"이다.

이번 과제의 장애물은 경사로와 고르지 않은 지면이다. 출발점은 Astra가 만든 에셋과 Isaac Lab의 [Microduck 예제 환경](https://github.com/isaac-sim/IsaacLab/tree/main/source/isaaclab_tasks/isaaclab_tasks/contrib/microduck){:target="_blank"}이다. 이 환경에서 사전 학습한 보행 정책에 머리 카메라의 깊이 정보와 로봇 상태 관측을 더해 경사로와 장애물을 통과하도록 추가 학습했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"이 적응 학습에는 NVIDIA RTX PRO 6000 Blackwell Workstation Edition GPU에서 병렬 환경 4,096개를 사용했습니다."</blockquote></details>

원문은 예제 환경 링크를 `develop` 브랜치로 걸었지만 지금 IsaacLab 저장소에는 그 브랜치가 없다. 위 링크는 같은 경로가 있는 `main` 브랜치로 바꿨다(`flat_env_cfg.py`, `rough_env_cfg.py`, `agents/`, `mdp/`가 들어 있다).

## 5단계: RTX Spark에서 검증

학습한 정책은 RTX Spark로 가져와 Isaac Sim에서 돌린다. 학습 중에는 정책이 계속 바뀌지만 평가 때는 고른 정책 하나를 실행해 어디서 성공하고 어디를 보완해야 하는지 본다. 기록할 항목은 코스 완주 여부, 넘어짐, 충돌, 진행이 멈추는 상황이다. 이 기록을 롤아웃 영상과 함께 보면서 환경이나 학습 설정을 고친다.

RTX Spark에 Isaac Sim을 설치할 때는 [Windows x86 바이너리](https://docs.isaacsim.omniverse.nvidia.com/6.2.0/installation/download.html){:target="_blank"}를 받으면 된다. 원문에 따르면 Arm의 Windows x86 에뮬레이션으로 바로 실행된다.

## 6단계: 실제 로봇으로 넘어가기 전

원문은 하드웨어 배포 세부 절차를 다루지 않는다. 대신 그 전에 시뮬레이션을 실제 로봇에 가깝게 맞추는 두 가지를 소개한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Isaac Lab은 서보 응답을 모델링하는 Better Actuator Models(BAM)를 지원합니다. 명령 지연, 기계적 백래시(구동계의 미세한 유격), 모터가 공유 배터리에서 전력을 끌어 쓸 때 생기는 전압 강하를 모델링합니다."</blockquote></details>

[BAM](https://isaac-sim.github.io/IsaacLab/develop/source/concepts/actuators.html#bam-servo-model){:target="_blank"} 설정을 Microduck 실측값에 맞추면 sim-to-real 격차가 줄어든다. 그래도 센서, 지면 접촉, 제어 타이밍은 실제와 다를 수 있어서 [도메인 랜덤화](https://isaac-sim.github.io/IsaacLab/develop/source/how-to/transfer_policies_between_physx_and_newton.html#introducing-domain-randomization){:target="_blank"}로 물리 파라미터와 관측 노이즈를 현실적인 범위에서 흔들어 정책이 그 차이에 익숙해지게 한다. 원문에는 Isaac Lab에서 학습한 보행 정책으로 실제 Microduck이 걷는 영상도 실렸다.

## 에이전트 파이프라인에 옮겨 올 것

- **인계 패키지에 '모르는 것'을 적는다.** Hermes는 계획과 에셋 목록만 넘기지 않는다. 이미지에서 잘렸거나 가려진 부분, 미확인 사항, 출처 링크까지 함께 넘긴다. Astra도 확정할 수 없는 치수를 '모델링 가정'으로 남긴다. 다음 에이전트나 사람이 무엇을 다시 확인해야 하는지 산출물만 보고 알 수 있다.
- **종료 조건을 프롬프트에 박는다.** "모든 에셋이 완료되었거나 진행 불가임이 입증되었는지"처럼 실패도 완료로 인정하는 조건과, "물리 하드웨어는 조작하지 마라" 같은 금지선을 계획 단계 프롬프트에 넣었다.
- **데스크톱 도구는 프로그래밍 방식으로 돌린다.** FreeCAD와 Blender를 GUI 조작이 아닌 프로그래밍 실행으로 붙여서 에이전트가 직접 제어하고 결과를 점검할 수 있게 했다.
- **무거운 학습만 클라우드로 보낸다.** 계획과 검증은 로컬 RTX Spark에서, 병렬 환경 4,096개짜리 학습은 클라우드 GPU에서 했다.

## 바로 실행해 볼 리소스

| 리소스 | 용도 |
|---|---|
| [Microduck 저장소](https://github.com/pollen-robotics/microduck){:target="_blank"} | 로봇 문서·하드웨어 |
| [Isaac Lab Microduck 예제 환경](https://github.com/isaac-sim/IsaacLab/tree/main/source/isaaclab_tasks/isaaclab_tasks/contrib/microduck){:target="_blank"} | 학습 환경 출발점 |
| [Hugging Face Microduck sim-to-real 튜토리얼](https://huggingface.co/blog/nvidia/microduck-isaac-lab-sim-to-real){:target="_blank"} | 실습 가이드 |
| [네이티브 BAM 서보 Microduck 보행 데모](https://isaac-sim.github.io/IsaacLab/demos/v3/?demo=microduck){:target="_blank"} | 데모 |
| [Isaac Lab 문서 (v3.0.0-EA)](https://isaac-sim.github.io/IsaacLab/v3.0.0-EA/index.html){:target="_blank"} · [Isaac Lab Learning Path](https://docs.nvidia.com/learning/physical-ai/getting-started-with-isaac-lab/latest/index.html){:target="_blank"} | 강화 학습·모방 학습 프레임워크 |
| [Isaac Sim 문서](https://docs.isaacsim.omniverse.nvidia.com/latest/index.html){:target="_blank"} | 로봇·환경 시뮬레이션 |
| [SimReady Central Library](https://huggingface.co/blog/nvidia/simready-central-assets-robotics-ai-factories){:target="_blank"} | 공개 OpenUSD 에셋 |
| [Omniverse 라이브러리](https://developer.nvidia.com/omniverse){:target="_blank"} | 렌더링·시뮬레이션 통합 |

## 참고 자료

- 원문: [AI 에이전트로 Microduck 로봇에게 시뮬레이션에서 걷기 가르치기](https://developer.nvidia.com/ko-kr/blog/using-ai-agents-to-teach-robots-like-microduck-to-walk-in-simulation/){:target="_blank"} (NVIDIA Technical Blog, 2026-10-23)
- [Hermes Agent](https://hermes-agent.nousresearch.com/){:target="_blank"} · [NVIDIA Nemotron 3.5 Bolt (Hugging Face)](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Bolt-74B-A9B-24GB){:target="_blank"} · [RTX Spark](https://www.nvidia.com/ko-kr/products/rtx-spark/){:target="_blank"}
- 관련 글: [Nemotron 3.5 Super VL·Bolt 공개]({{site.baseurl}}/nvidia/llm-model/2026/10/23/nemotron-3-5-super-vl-bolt.html), [RTX Spark 출시 주간의 소프트웨어 스택]({{site.baseurl}}/nvidia-analysis/llm-serving/2026/10/16/nvidia-rtx-spark-launch-week-software-stack.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/0VCIV2h_Nlg){:target="_blank"}

