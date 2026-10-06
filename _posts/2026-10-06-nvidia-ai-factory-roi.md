---
layout: post
comments: true
title: "NVIDIA, 생산성·내구성·범용성 갖춘 AI 팩토리로 투자 수익 극대화"
description: "메가와트당 처리량 30배(Vera Rubin NVL72), 토큰 비용 최대 45분의 1, A100 6년째 현역 — AI 팩토리 ROI를 좌우하는 세 요소에 NVIDIA가 어떻게 답하는지 원문을 대조해 정리한다."
img: nvidia_ai_factory_roi_title.webp
date: 2026-10-06 19:30:00 +0900
last_modified_at: 2026-10-06 19:30:00 +0900
tags: [nvidia, ai-factory, vera-rubin, gpu-economics, llm-infra, llm]
related: llm-infra
categories: [nvidia, llm-infra]
source_url: https://blogs.nvidia.co.kr/blog/productive-durable-fungible-ai-factories/
source_date: 2026-10-01
---

NVIDIA 한국 블로그가 10월 1일 AI 팩토리의 투자수익률(ROI)을 다룬 글을 냈다. 메가와트급 AI 팩토리 하나를 짓는 데 약 6,000만 달러가 들기 때문에, 운영 기업은 수익 전망이 분명해야 투자를 결정할 수 있다는 문제의식에서 출발한다. 글은 수익을 좌우하는 요소를 셋으로 나누고, 이에 대응하는 NVIDIA의 설계 방향을 생산성·내구성·범용성이라는 이름으로 묶는다. 제조사가 직접 쓴 글이라 수치는 대부분 외부 분석을 인용하는 방식이고, 여기서는 원문과 대조한 내용만 옮긴다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** AI 팩토리의 수익률은 수익 창출 능력, 유효 수명, 수요가 결정하며, NVIDIA는 이를 각각 생산성·내구성·범용성으로 대응한다. SemiAnalysis 데이터 기준 Vera Rubin NVL72는 GB300 NVL72보다 메가와트당 처리량이 30배 이상 높고 DeepSeek V4 Pro에서 100만 토큰당 비용이 최대 45분의 1이다. A100은 출시 6년이 지나도 초기 가격의 약 25% 가치를 유지하고, CoreWeave는 A100 주문 기간을 2029년까지 연장했다. 범용성은 AI 전 단계와 비AI 워크로드까지 한 아키텍처로 덮는다는 주장이다.

## 1. ROI를 가르는 세 변수

원문은 AI 팩토리 수익률을 결정하는 요소를 이렇게 나열한다.

- **수익 창출 능력**: 생산 가능한 모든 토큰을 판매한다고 가정했을 때 1년 동안 낼 수 있는 수익
- **유효 수명**: AI 하드웨어가 계속 수익을 낼 수 있는 기간
- **수요**: AI 팩토리가 만드는 토큰에 대한 수요 규모

<details class="evidence"><summary>원문 근거</summary><blockquote>"한 요소의 강점만으로 다른 요소의 약점을 완전히 상쇄할 수는 없습니다."</blockquote></details>

NVIDIA는 이 셋에 각각 생산성(수익 창출 능력), 내구성(유효 수명), 범용성(수요)을 짝지어 놓았다. 세 요소가 서로 독립이 아니라는 점도 짚는다. 더 다양한 워크로드를 처리하는 팩토리가 더 많은 수요를 확보하고, 그 수요가 해마다 이어지는 수익으로 연결된다는 논리다.

## 2. 생산성: 메가와트당 처리량

전력이 AI 팩토리 확장을 가로막는 결정적 제약이라서, 메가와트당 초당 토큰 처리량이 수익 창출 능력의 핵심 지표가 된다고 원문은 말한다. 같은 전력에서 토큰을 더 많이 만들수록, 토큰당 비용이 낮을수록 수익이 커진다.

SemiAnalysis AgentX 데이터에 따르면 Vera Rubin NVL72는 GB300 NVL72보다 메가와트당 처리량이 30배 이상 높다. DeepSeek V4 Pro 기준으로는 100만 토큰당 비용이 최대 45분의 1이다. 원문은 이 향상이 모델·워크로드부터 소프트웨어, 컴퓨팅, 네트워킹, 메모리까지 하나의 시스템으로 함께 최적화하는 "극한의 공동 설계(extreme codesign)"에서 나온다고 설명한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA Vera Rubin NVL72 시스템은 NVIDIA GB300 NVL72 대비 메가와트당 처리량이 30배 이상 높으며, DeepSeek V4 Pro 모델에서는 100만 토큰당 비용을 최대 45분의 1 수준으로 낮춥니다."</blockquote></details>

토큰 비용이 세대마다 크게 내려가면 컴퓨팅 수요가 줄지 않느냐는 질문에도 원문은 답한다. 줄지 않고 오히려 늘어난다는 것이다. 비용이 낮아지면 경제성이 생기는 활용 사례가 늘고, 그 새 사례가 쓰는 토큰이 효율 향상으로 아낀 양을 넘어선다는 설명이다.

## 3. 내구성: 세대가 바뀐 뒤에도 돈을 버는 하드웨어

원문이 내구성의 근거로 드는 사례는 다음과 같다.

- **A100(2020년 출시)**: 6년이 지난 지금도 상용 서비스에 쓰인다. CoreWeave는 2020년에 처음 도입한 A100의 주문 기간을 2029년까지 연장했다.
- **감가상각 기간**: 주요 사업자들이 서버 감가상각 기간을 계속 늘려왔다. Sprout의 2026년 9월 분석에 따르면 모든 주요 사업자가 서버 사용 수명을 연장했고, Microsoft의 V100은 회계상 내용연수 6년을 넘겨 8.4년간 운영됐다.
- **유효 수명 평가**: Barkr는 중고 판매 가격을 바탕으로 H100 8-GPU 시스템의 유효 수명을 5~6년, GB300 NVL72를 9~10년으로 평가했다.
- **중고 가치**: Silicon Data에 따르면 출시 6년 된 A100은 초기 가격의 약 25% 가치를 유지한다. 5년 감가상각을 적용하면 1년여 전에 가치가 0이 됐어야 한다는 점과 대비된다.
- **임대 시장**: Ornn Data는 A100을 5년 계약으로 빌릴 때 시장 가격이 1개월 계약 대비 약 80% 수준이라고 분석했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"2020년 출시된 NVIDIA A100 GPU는 6년이 지난 지금도 상용 서비스에 활용되며 경제적 가치를 입증하고 있습니다. CoreWeave는 최근 2020년 처음 도입한 A100 GPU의 주문 기간을 2029년까지 연장했습니다."</blockquote></details>

<details class="evidence"><summary>원문 근거</summary><blockquote>"출시된 지 6년이 지난 A100 GPU는 여전히 초기 가격의 약 25%에 해당하는 가치를 유지하고 있습니다."</blockquote></details>

소프트웨어 쪽 근거는 CUDA의 세대 간 호환이다. 새 아키텍처가 나와도 기존 하드웨어의 활용 가치가 사라지지 않고, 소프트웨어와 커널 최적화가 이어지면서 기존 장비의 성능도 계속 오른다는 설명이다. 다만 Sprout 자료는 회계상 내용연수가 실제 물리적 수명을 보수적으로 추정한 지표라는 단서가 달려 있다.

## 4. 범용성: 한 아키텍처로 AI 전 단계와 비AI 워크로드까지

특정 작업 하나만을 위해 지은 팩토리는 그 워크로드의 수요가 이어진다는 전제에 기댄다. 원문은 이 점을 단일 워크로드용 맞춤 ASIC과 NVIDIA GPU를 가르는 기준으로 삼는다. Tensor Cores와 Transformer Engine으로 프로그래밍 가능한 아키텍처에 AI 최적화 하드웨어를 얹었기 때문에 전문성과 유연성을 한 칩에서 낸다는 주장이다.

| 구분 | 원문이 말하는 범위 |
|------|----------|
| 모델 유형 | 언어·비전·생물학·물리학·로보틱스, 오픈 모델과 독점 모델 |
| 파이프라인 단계 | 데이터 처리, 사전 학습, 사후 학습, 추론 |
| 배포 환경 | 하이퍼스케일, AI 클라우드, 소버린 프로그램, 기업 데이터센터, 엣지 |
| 비AI 워크로드 | 데이터 처리, 과학 컴퓨팅, 시뮬레이션, 그래픽 |

CUDA 위에는 사전 구축된 CUDA-X 라이브러리가 1,000개 넘게 올라가 있고, 1,000만 명 이상의 개발자가 이를 기반으로 일한다고 원문은 밝힌다.

고객 사례는 다음과 같이 소개된다.

- **Lilly**: GPU 1,016개 온프레미스 클러스터에서 단백질·저분자·유전체학 모델을 구축·운영하고, 자체 팀용 챗봇과 에이전틱 워크플로우를 만든다.
- **Pinterest**: Blackwell, Hopper, 이전 아키텍처를 아우르는 GPU 14,000개 규모의 하이퍼스케일 클라우드에서 비전 언어 모델을 사후 학습하고 배포한다.
- **Revolut**: cuDF로 수십억 건의 거래 기록을 처리한 뒤 AI 클라우드에서 파운데이션 모델을 학습·배포한다.
- **Runway**: Hopper에서 월드 모델을 학습하고 Blackwell에서 서비스한다.
- **Texas A&M University**: 분자 시뮬레이션과 AI 신약 개발에 쓰는 슈퍼컴퓨터가 26개 프로젝트, 7개 기관에 걸쳐 95~98% 활용도를 기록한다.
- **Cosm**: 분산형 데이터센터로 고해상도 동영상 재생, 스트리밍, 실시간 그래픽, 동기화된 디스플레이를 처리한다.
- **Dassault Systèmes**: Wichita State의 항공기 인증과 Lucid Motors의 차량 설계에 쓰이는 버추얼 트윈 시뮬레이션을 지원한다.
- **Unilever**: 디지털 트윈으로 제품 이미지를 만들어 제작 비용을 절반으로 줄였다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Unilever는 사진 촬영 대신 디지털 트윈을 활용해 제품 이미지를 제작함으로써 제작 비용을 절반으로 줄였습니다."</blockquote></details>

## 5. 개발 환경에서 읽을 점

원문이 제조사 시각이라는 점을 감안하고, 인프라를 고르는 입장에서 가져갈 만한 것만 추렸다.

1. **TCO에 유효 수명을 넣는다.** 원문 근거로는 A100이 6년째 현역이고 중고 가치가 약 25% 남아 있다. 감가상각 기간을 짧게 잡은 계산은 실제 회수 기간을 과소평가할 수 있다.
2. **범용 클러스터로 합칠 수 있는지 먼저 본다.** Texas A&M의 95~98% 활용도는 여러 프로젝트가 한 시스템을 나눠 쓴 사례다. 전용 가속기를 들이기 전에 기존 GPU 클러스터로 워크로드를 통합할 수 있는지 검토해 볼 만하다.
3. **서빙 단가는 모델과 세대별로 다시 계산한다.** 45분의 1은 DeepSeek V4 Pro와 Vera Rubin NVL72 조합에서 나온 최대치다. 내 모델과 트래픽에서도 같은 폭이 나오는지는 vLLM·SGLang 같은 추론 스택으로 직접 측정해야 알 수 있다.

## 6. 참고 자료

- 원문: [NVIDIA, 생산성·내구성·범용성 갖춘 AI 팩토리로 투자 수익 극대화](https://blogs.nvidia.co.kr/blog/productive-durable-fungible-ai-factories/){:target="_blank"} (NVIDIA 한국 블로그, 2026-10-01)
- [SemiAnalysis AgentX: Vera Rubin NVL72 Agentic Inference](https://newsletter.semianalysis.com/p/vera-rubin-nvl72-agentic-inference){:target="_blank"}
- [Sprout: The Productive Life of a Data Center GPU](https://www.sproutup.com/resources/blog/how-long-does-a-data-center-gpu-actually-last){:target="_blank"} (2026-09)
- [Barkr: GPU 중고가 기준 유효 수명 평가](https://barkr.ai/market-report#the-resale-standard-forward-looking-gpu-valuations){:target="_blank"}
- [Silicon Data: A100 잔존 가치 분석](https://x.com/Silicon_Data/status/2100302896646512643){:target="_blank"}
- [Ornn Data: A100 임대 가격 분석](https://data.ornn.com/the-economics-of-open-weight-inference.pdf){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/fImCPTZ026U){:target="_blank"}
