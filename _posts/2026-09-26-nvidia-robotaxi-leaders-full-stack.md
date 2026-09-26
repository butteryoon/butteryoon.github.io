---
layout: post
comments: true
title: "NVIDIA 로보택시 3대 컴퓨터 — 학습·시뮬레이션·차량 내 컴퓨팅과 채택 기업 지도"
description: "NVIDIA 블로그 코리아 해설. 로보택시 개발 주기를 DGX 학습, RTX PRO 기반 Omniverse·Cosmos 시뮬레이션·검증, DRIVE AGX Thor 기반 Hyperion 10 차량 내 컴퓨터로 나눈 NVIDIA의 구도와, Uber·Waymo·Zoox·현대차그룹 등 모빌리티 사업자·AV 개발사·완성차 업체의 채택 현황을 정리한다."
img: nvidia_robotaxi_full_stack_title.webp
date: 2026-09-26 20:30:00 +0900
last_modified_at: 2026-09-26 20:30:00 +0900
tags: [nvidia, robotaxi, autonomous-driving, physical-ai, drive-hyperion, drive-agx-thor, alpamayo, cosmos, omniverse, llm]
related: llm
categories: dev
---
NVIDIA 블로그 코리아에 9월 20일 올라온 [「운전대를 잡은 피지컬 AI: 글로벌 로보택시 기업들은 NVIDIA 기술로 무엇을 만들고 있나」](https://blogs.nvidia.co.kr/blog/robotaxi-leaders-full-stack-open-platform/){:target="_blank"}를 읽고 정리했다. 원문은 로보택시 시장이 2035년 4,000억 달러 규모, 상용 차량 600만 대 이상으로 커질 것이라는 전망으로 시작한다. 무인 차량 한 대를 굴리는 것과 수천 대 플릿에서 똑같이 안전한 성능을 내는 것은 전혀 다른 문제이고 그 간극을 메우는 게 컴퓨팅이라는 주장이다. 벤더 블로그인 만큼 기술 구도와 파트너 목록 위주로 읽었다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** NVIDIA는 로보택시 개발 주기를 세 대의 컴퓨터로 나눈다. DGX에서 모델을 학습하고, RTX PRO 서버 위의 Omniverse·Cosmos로 시뮬레이션·검증하고, 차량에서는 DRIVE AGX Thor 두 개를 얹은 Hyperion 10으로 추론한다. Alpamayo VLA 모델에 메타 행동과 생각의 사슬(CoT) 추론 데이터를 넣자 최소 평균 변위 오차가 2.08에서 1.18로 43% 줄었다고 한다. Hyperion 10은 카메라 14대, 레이더 9개, 라이다 3개, 초음파 센서 12개를 쓰고 컴퓨팅과 센싱을 이중화했다. 원문은 상용 규모로 운영되는 주요 로보택시 프로그램이 모두 이 스택의 일부 또는 전부 위에서 돌아간다고 주장하며 Uber·Waymo·Zoox·현대차그룹 등 20여 곳의 채택 사례를 나열한다.

## 로보택시 기술 스택을 세 대의 컴퓨터로 나누기

원문이 말하는 로보택시 기술 스택은 데이터와 모델 학습에서 시작해 시뮬레이션·안전 검증을 거쳐 차량 내 실시간 컴퓨팅까지 이어지는 기술 전체다. NVIDIA는 이를 학습 컴퓨터, 시뮬레이션·검증 컴퓨터, 차량 내 컴퓨터로 나눈다. 개발사는 셋 중 하나만 가져다 써도 되고 셋을 모두 조합해도 된다는 게 "모듈형"이라는 표현의 뜻이다.

### 학습 컴퓨터: DGX와 Alpamayo

주행 모델은 플릿이 쌓아 가는 데이터로 계속 학습되고 그 학습이 DGX 시스템에서 돌아간다. 모델 쪽 빌딩 블록은 Alpamayo 포트폴리오다. 개방형 추론 비전-언어-행동(VLA) 모델, 시뮬레이션 프레임워크, 피지컬 AI 데이터세트로 구성되고 강화 학습 블루프린트와 포스트 트레이닝·증류 레시피가 함께 제공된다. 추론 모델은 복잡한 주행 상황을 작은 단계로 쪼개 하나씩 추론한 뒤 가장 안전한 궤적을 고른다. 드물게 나타나는 롱테일 상황을 겨냥한 설계다.

원문에 실린 수치는 하나다. 난도 높은 자율주행 평가에서 메타 행동과 CoT 추론 데이터를 추가했더니 최소 평균 변위 오차(예측 경로가 기준 경로에서 벗어난 평균 거리)가 2.08에서 1.18로 43% 줄었다. 어떤 벤치마크인지는 원문에 나오지 않는다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"난도 높은 자율주행 평가에서 메타 행동(meta-action)과 생각의 사슬(chain-of-thought) 추론 데이터를 추가하자 VLA 모델의 궤적 예측 정확도가 향상됐다. 예측 경로가 기준 경로에서 벗어난 평균 거리를 뜻하는 최소 평균 변위 오차가 2.08에서 1.18로 43% 줄었다."</blockquote></details>

### 시뮬레이션·검증 컴퓨터: RTX PRO 위의 Omniverse와 Cosmos

롱테일 시나리오는 실제 주행 거리를 아무리 늘려도 충분히 모이지 않는다. 그래서 Omniverse NuRec이 센서 데이터로 실제 주행 장면을 재구성하고 Cosmos 월드 파운데이션 모델이 그 장면을 물리 기반으로 변형한다. 수천 건의 실제 코너 케이스가 주행 거동·교통·날씨·조명·센서 조건을 뒤섞은 수백만 가지 조합으로 불어난다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"덕분에 개발자는 수천 건의 실제 코너 케이스를 주행 거동과 교통, 날씨, 조명, 센서 조건이 뒤섞인 수백만 가지 조합으로 늘릴 수 있죠."</blockquote></details>

이 워크로드는 RTX PRO 서버에서 폐루프 시뮬레이션과 검증으로 돌아간다. AlpaSim 시뮬레이션 프레임워크는 추론 기반 자율주행 모델의 학습·평가까지 이 흐름을 넓혀 배포 전에 약점을 찾게 해 준다.

### 차량 내 컴퓨터: DRIVE AGX Thor 기반 Hyperion 10

Hyperion은 레벨 4 대응 로보택시용 차량 내 컴퓨팅·센서 레퍼런스 아키텍처다. Hyperion 10은 Blackwell 플랫폼 기반 DRIVE AGX Thor SoC 두 개에 고화질 카메라 14대, 레이더 9개, 라이다 3개, 초음파 센서 12개를 묶어 실시간 360도 센서 퓨전을 한다. 컴퓨팅과 센싱을 이중화해서 센서나 컴퓨팅 부품 하나가 고장 나도 주행을 이어 갈 수 있다. 듀얼 Thor는 인식·추론·경로 계획·주행 조작용 VLA 모델까지 구동하도록 설계됐다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Hyperion 10은 NVIDIA Blackwell 플랫폼 기반의 NVIDIA DRIVE AGX Thor SoC 두 개에 고화질 카메라 14대와 레이더 9개, 라이다 3개, 초음파 센서 12개를 결합해 실시간 360도 센서 퓨전을 구현합니다."</blockquote></details>

안전은 Halos가 맡는다. Halos OS가 양산 수준의 안전 기반을 깔고 그 위에 독립 검사·시스템 검증·대규모 시뮬레이션, 클라우드에서 차량까지 이어지는 지속 테스트를 묶은 검증·인증 프레임워크를 얹는 구조다.

## 누가 어떤 층을 쓰는가

원문은 아시아·유럽·중동·북미에서 모빌리티 사업자, AV 개발사, 완성차 업체가 세 컴퓨터 중 어떤 것을 쓰는지 나열한다. 모두 NVIDIA가 밝힌 협력 현황이다.

### 모빌리티 플랫폼·운영사

| 기업 | 원문이 밝힌 협력 내용 |
|------|-----------|
| **Uber** | Hyperion 기반 플릿을 늘려 2028년까지 28개 도시에 도달할 계획. 드문 상황용 플릿 데이터를 선별하는 로보택시 AI 데이터 팩토리를 Cosmos 위에 구축 중. Autobrains, Avride, Lucid, May Mobility, Mercedes-Benz, Momenta, Nissan, Nuro, Pony.ai, Stellantis, Waabi, Wayve, WeRide, Zoox의 NVIDIA 기반 서비스를 Uber 플랫폼에 올리는 중 |
| **May Mobility** | Uber 네트워크로 호출 서비스를 운영할 계획. 소프트웨어 스택은 DRIVE 플랫폼 위에서 개발. 애틀랜타에서 Lyft와 시범 서비스를 시작 |
| **Bolt** | NVIDIA 기술로 유럽 전역에서 자율주행차 개발·확장 |
| **Lyft** | 향후 자율주행 플릿의 레퍼런스 아키텍처로 Hyperion을 쓸 계획 |
| **WeRide** | Grab과 손잡고 Hyperion·DRIVE AGX Thor 기반 GXR을 동남아시아 주요 시장에 선보일 계획 |
| **Waymo** | NVIDIA와 협력해 자율주행 컴퓨팅 시스템 구축 |

<details class="evidence"><summary>원문 근거</summary><blockquote>"Uber는 NVIDIA Hyperion 기반 플릿을 확대해 2028년까지 28개 도시에 도달할 계획입니다."</blockquote></details>

### AV 개발사

| 기업 | 원문이 밝힌 협력 내용 |
|------|-----------|
| **Wayve · Nissan · Uber** | Nissan 차량 엔지니어링, Wayve 임바디드 AI, Hyperion을 결합한 프로토타입으로 글로벌 로보택시 프로그램 개발 |
| **Autobrains** | 뮌헨에서 Uber와, 동남아시아에서 VinFast와 로보택시 프로그램 개발. Hyperion 위에 에이전틱 AI 기술을 얹는 형태 |
| **Zoox** | 차량 내 컴퓨팅과 클라우드 기반 학습·시뮬레이션에 DRIVE 활용 |
| **Momenta** | DriveOS 위에서 구동되는 DRIVE AGX 기반 소프트웨어 스택 개발 |
| **Pony.ai** | Hyperion과 DRIVE AGX Thor로 차세대 자율주행 도메인 컨트롤러 개발 |
| **Tensor** | DRIVE AGX Thor SoC 8개를 탑재한 레벨 4 Robocar 개발 |
| **Waabi** | Waabi Driver 플랫폼이 DRIVE AGX Thor 위에 구축됨. Uber와의 배포 협력으로 로보택시 시장 진출 |
| **TIER IV · Isuzu** | Hyperion·DRIVE AGX Thor 기반 레벨 4 자율주행 버스 배포 |
| **Lenovo** | SWM의 차세대 로보택시 프로그램에 DRIVE AGX Thor 기반 AD1 레벨 4 도메인 컨트롤러 공급 |
| **DeepRoute.ai** | DRIVE AGX Thor를 얹은 Hyperion 위에서 차세대 로보택시 개발 |

### 완성차 업체

| 기업 | 원문이 밝힌 협력 내용 |
|------|-----------|
| **Tesla** | NVIDIA 슈퍼컴퓨터에서 자율주행 신경망 학습 |
| **Mercedes-Benz · Uber** | 신형 S-Class 기반 로보택시 생태계 개발. Hyperion 아키텍처, 풀스택 DRIVE AV L4 소프트웨어, Alpamayo 개방형 모델·시뮬레이션 도구·데이터세트 활용 |
| **Stellantis · Wayve · Uber** | Hyperion과 AI 컴퓨팅으로 레벨 4 무인 모빌리티 서비스 개발·배포 |
| **Lucid · Nuro · Uber** | Hyperion 플랫폼의 일부인 DRIVE AGX Thor로 글로벌 로보택시 서비스 개발 |
| **현대자동차 · 기아** | 협력을 넓혀 Hyperion 기반 데이터 주도형 자율주행 시스템 개발. 합작법인 모셔널(Motional)과도 레벨 4 로보택시 서비스 발전 방안을 모색할 예정 |
| **Geely** | 생태계 파트너들과 Hyperion으로 로보택시를 개발·상용화할 계획 |
| **Zeekr** (Geely Auto Group 브랜드) | 중앙 집중형 도메인 컨트롤러에 DRIVE AGX Thor 채택 |

## 읽고 나서

표를 보면 "채택"의 깊이가 회사마다 크게 다르다. Tesla와 Waymo는 학습용 슈퍼컴퓨터나 컴퓨팅 시스템 협력 한 줄로 끝나지만 Mercedes-Benz는 Hyperion 아키텍처부터 DRIVE AV L4 소프트웨어와 Alpamayo까지 스택 전체를 가져간다. "주요 로보택시 프로그램이 모두 NVIDIA 스택 위에 있다"는 원문의 주장은 세 컴퓨터 중 하나라도 쓰면 포함된다는 뜻이다. 학습 인프라만 쓰는 곳과 차량 내 컴퓨터까지 NVIDIA로 채운 곳은 구분해서 읽어야 한다.

기술 쪽에서 눈여겨볼 대목은 VLA 모델이 차량 내 추론의 중심으로 올라왔다는 점이다. 학습 단계의 CoT 추론 데이터, 시뮬레이션 단계의 AlpaSim, 차량의 듀얼 Thor가 모두 추론 기반 주행 모델을 전제로 짜여 있다. 원문에 나온 정량 근거는 최소 평균 변위 오차 수치 하나뿐이니, 이 방향이 실제 도로 성능으로 이어지는지는 벤치마크와 운행 데이터가 더 나와야 판단할 수 있다.

## 참고 자료

- 원문: [운전대를 잡은 피지컬 AI: 글로벌 로보택시 기업들은 NVIDIA 기술로 무엇을 만들고 있나](https://blogs.nvidia.co.kr/blog/robotaxi-leaders-full-stack-open-platform/){:target="_blank"} (NVIDIA Korea, 2026-09-20)
- [NVIDIA 자율주행차 솔루션](https://www.nvidia.com/ko-kr/solutions/autonomous-vehicles/){:target="_blank"}
- [NVIDIA DRIVE Hyperion](https://www.nvidia.com/ko-kr/solutions/autonomous-vehicles/drive-hyperion/){:target="_blank"}
- [NVIDIA Alpamayo](https://www.nvidia.com/ko-kr/solutions/autonomous-vehicles/alpamayo/){:target="_blank"}
- [NVIDIA Halos 자율주행 안전 프레임워크](https://www.nvidia.com/ko-kr/ai-trust-center/halos/autonomous-vehicles/){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/rT3fpi94lnw){:target="_blank"}
