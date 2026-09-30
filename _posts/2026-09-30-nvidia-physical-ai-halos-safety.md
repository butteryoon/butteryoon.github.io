---
layout: post
comments: true
title: "NVIDIA Halos — 피지컬 AI를 대규모로 배포하려면 모든 계층에 안전이 필요하다"
description: "NVIDIA 블로그 코리아 해설. 자율주행차와 로봇이 사람과 같은 공간에서 움직이게 되면서 하드웨어·소프트웨어·AI 거동·운영 환경·수명 주기 전반에 안전 증거가 필요해졌다. NVIDIA Halos의 자율주행·로보틱스 구성 요소, 생태계 참여 기업, TÜV SÜD·TÜV Rheinland·ANAB 인증 현황을 정리한다."
img: nvidia_halos_safety_title.webp
date: 2026-09-30 20:30:00 +0900
last_modified_at: 2026-09-30 20:30:00 +0900
tags: [nvidia, halos, physical-ai, safety, functional-safety, autonomous-vehicles, robotics, igx-thor, drive-agx-thor, llm-infra]
related: llm-infra
categories: [nvidia, ai-infrastructure, safety]
source_url: https://blogs.nvidia.co.kr/blog/physical-ai-halos-safety/
source_date: 2026-09-30
---
NVIDIA 블로그 코리아에 9월 30일 올라온 [「피지컬 AI를 대규모로 배포하려면 모든 계층에 안전이 필요한 이유」](https://blogs.nvidia.co.kr/blog/physical-ai-halos-safety/){:target="_blank"}를 읽고 정리했다. 자율주행차와 로봇이 도로·공장·창고처럼 사람과 함께 쓰는 공간으로 들어오면 배포 직전에 한 번 점검하는 방식으로는 안전을 입증할 수 없다는 게 원문의 출발점이다. 그 대안으로 NVIDIA가 내놓은 풀스택 안전 시스템이 Halos다. 나흘 전 정리한 [로보택시 3대 컴퓨터 글]({{site.baseurl}}/dev/2026/09/26/nvidia-robotaxi-leaders-full-stack.html)에서는 Halos가 한 문단으로만 나왔는데, 이번 글은 Halos 하나만 다룬다. 벤더 블로그라서 구성 요소와 인증 현황 위주로 읽었다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** NVIDIA Halos는 하드웨어부터 AI 거동과 운영 환경까지 피지컬 AI의 모든 계층에서 안전 증거를 만들고 추적하게 해 주는 풀스택 안전 시스템이다. 자율주행 쪽은 DRIVE AGX Thor·Hyperion·Halos OS·Alpamayo·안전 평가 프레임워크로, 로보틱스 쪽은 IGX Thor·Halos Core·Holoscan Sensor Bridge·Isaac Lab·Outside-In Safety 블루프린트로 구성된다. TÜV SÜD가 DriveOS 6.0을 ISO 26262 ASIL D로 인증했고, ANAB는 Halos AI Systems Inspection Lab을 ISO/IEC 17020 검사 기관으로 인정했다. 원문은 2035년까지 L3~5 자율주행차 4,900만 대(ABI Research), 2026~2035년 산업용 로봇 약 6,000만 대(Omdia)라는 전망을 근거로 들고, 처음부터 기능 안전을 설계에 넣느냐가 프로토타입과 확장 가능한 솔루션을 가른다고 말한다.

## 배경: 연구를 벗어나 대규모 배포로

원문은 시장 전망으로 시작한다. ABI Research는 2035년까지 레벨 3~5 자율주행차 보급 대수를 4,900만 대로, Omdia는 2026년부터 2035년까지 배포될 산업용 로봇을 약 6,000만 대로 본다. 기계가 이만큼 늘어나면 안전도 그 규모를 따라가야 한다는 주장이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"ABI Research는 2035년까지 레벨 3~5 자율주행차(AV) 보급 대수가 4,900만 대에 이를 것으로 전망하고, Omdia는 2026년부터 2035년 사이에 약 6,000만 대의 산업용 로봇이 배포될 것으로 추정하고 있습니다."</blockquote></details>

원문이 말하는 피지컬 AI의 안전은 AI로 움직이는 기계가 판단을 물리적 행동으로 옮길 때 안전하게 움직인다는 것을 입증하는 일이다. 자율주행차는 수년간 테스트를 거치며 하드웨어·소프트웨어 고장, 의도된 기능의 한계, AI 특유의 리스크에 어떻게 대응하는지 입증해 왔다. 로보틱스도 이제 같은 변곡점에 와 있다. 제조사·규제 기관·보험사·산업안전 담당 조직이 요구하는 것은 사람이 개입하지 않아도 이 모든 요소가 안전하게 맞물려 돌아간다는 증거다.

## 기존 안전 모델로는 부족한 네 가지 이유

원문은 새 안전 기준을 규정하는 변화로 네 가지를 든다.

- **환경이 역동적이다.** 도로와 공장, 창고는 고정 구역이나 차단벽만으로 통제할 수 없다. 시스템이 상황을 인지해 행동을 바꾸고 예상 밖의 일이 생기면 안전한 상태로 들어가야 한다.
- **AI 거동은 따로 보증해야 한다.** 기존 기능 안전 테스트에 더해 AI 소프트웨어까지 평가해야 하고 설계 시점·런타임·검증 시점의 가드레일을 함께 쓴다. ISO/IEC TS 22440 같은 새 표준이 AI 특유의 리스크를 다루기 시작했다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"테스트는 기존 기능 안전과 더불어 AI 소프트웨어까지 평가해야 하며, 설계 시점과 런타임, 검증 시점의 가드레일을 함께 활용해야 합니다. ISO/IEC TS 22440 같은 새로운 표준이 이러한 AI 특유의 리스크를 다루기 시작했습니다."</blockquote></details>
- **배포는 한 번으로 끝나지 않는다.** 소프트웨어·모델 업데이트, 새 작업, 달라지는 운영 조건 때문에 시스템은 계속 바뀐다. 중대한 변경이 있으면 추가 안전 테스트가 필요할 수 있다.
- **대규모 검증에는 시뮬레이션이 필요하다.** 가능한 시나리오가 너무 많고 복잡해서 실제 환경 테스트만으로는 모자란다. 시뮬레이션, 합성 데이터 생성, 시나리오 재구성을 함께 써야 한다.

그래서 원문의 결론은 안전이 하드웨어부터 AI 거동과 운영 환경까지 설계·배포·검증 전반에서 실제로 돌아가는 체계여야 한다는 것이다.

## Halos의 구성: 자율주행과 로보틱스

NVIDIA는 Halos를 피지컬 AI를 위한 "최초이자 유일한 풀스택 안전 시스템"이라고 부른다. 10년 넘게 자율주행 안전을 개발하며 쌓은 경험에서 나왔다고 한다. 원칙은 두 영역이 공유하지만 플랫폼과 표준, 증거는 영역마다 따로 준비한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA Halos는 피지컬 AI를 위한 최초이자 유일한 풀스택 안전 시스템으로, 개발자가 설계와 검증, 배포의 모든 계층에서 안전을 엔지니어링하도록 돕습니다. 원칙은 자율주행과 로보틱스가 공유하지만, 플랫폼과 표준, 증거는 각 영역에 맞게 따로 준비되죠."</blockquote></details>

### 자율주행차

| 계층 | 구성 요소 | 역할 |
|------|-----------|------|
| 하드웨어 | [NVIDIA DRIVE AGX Thor](https://www.nvidia.com/ko-kr/solutions/autonomous-vehicles/in-vehicle-computing/){:target="_blank"} | 안전을 고려해 설계한 가속 컴퓨팅 |
| | [NVIDIA Hyperion](https://www.nvidia.com/ko-kr/solutions/autonomous-vehicles/drive-hyperion/){:target="_blank"} | 레벨 4 자율주행차용 풀스택 차량 플랫폼과 레퍼런스 아키텍처 |
| 운영체제·미들웨어 | [Halos OS](https://blogs.nvidia.com/blog/halos-os-robotaxi-safety/){:target="_blank"} | ASIL-D 인증 DriveOS 기반의 통합 소프트웨어 토대 |
| | Halos Core, Halos Middleware | 시스템 격리, 모니터링, 결정론적 통신 |
| 엔드투엔드 모델 | [NVIDIA Alpamayo](https://www.nvidia.com/ko-kr/solutions/autonomous-vehicles/alpamayo/){:target="_blank"} | 롱테일 시나리오에 설명 가능성을 더하는 개방형 추론 비전-언어-행동(VLA) 모델 |
| 시뮬레이션·검증 | [Halos 안전 평가 프레임워크](https://docs.nvidia.com/common/resources/Nvidia_Halos_Safety_Evaluation_Framework_Tech_Brief.pdf){:target="_blank"} | 자동화 수준별 안전 논증(safety case)을 뒷받침할 증거를 만드는 도구와 지침 |

원문에 따르면 이 요소들이 맞물려 클라우드의 AI 개발·시뮬레이션이 차량 내 배포로 이어지고 안전 증거를 차량 수명 주기 내내 추적할 수 있다.

### 로보틱스

| 계층 | 구성 요소 | 역할 |
|------|-----------|------|
| 하드웨어 | [NVIDIA IGX Thor](https://www.nvidia.com/ko-kr/edge-computing/products/igx/){:target="_blank"} | 가속 컴퓨팅과 기능 안전을 담은 산업용 모듈. 전용 Functional Safety Island를 갖췄고 IEC 61508·ISO 13849 등에 맞춰 개발되는 시스템을 지원 |
| 소프트웨어 | IGX용 Halos Core | 결함 탐지·모니터링·보고 같은 안전 운영 기능의 토대, 센서·액추에이터 등 안전 구성 요소를 잇는 통신·처리 |
| 실시간 센싱 | [Holoscan Sensor Bridge](https://www.nvidia.com/ko-kr/technologies/holoscan-sensor-bridge/){:target="_blank"} | 센서 데이터를 AI·안전 처리와 연결해 잘못된 정보를 걸러 내고 정해진 안전 대응을 실행 |
| 시뮬레이션·검증 | [Isaac Lab](https://developer.nvidia.com/isaac/lab){:target="_blank"}, [Omniverse](https://developer.nvidia.com/omniverse){:target="_blank"} 라이브러리 | 여러 조건과 엣지 케이스에서 로봇 거동을 시험해 실제 환경 검증을 보완 |
| 아웃사이드인 안전 | [Halos Outside-In Safety 블루프린트](https://github.com/NVIDIA/halos-outside-in-safety){:target="_blank"}(오픈 소스) | 외부 카메라와 비전 AI 에이전트로 온보드 센서 너머까지 인지 범위를 넓히고 시설 단위 모니터링을 지원 |

두 영역에 모두 걸치는 것이 [Halos AI Systems Inspection Lab](https://www.nvidia.com/ko-kr/ai-trust-center/physical-ai/safety-certification/){:target="_blank"}이다. 안전·사이버보안·AI 안전 요구사항을 반복 가능한 검사로 바꾸고 Halos 통합 결과물이 제3자 기관의 최종 시스템 수준 인증을 받을 수 있도록 준비를 돕는다.

## 생태계 참여 기업

원문이 나열한 기업은 이렇다.

- **자율주행 차량 개발:** Geely, Isuzu, Nissan(Wayve 소프트웨어 탑재), Einride가 Halos OS의 지원을 받아 Hyperion 위에서 레벨 4 대응 차량을 만든다. Uber, Grab, Lyft 같은 모빌리티 사업자도 Hyperion으로 로보택시 개발과 배포를 넓히고 있다.
- **Inspection Lab 참여:** AUMOVIO, Bosch, Gatik, Hesai, Lucid, MIRA, onsemi, PlusAI, Sony, Valeo, Wayve. 자율주행 개발, ADAS, 센서, 반도체, 시스템 통합, 검증, 안전 보증 분야에 걸쳐 있다.
- **로보틱스:** acontis와 QNX가 임베디드 소프트웨어를 맡고 Advantech와 NexCOBOT이 안전을 고려한 IGX 시스템을 만든다. Infineon, NXP, STMicroelectronics, Texas Instruments는 센서와 안전 마이크로컨트롤러 같은 반도체를 공급한다. KION Group은 자율주행 지게차용 기능 안전 에이전트를 개발 중이고 Agility는 Digit 5 휴머노이드의 안전 시스템에 IGX Thor와 Halos Core를 통합하고 있다.

## 독립 평가와 인증

| 영역 | 기관 | 내용 |
|------|------|------|
| 자율주행차 | TÜV SÜD | 자동차 제품 수명 주기 소프트웨어 프로세스와 DriveOS 6.0을 ISO 26262 ASIL D로, 자동차 엔지니어링 프로세스를 ISO/SAE 21434로 인증 |
| | TÜV Rheinland | NVIDIA DRIVE AV에 대한 독립적 UNECE 안전성 평가 수행 |
| 로보틱스 | TÜV Rheinland | IGX Thor, Halos OS, Holoscan Sensor Bridge의 기능 안전 인증 준비 상태를 검사하는 중. TÜV SÜD가 ISO 26262 기준으로 Thor SoC와 Halos Core를 먼저 검사한 결과를 토대로 한다 |
| 피지컬 AI 전반 | ANAB | Halos AI Systems Inspection Lab을 ISO/IEC 17020 검사 기관으로 인정 |

로보틱스 쪽은 아직 "인증 완료"가 아니라 "인증 준비 상태 검사 중"이라는 점을 구분해 읽어야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"로보틱스 분야에서는 TÜV Rheinland가 NVIDIA IGX Thor와 Halos OS, Holoscan Sensor Bridge의 기능 안전 인증 준비 상태를 검사하고 있습니다. 앞서 TÜV SÜD가 ISO 26262를 기준으로 Thor SoC와 Halos Core를 검사한 결과 위에서 이뤄지는 작업입니다."</blockquote></details>

## 읽고 나서

원문은 이렇게 끝난다. 피지컬 AI를 확장하는 기업은 가장 뛰어난 시스템을 만드는 데서 멈추지 않고 평가받고 인증받아 현실에서 신뢰받는 시스템을 만들 것이며 처음부터 기능 안전을 설계에 넣느냐가 프로토타입과 확장 가능한 솔루션을 가른다는 것이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"처음부터 기능 안전을 설계에 넣는 것, 그것이 프로토타입과 확장 가능한 솔루션을 가르는 차이죠."</blockquote></details>

몇 가지가 눈에 띈다.

- **안전을 증거의 문제로 본다.** 규제 기관과 보험사가 받아 줄 증거를 수명 주기 내내 만들고 추적하는 게 Halos가 겨냥하는 지점이다. 안전 평가 프레임워크가 자동화 수준별 안전 논증을 뒷받침하는 도구로 설명되는 것도 같은 맥락이다.
- **AI 거동 보증이 별도 축으로 올라왔다.** 결정론적 소프트웨어를 전제로 한 기존 기능 안전만으로는 모자라고 설계·런타임·검증 시점의 가드레일을 함께 쓴다는 설명이다. 모델이 업데이트될 때마다 재검증이 필요할 수 있다는 지적은 LLM 에이전트를 운영할 때도 그대로 통한다.
- **인증 범위를 따져 읽어야 한다.** ASIL D 인증을 받은 것은 DriveOS 6.0과 소프트웨어 프로세스이고 로보틱스 구성 요소는 아직 인증 준비 상태를 검사받는 단계다. Inspection Lab 역시 최종 인증 기관이 아니라 제3자 인증을 준비하도록 돕는 검사 기관이다.
- **바로 손대 볼 수 있는 건 오픈 소스 블루프린트다.** Outside-In Safety 블루프린트는 GitHub에 공개돼 있어서 외부 카메라와 비전 AI 에이전트로 로봇 행동을 제어하는 구조를 직접 살펴볼 수 있다.

## 참고

- 원문: [피지컬 AI를 대규모로 배포하려면 모든 계층에 안전이 필요한 이유](https://blogs.nvidia.co.kr/blog/physical-ai-halos-safety/){:target="_blank"} (NVIDIA 블로그 코리아, 2026-09-30)
- [NVIDIA Halos — 자율주행차](https://www.nvidia.com/ko-kr/ai-trust-center/halos/autonomous-vehicles/){:target="_blank"} / [로보틱스](https://www.nvidia.com/ko-kr/ai-trust-center/halos/robotics/){:target="_blank"}
- [NVIDIA Halos 안전 평가 프레임워크 기술 요약(PDF)](https://docs.nvidia.com/common/resources/Nvidia_Halos_Safety_Evaluation_Framework_Tech_Brief.pdf){:target="_blank"}
- [NVIDIA/halos-outside-in-safety (GitHub)](https://github.com/NVIDIA/halos-outside-in-safety){:target="_blank"}
- [Halos AI Systems Inspection Lab](https://www.nvidia.com/ko-kr/ai-trust-center/physical-ai/safety-certification/){:target="_blank"}
- [Halos OS 소개(NVIDIA 블로그)](https://blogs.nvidia.com/blog/halos-os-robotaxi-safety/){:target="_blank"}
- 이 블로그의 관련 글: [NVIDIA 로보택시 3대 컴퓨터 — 학습·시뮬레이션·차량 내 컴퓨팅과 채택 기업 지도]({{site.baseurl}}/dev/2026/09/26/nvidia-robotaxi-leaders-full-stack.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/8gr6bObQLOI){:target="_blank"}
