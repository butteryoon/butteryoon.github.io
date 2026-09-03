---
layout: post
comments: true
title: "NVIDIA COMPASS — 코딩 에이전트가 크로스 임바디먼트 로봇 내비게이션 정책을 학습시키는 워크플로우"
description: "NVIDIA 테크니컬 블로그 해설. COMPASS 프레임워크로 X-Mobility 기본 정책을 잔차 RL로 특정 로봇·환경에 적응시키고 증류해 크로스 임바디먼트 내비게이션 정책을 얻는 과정을, Codex·Claude Code가 리포지토리 스킬과 승인 게이트로 자동화하는 방법을 정리한다."
img: command-title.webp
date: 2026-09-03 18:20:00 +0900
last_modified_at: 2026-09-03 18:20:00 +0900
tags: [nvidia, compass, x-mobility, robotics, residual-rl, agentic-workflow, isaac-sim, isaac-lab, codex, claude-code, ros2] # add tag
related: dev
categories: dev
---
NVIDIA 테크니컬 블로그에 올라온 [「AI 에이전트를 활용한 크로스 임바디먼트 로봇 내비게이션 정책 학습 방법」](https://developer.nvidia.com/ko-kr/blog/how-to-train-a-cross-embodiment-robot-navigation-policy-with-ai-agents/){:target="_blank"}(2026년 8월 28일, Yan Chang·Mihir Acharya·Wei Liu·Katie Washabaugh·Aishwarya Singh)을 읽고 정리했다. 사전 학습된 X-Mobility 정책을 잔차 강화학습(residual RL)으로 보정하고 여러 전문가를 하나의 정책으로 증류하는 COMPASS 프레임워크가 골자다. 이 학습 파이프라인 전체를 Codex나 Claude Code 같은 코딩 에이전트가 리포지토리에 담긴 스킬로 실행하고 사람은 승인 게이트에서만 개입한다는 구성이 흥미롭다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** COMPASS는 X-Mobility 기본 정책 위에 로봇·환경별 잔차 전문가를 RL로 학습시키고 이를 증류해 크로스 임바디먼트 내비게이션 정책을 만드는 프레임워크다. 리포지토리에 포함된 `compass`·`compass-doctor`·`compass-newembodiment` 스킬을 Codex(`$compass`)나 Claude Code(`/compass`)가 실행해 환경 검증, 씬 준비, 학습, 평가, 패키징을 진행한다. 각 단계는 로그·체크포인트·지표 같은 증거를 남겨야 다음으로 넘어가고 개발자 승인 게이트에서 사람이 판단한다.

## 1. COMPASS 아키텍처: 잔차 RL과 증류

COMPASS는 "Cross-embOdiment Mobility Policy via ResiduAl RL and Skill Synthesis"의 약자다. 구조는 세 층이다.

- **X-Mobility 기본 정책**: NVIDIA가 사전 학습한 범용 이동 정책이다. [Hugging Face nvidia/X-Mobility](https://huggingface.co/nvidia/X-Mobility){:target="_blank"}에서 체크포인트를 받는다.
- **잔차 전문가(residual specialist)**: 기본 정책의 출력을 보정하는 소형 RL 정책이다. 처음부터 학습하지 않고 특정 로봇·환경에 필요한 델타만 배우므로 데이터와 계산이 적게 든다.
- **증류(distillation)**: 로봇·환경별 잔차 전문가들의 데이터를 모아 하나의 공유 크로스 임바디먼트 정책으로 합친다.

원문의 그림 1이 이 흐름(기본 정책 → 임바디먼트별 전문가 → 크로스 임바디먼트 정책)을 보여준다. 튜토리얼의 기준 로봇은 Boston Dynamics Spot이다.

## 2. 에이전틱 워크플로우: 리포지토리 스킬과 승인 게이트

[COMPASS 리포지토리](https://github.com/NVlabs/COMPASS){:target="_blank"}는 `.agents/skills/` 아래에 세 개의 스킬을 담고 있다.

| 스킬 | 역할 |
|------|------|
| `compass` | 메인 워크플로우 오케스트레이션 |
| `compass-doctor` | 읽기 전용 진단 |
| `compass-newembodiment` | 새 로봇(임바디먼트) 온보딩 |

Codex에서는 `$compass` 프롬프트로, Claude Code에서는 `/compass`로 호출한다. Codex는 심볼릭 링크 스킬 디렉토리를 지원해 리포지토리 스킬을 기존 위치에 그대로 두고 쓸 수 있다.

워크플로우의 원칙은 두 가지다. 첫째, **증거 기반 진행**이다. 각 단계는 검증 가능한 산출물(로그, 비디오, 체크포인트, 지표)을 만들어야 다음 단계로 간다. 둘째, **개발자 승인 게이트**다. 환경 검증과 스모크 테스트 통과, 씬 등록과 점유 지도 검사, 학습 전후 평가와 체크포인트 승격 같은 지점에서 사람이 승인해야 진행한다.

원문이 제시하는 프롬프트 예시는 이런 식이다.

```
$compass Validate the COMPASS environment for Spot.
$compass Train and evaluate Spot in the built-in combined_multi_rack warehouse.
$compass Find suitable SAGE-10K living-room scenes for Spot.
$compass Compare the pretrained X-Mobility base policy with available Spot residual checkpoints.
$cuvslam-onboard Configure cuVSLAM as the odometry source.
```

## 3. 씬 소스 세 가지와 오도메트리 옵션

| 경로 | 설명 | 승인 게이트 |
|------|------|-------------|
| **내장 물류창고** (`combined_multi_rack`) | 즉시 재현 가능한 베이스라인. 등록된 씬과 점유 지도를 사용 | 단일 환경 스모크 테스트 후 1회 |
| **SAGE-10K** ([nvidia/SAGE-10k](https://huggingface.co/datasets/nvidia/SAGE-10k){:target="_blank"}) | 1만 개의 생성 실내 씬. 후보 선정 → USD 변환·등록 → 점유 지도 생성 → 스모크 테스트 | 2회 (변환된 USD 검사, 점유 지도 검사) |
| **Omniverse NuRec** | 스테레오 RGB 캡처로 실제 환경을 재구성해 Real2Sim 학습. 선택적 경로 | 별도 [워크플로우 문서](https://github.com/NVlabs/COMPASS/blob/real2sim/isaaclab_3.0/docs/handbook/workflows/nurec_real2sim.md){:target="_blank"} 참조 |

오도메트리는 별도 옵션이다. GPS가 차단되거나 불안정한 환경에서 카메라 기반 상태 추정이 필요하고 로봇이 검증된 오도메트리·변환 데이터를 제공하지 않는다면 [cuVSLAM](https://nvidia-isaac-ros.github.io/repositories_and_packages/isaac_ros_visual_slam/index.html){:target="_blank"}을 쓴다. `/chassis/odom` 토픽과 `odom`→`base_link` 변환을 제공하며 `$cuvslam-onboard` 스킬로 구성한다.

## 4. 하드웨어·소프트웨어 요구사항

원문이 검증한 구성은 다음과 같다.

- **OS**: Ubuntu 22.04 또는 24.04, Linux 드라이버 580.95.05 (Isaac Sim 6.0 기준 검증)
- **GPU**: RTX 계열, 최소 GeForce RTX 4080, VRAM 16GB 이상
- **시스템 RAM**: 최소 32GB
- **컨테이너**: Docker Engine 24 이상, NVIDIA Container Toolkit
- **시뮬레이션**: [Isaac Sim 6.0](https://docs.isaacsim.omniverse.nvidia.com/6.0.0/index.html){:target="_blank"} + [Isaac Lab 3.0](https://isaac-sim.github.io/IsaacLab/main/index.html){:target="_blank"} (고정 리비전)
- **자산**: `nvidia/COMPASS` 시뮬레이션 자산, `nvidia/X-Mobility` 체크포인트. Hugging Face 읽기 토큰 필요

토큰 취급을 두고 원문이 준 주의가 눈에 띈다. 토큰은 현재 셸에만 노출하고 에이전트 프롬프트에 붙여넣거나 소스 제어에 커밋하지 말라고 한다. 에이전트도 토큰을 요청·표시하거나 로그에 저장해서는 안 된다.

## 5. 잔차 학습 실행

Spot을 내장 물류창고에서 학습시키는 명령은 이렇다.

```bash
python run.py \
  -c configs/train_config.gin \
  -o ./outputs/spot_combined_multi_rack \
  -b ./assets/x_mobility.ckpt \
  --enable_cameras \
  --embodiment spot \
  --environment combined_multi_rack
```

`-b`로 X-Mobility 기본 체크포인트를 지정하고 `--environment`의 키를 바꾸면 SAGE-10K에서 등록한 씬으로 학습할 수 있다. `--num_envs`로 병렬 환경 수를 조절하며 단일 환경은 스모크 테스트 용도다.

학습 중에는 보상 요소, 목표 진척도, 접촉과 낙하, 에피소드 종료, 처리량, GPU 메모리를 모니터링한다. 원문은 마지막 이터레이션이 최고라고 단정하지 말고 주기적으로 체크포인트를 저장해 동일 조건에서 평가하라고 강조한다.

## 6. 평가와 배포

평가의 핵심은 **동일 조건 비교**다. 사전 학습된 X-Mobility 베이스라인과 잔차 체크포인트를 같은 시드, 목표, 초기 상태, 롤아웃 길이, 종료 조건으로 비교한다. 표준 COMPASS 지표는 목표 도달률, 낙하 빈도, 이동 시간이고 목표 진척도·접촉 동작·타임아웃·명령 안정성은 파생 증거로 따로 기록한다. 개발자가 승인한 체크포인트만 승격(promotion)과 패키징으로 넘어간다.

배포는 ROS 2 `compass_inference` 노드가 맡는다.

- **입력**: 전방 카메라 이미지, 내비게이션 타깃, 오도메트리에서 파생한 로봇 속도
- **출력**: `/cmd_vel` 토픽으로 선속도·각속도 발행
- 순환 상태와 이전 액션은 추론 내부에서 처리하며 외부 ROS 인터페이스로 노출하지 않는다.

실제 로봇에 올리기 전에는 좌표계, 업데이트 주기, 정규화, 명령 제한, 정지 동작, 물리 로봇 컨트롤러를 검증해야 한다.

## 7. 읽고 나서

이 글은 로봇 학습 튜토리얼이면서 동시에 에이전트 운영 사례다. 스킬을 리포지토리에 두고 단계마다 증거 산출물을 요구하고 사람은 승인 게이트에서만 개입하는 구조는 이 블로그에서 다룬 [Karpathy의 autoresearch 루프]({{site.baseurl}}/dev/2026/08/30/karpathy-autoresearch-loop.html)나 [Everything Claude Code 스킬 가이드]({{site.baseurl}}/dev/2026/09/02/everything-claude-code-skills.html)와 같은 방향을 가리킨다. 도메인이 코드 수정이든 로봇 정책 학습이든, 에이전트에게 맡길 수 있는 범위는 결국 "검증 가능한 산출물을 얼마나 촘촘히 정의하느냐"에 달린다.

잔차 학습 자체도 실용적이다. 기본 정책을 재사용하고 델타만 배우면 스크래치 학습보다 데이터와 GPU 시간이 줄고 증류로 정책을 하나로 합치면 여러 로봇에 배포할 때 관리할 모델도 하나가 된다. 원문은 절감 폭을 수치로 제시하지 않으므로 구체적인 비용은 직접 재현해 확인해야 한다.

## 8. 참고 자료

- [원문: AI 에이전트를 활용한 크로스 임바디먼트 로봇 내비게이션 정책 학습 방법](https://developer.nvidia.com/ko-kr/blog/how-to-train-a-cross-embodiment-robot-navigation-policy-with-ai-agents/){:target="_blank"}
- [COMPASS 리포지토리 (NVlabs)](https://github.com/NVlabs/COMPASS){:target="_blank"}
- [COMPASS 핸드북 빠른 시작](https://nvlabs.github.io/COMPASS/docs/quickstart.html){:target="_blank"}
- [X-Mobility 체크포인트](https://huggingface.co/nvidia/X-Mobility){:target="_blank"}
- [COMPASS 시뮬레이션 자산](https://huggingface.co/nvidia/COMPASS){:target="_blank"}
- [SAGE-10K 데이터셋](https://huggingface.co/datasets/nvidia/SAGE-10k){:target="_blank"}
- [Omniverse NuRec](https://developer.nvidia.com/omniverse/nurec){:target="_blank"}
- [Isaac Lab NuRec 가이드](https://isaac-sim.github.io/IsaacLab/develop/source/policy_deployment/03_compass_with_NuRec/compass_navigation_policy_with_NuRec.html){:target="_blank"}
- [cuVSLAM (Isaac ROS Visual SLAM)](https://github.com/nvidia-isaac/cuVSLAM){:target="_blank"}

