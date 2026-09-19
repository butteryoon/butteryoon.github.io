---
layout: post
comments: true
title: "NVIDIA DSX — 메가와트를 토큰으로 바꾸는 AI 팩토리 전력 최적화"
description: "NVIDIA 블로그 해설. Lambda가 DSX MaxLPS로 같은 전력 예산에서 노드 16개 대신 19개를 돌려 토큰 처리량을 24% 늘린 검증 결과와, Silicon Valley Power 수요 신호에 1분 안에 응답한 Emerald AI Conductor 실증, Vera Rubin NVL72의 메가와트당 GPU 40% 추가 확보 전망을 정리한다."
img: nvidia_dsx_ai_factory_power_title.webp
date: 2026-09-19 19:30:00 +0900
last_modified_at: 2026-09-19 19:30:00 +0900
tags: [nvidia, dsx, vera-rubin, ai-factory, power-efficiency, ai-infra, tokens-per-watt, emerald-ai, lambda, groq-3-lpx, llm]
related: llm
categories: dev
---
NVIDIA 블로그 코리아에 9월 16일 올라온 [「NVIDIA, 메가와트에서 토큰까지 AI 팩토리 생산성 극대화」](https://blogs.nvidia.co.kr/blog/from-megawatts-to-tokens-how-nvidia-maximizes-ai-factory-production/){:target="_blank"}를 읽고 정리했다. 앞서 다룬 [AgentX 와트당 성능 글]({{site.baseurl}}/dev/2026/08/29/nvidia_vera_rubin_blackwell_agentx_perf_per_watt.html)이 칩과 랙의 효율 이야기였다면, 이번에는 팩토리 바깥, 그러니까 전력망까지 범위를 넓힌 이야기다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 클라우드 사업자 Lambda가 HGX B200 5랙 19노드 클러스터에서 DSX MaxLPS를 처음 검증했다. 풀파워 노드 16개에 해당하는 전력 예산으로 노드 19개를 돌려 토큰 처리량을 24%(초당 약 400만→500만 개) 끌어올렸고, 와트당 성능은 23% 개선했다. 산타클라라에서는 Emerald AI Conductor가 Silicon Valley Power의 수요 신호에 1분 안에 응답해 전력을 4MW에서 3MW로 내리면서도 우선순위 높은 추론은 그대로 돌렸다. NVIDIA는 차세대 Vera Rubin NVL72에서 DSX MaxLPS로 같은 메가와트 예산에서 GPU를 최대 40% 더 확보할 수 있다고 본다.

## 전력이 곧 제약이다

AI 팩토리 경제에서 잣대는 기가와트당 해내는 작업량이다. 젠슨 황은 "1기가와트 팩토리가 2기가와트 팩토리가 되는 일은 결코 없다"고 말했다. 전력은 늘리기 어려우니 같은 전력으로 더 짜내는 수밖에 없다는 뜻이다.

업계는 그동안 시설 설계부터 랙 단위 전력 변환까지 층층이 효율을 깎아 왔다. NVIDIA DSX는 그 원칙을 워크로드 자체로 끌어올린다. 랙 프로비저닝을 더 똑똑하게 해서 실제로 필요한 곳에 전력을 놓고, 촘촘한 스케줄링·빠른 재시작·가벼운 체크포인팅으로 GPU가 노는 시간을 줄인다. 지난 5월 GTC 타이베이에서 공개된 플랫폼이다.

DSX 제품군은 다섯 갈래다. 전력 최적화를 맡는 **MaxLPS**, 전력망 신호에 반응하는 **Flex**, 수명주기 관리용 오픈소스 소프트웨어 **OS**, 배포 전 설계를 검증하는 시뮬레이션 도구 **Sim**, 그리고 세대별 검증 아키텍처인 **레퍼런스 디자인**이다.

## DSX MaxLPS: 같은 예산으로 더 많은 컴퓨팅

MaxLPS는 GPU와 랙 단위 소비 전력을 실시간으로 지켜보다가 노드 사이로 여유 전력을 옮긴다. 학습과 추론은 전력을 쓰는 방식이 다르기 때문에, 둘이 섞여 도는 팩토리일수록 재배분 여지가 크다. 정적 프로비저닝이 묶어 둔 용량을 되찾아 오는 구조다.

AI Infra Summit에서 공개된 Lambda의 결과가 HGX B200 서버에서의 첫 검증 사례다.

- 5개 랙, 19노드 클러스터
- 풀파워 노드 16개분 전력 예산으로 노드 19개 구동
- 클러스터 전체 토큰 처리량 **24% 증가** (초당 약 400만 → 500만 개)
- 와트당 성능 **23% 개선**

  <details class="evidence"><summary>원문 근거</summary><blockquote>"결과는 이랬습니다. 풀파워 노드 16개와 같은 전력 예산 안에서 노드 19개를 구동해 클러스터 전체 토큰 처리량을 24% 끌어올린 것인데요. 초당 약 400만 개에서 500만 개로 늘었고, 와트당 성능도 23% 개선됐습니다."</blockquote></details>

Lambda 클라우드 서비스 부문 사장 Dave Ward는 "고정된 전력 예산이라는 한계를 넘어섰다고 본다"며 "묶여 있던 용량을 되찾아 실제 활용으로 바꾼다"고 평가했다.

차세대는 폭이 더 크다. NVIDIA 예측으로는 적합한 배포 환경에서 Vera Rubin NVL72 AI 팩토리가 같은 메가와트 예산으로 GPU 용량을 최대 40%까지 더 확보한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA의 예측에 따르면 DSX MaxLPS는 적합한 배포 환경에서 차세대 Vera Rubin NVL72 AI 팩토리가 같은 메가와트 전력 예산으로 GPU 용량을 최대 40%까지 더 확보하게 해 줍니다."</blockquote></details>

## 산타클라라: 운영자 없이 1메가와트를 덜어낸 저녁

8월의 무더운 저녁, 해가 지고 에어컨 부하가 치솟자 Silicon Valley Power가 한 AI 팩토리에 전력 소비를 조정하라는 신호를 보냈다. Emerald AI 팀과 데이터센터 엔지니어, 전력회사 관계자 마흔 명 남짓이 Zoom으로 지켜봤고, 아무도 손을 대지 않았다.

주의할 점은 이게 DSX Flex를 설치한 사례가 **아니라는** 것이다. 원문은 이 대목을 분명히 못박는다. 산타클라라에서 돌아간 것은 NVIDIA의 Eos AI 팩토리이고, 거기서 동작한 소프트웨어는 Emerald AI의 전력망 오케스트레이션 플랫폼 **Conductor**다. DSX Flex보다 앞선 사례이자, 이 개념이 상용 규모에서 통한다는 증거라는 점에서 더 중요하다는 게 원문의 설명이다.

Eos는 Silicon Valley Power의 Flexible Load Interconnect Program에 참여하고 있다. AI 팩토리를 급전 가능한 자원으로 다루도록 설계된 첫 상용 전력회사 프로그램이다. 신호가 오면 Conductor는 1분 안에 응답한다.

- 이후 Silicon Valley Power가 보낸 수요 신호 **200건 이상**, 매번 빠짐없이 작동
- 우선순위가 가장 낮은 작업이 자리를 내주고, 우선순위 높은 추론은 계속 가동
- 전력 **4MW → 3MW** 자동 감축, 운영자 개입 없음

  <details class="evidence"><summary>원문 근거</summary><blockquote>"그 8월 저녁, Silicon Valley Power가 신호를 보내자 Conductor는 미리 정해 둔 워크로드 우선순위에 따라 실행에 들어갔습니다. 우선순위가 가장 낮은 작업이 자리를 내주고, 우선순위가 높은 추론은 계속 돌아갔으며, 전력은 4메가와트에서 3메가와트로 떨어졌죠. 모두 자동이었고, 운영자는 필요하지 않았습니다."</blockquote></details>

DSX Flex는 이 패턴을 일반화하려고 만든 물건이다. 부하 차단 요청, 수요 반응 이벤트, 가격 신호 같은 전력망 신호를 받아 미리 정해 둔 우선순위대로 자동 대응한다. 플랫폼이 성숙하면 Emerald AI Conductor도 DSX Flex에 통합될 예정이다. DSX Flex를 전담으로 적용하는 첫 상용 배포는 버지니아주 매너서스에 들어서는 96MW급 Vera Rubin AI 팩토리이며, 두 개 대륙에서 진행한 다섯 차례 사전 실증 위에 세워진다.

새 송전선을 놓으려고 10년을 기다리는 대신 이미 깔린 전력망에서 여유를 끌어내는 길이라는 점에서, 이 실증의 함의는 시설 한 곳을 훌쩍 넘어선다.

## 에이전틱 워크로드와 Groq 3 LPX

여기서부터는 같은 날 [AI Infra Summit 정리 글](https://blogs.nvidia.co.kr/blog/ai-infra-summit-vera-rubin-dsx-energy-efficiencies-tokens-per-watt-ai-factories/){:target="_blank"}에 실린 내용이다. 앞의 전력 이야기와 짝을 이루는 추론 계층 쪽 숫자다.

에이전트가 추론 단계와 도구 호출을 사슬처럼 이어 가면 지연 시간과 컨텍스트가 순식간에 불어난다. 세션 하나에 쌓이는 입력 토큰이 단순한 채팅 요청의 약 15배다. [Groq 3 LPX]({{site.baseurl}}/dev/2026/08/26/nvidia_groq3_lpx_vera_rubin.html)는 Vera Rubin에 결정론적 초저지연 추론을 더해 MaxLPS를 보완한다.

- 10만 컨텍스트 **Qwen 3.8 27B** 워크로드에서 사용자당 초당 **2,529개 출력 토큰**
- **2조 개가 넘는 파라미터** 모델을 긴 컨텍스트로 돌릴 때, 두 기술을 합친 플랫폼이 GB200 NVL72 대비 메가와트당 토큰 처리량을 최대 **35배**까지 향상

  <details class="evidence"><summary>원문 근거</summary><blockquote>"두 기술을 합친 플랫폼은 2조 개가 넘는 파라미터 모델을 긴 컨텍스트로 돌릴 때 GB200 NVL72 대비 메가와트당 토큰 처리량을 최대 35배 까지 끌어올립니다. 숫자가 이를 말해 줍니다. 10만 컨텍스트의 Qwen 3.8 27B 워크로드에서 Groq 3 LPX는 사용자당 초당 2,529개의 출력 토큰을 기록했습니다."</blockquote></details>

35라는 숫자가 두 군데 나오는데 단위가 다르니 헷갈리지 말아야 한다. 팩토리 차원에서 DSX MaxLPS만 적용했을 때는 같은 사이트 전력 범위에서 GPU 최대 40% 추가에 토큰 처리량 최대 **35% 향상**이고, 위의 35배는 Groq 3 LPX를 결합한 플랫폼과 GB200 NVL72를 비교한 배수다.

SemiAnalysis AgentX 쪽 결과도 같이 공개됐다. DeepSeek V4 Pro에서 Vera Rubin NVL72는 GB300 NVL72 대비 메가와트당 처리량이 최대 30배 높았고, 100만 토큰당 비용은 최대 45배 낮게 나타났다. 전력이 제한된 배포 환경에서 메가와트당 처리량은 매출 규모를, 100만 토큰당 비용은 마진을 결정한다.

NVLink 6의 다층 복원력 아키텍처도 여기에 얹힌다. 물리 계층에서는 맞춤형 순방향 오류 정정과 재시도가 무손실 패브릭을 유지하고, 네트워크 계층에서는 크레딧 기반 흐름 제어와 동적 라우팅이 장애를 국소적으로 가둬 연쇄 정체를 막는다.

CPU 쪽에서는 스타트업들의 Vera CPU 측정 결과가 줄줄이 나왔다. Perplexity는 보안 샌드박스 플랫폼 SPACE에서 샌드박스 시작이 1.9배 빨라졌다고 밝혔고, DeepInfra는 오케스트레이션 단계 지연이 2.2배 줄었다고 보고했다. Redpanda는 다른 CPU 대비 지연 5.5배 감소에 처리량 73% 증가, Starburst는 쿼리 처리량 3배, Kinetica는 분석 쿼리 2.7배를 제시했다. ClickHouse는 ClickBench에서 "지금까지 측정한 것 중 가장 빠른 머신"이라고 평했다.

## 다음 계층: 800V DC

랙이 조밀해질수록 기존 저전압 전력 경로는 변환 복잡성과 배전 제약을 더한다. NVIDIA의 800VDC 아키텍처는 변환 단계를 줄이고 공급 효율을 높여 더 조밀한 가속 컴퓨팅 랙을 받치도록 설계됐고, DSX 레퍼런스 디자인에 반영돼 있다.

냉각이 얼마나 큰 몫인지 보여 주는 숫자도 하나 있다. 직접 액체 냉각을 쓰는 GB200 NVL72 랙은 전력이 컴퓨팅에 닿기 전에 약 120kW의 열을 어딘가로 내보내야 한다.

## 읽고 남는 것

부품 하나로는 팩토리를 최적화할 수 없다는 게 이 글의 뼈대다. 더 빠른 GPU도 네트워크를 기다리고, 잘못된 프로비저닝은 전력을 묶어 두며, 냉각은 GPU로 갈 전기를 가져간다.

정리하면 이런 얘기다. 첫째, 전력 효율은 곧 수익성이다. 고정 예산에서 처리량 24%가 늘었다는 건 같은 인프라로 추론을 24% 더 판다는 뜻이다. 둘째, 전력망 연동은 이제 실험이 아니라 상용 프로그램의 영역으로 넘어왔다. 워크로드에 우선순위를 매겨 두면 수요 반응과 서비스 품질을 동시에 가져갈 수 있다. 셋째, 에이전틱 워크로드는 컨텍스트와 도구 호출 연쇄 때문에 기존 서빙 스택의 가정을 흔든다. 넷째, DSX Sim처럼 자본이 묶이기 전에 병목을 찾는 시뮬레이션이 갈수록 중요해진다.

결국 질문은 하나로 모인다. 이 팩토리는 소비한 메가와트당 얼마나 쓸모 있는 일을 해내는가.

## 참고 자료

- 원문: [NVIDIA, 메가와트에서 토큰까지 AI 팩토리 생산성 극대화](https://blogs.nvidia.co.kr/blog/from-megawatts-to-tokens-how-nvidia-maximizes-ai-factory-production/){:target="_blank"} (2026-09-16)
- [AI Infra Summit: Vera Rubin과 DSX로 메가와트당 토큰 늘리기](https://blogs.nvidia.co.kr/blog/ai-infra-summit-vera-rubin-dsx-energy-efficiencies-tokens-per-watt-ai-factories/){:target="_blank"} — Groq 3 LPX·AgentX·Vera CPU 수치의 출처
- [Lambda 사례 연구](https://www.nvidia.com/en-us/case-studies/lambda/){:target="_blank"}
- [Groq 3 LPX 결정론적 실행 해설](https://developer.nvidia.com/blog/how-nvidia-groq-3-lpx-deterministic-execution-drives-power-efficient-high-interactivity-inference-on-nvidia-vera-rubin/){:target="_blank"}
- [SemiAnalysis AgentX 대시보드](https://inferencex.semianalysis.com/inference/deepseek-v4){:target="_blank"}
- [NVLink 6 다층 복원력](https://developer.nvidia.com/blog/how-nvidia-nvlink-6-delivers-multi-layer-resiliency-for-ai-factories/){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/q6n8nIrDQHE){:target="_blank"}
