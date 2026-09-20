---
layout: post
comments: true
title: "bash 하나면 충분한가 — Microsoft가 에이전트 툴 인터페이스 5종을 비교한 결과"
description: "Microsoft 연구진이 TheAgentCompany와 APEX-Agents에서 typed tools, bash, programmatic tool calling 등 5가지 에이전트 툴 인터페이스를 비교했다. bash 단독이 typed tools보다 21.8~24.5pp 앞서고 토큰은 19~72% 적게 썼다."
img: command-title.webp
date: 2026-09-20 20:10:00 +0900
last_modified_at: 2026-09-20 20:10:00 +0900
tags: [agent, tool-interface, bash, llm, microsoft, benchmark, ai-research]
related: llm
categories: [ai-research]
---

에이전트에게 도구를 쥐여주는 방법은 대체로 둘 중 하나였다. 스키마를 꼼꼼히 정의한 typed tool 카탈로그를 주거나, 아니면 그냥 쉘을 열어주거나. 업계 통념은 전자 쪽이었다. 스키마가 명확할수록 모델이 덜 헤매고 감사도 쉽다는 이유였다. Microsoft Copilot AI 팀이 이 통념을 정면으로 재본 논문을 냈는데, 결과가 반대로 나왔다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Microsoft 연구진이 Opus-4.8과 GPT-5.5로 에이전트 툴 인터페이스 5종을 비교했다. bash 단독이 typed tools 대비 TheAgentCompany에서 21.8~24.5pp, APEX-Agents에서 4.8~7.4pp 높은 점수를 냈고 총 토큰은 19~72% 적게 썼다. bash 위에 typed tools나 에이전트가 직접 만든 툴을 얹어도 유의미한 추가 이득은 없었다.

## 논문 정보

- 제목: [Is Bash All You Need? An Empirical Study of Tool Interfaces for Enterprise Digital Worker Agents](https://arxiv.org/abs/2609.11999){:target="_blank"} (arXiv:2609.11999)
- 저자: Hazel Mak, Susheel Suresh, Sahil Bhatnagar, Barry Wang, Chhaya Methani, Alejandro Gutierrez Munoz (Microsoft)
- 소개 경로: [DAIR.AI 트윗](https://x.com/dair_ai/status/2099925472629150164){:target="_blank"} / [DAIR.AI Academy 페이퍼 페이지](https://academy.dair.ai/papers/is-bash-all-you-need-an-empirical-study-of-tool-interfaces-for-enterprise-digita-2609.11999){:target="_blank"}

## 무엇을 비교했나

쉘 기반 에이전트가 코딩 과제에서 잘한다는 건 이미 알려진 사실이다. 이 논문이 파고든 지점은 코딩 바깥이다. 앱과 서비스 사이를 오가고, 동료와 조율하고, 전문적인 분석을 수행하는 "엔터프라이즈 업무"에서도 범용 쉘이 전용 도구를 이길 수 있는가.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Shell-based agents have shown strong results in coding, but enterprise work also involves moving between applications and services, coordinating with coworkers, and performing professional analysis."</blockquote></details>

비교한 인터페이스는 다섯 가지다.

1. typed tools — 스키마가 정의된 도구 카탈로그
2. typed tools + bash
3. bash 단독
4. bash + 에이전트가 만들어 유지하는 툴(persistent agent-synthesized tools)
5. programmatic tool calling(PTC) — 코드를 실행하되 그 코드의 동작을 typed tool 카탈로그 안으로 제한

벤치마크는 TheAgentCompany와 APEX-Agents 두 종, 평가 모델은 Opus-4.8과 GPT-5.5다. 프론티어 모델 두 개로 교차 검증한 셈이다.

## 결과

bash 단독이 두 벤치마크 모두에서 typed tools를 앞섰다. TheAgentCompany에서 21.8~24.5pp, APEX-Agents에서 4.8~7.4pp. 점수만 높은 게 아니라 총 토큰도 19~72% 적게 썼다. 성능과 비용을 맞바꾼 결과가 아니라는 뜻이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Bash alone outperforms typed tools on both benchmarks, improving score by 21.8-24.5 pp on TheAgentCompany and 4.8-7.4 pp on APEX-Agents while using 19-72% fewer total tokens."</blockquote></details>

더 눈에 띄는 건 조합이 통하지 않았다는 점이다. bash에 typed tools를 더하거나, 에이전트가 스스로 만든 도구를 계속 쌓아두게 해도 풀링한 점수에서 감지할 만한 이득이 나오지 않았다. 흔히 기대하는 "좋은 것 둘을 합치면 더 좋아진다" 식의 시나리오가 성립하지 않은 것이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Adding typed tools or persistent tool synthesis to bash produces no detectable pooled score gain."</blockquote></details>

PTC는 중간 위치다. 직접 typed 호출보다 토큰을 덜 쓰면서 과제 성능은 대체로 비슷했지만, 품질과 비용 효율 양쪽에서 bash 단독에는 대체로 못 미쳤다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"PTC uses fewer tokens than direct typed calls with broadly similar task performance, but generally underperforms bash alone in both quality and cost efficiency."</blockquote></details>

## 실무에서 어떻게 읽을까

저자들의 권고는 조건부다. 임의 실행을 격리할 수 있으면 bash 단독, 보안이나 컴플라이언스 정책상 고정된 툴 카탈로그가 필요하면 PTC.

<details class="evidence"><summary>원문 근거</summary><blockquote>"For enterprise practitioners, these results favor bash alone when arbitrary execution can be isolated and PTC when security or compliance policies require a fixed tool catalog."</blockquote></details>

이 조건절이 핵심이다. bash가 이긴 이유를 추정해보면 표현력 문제로 보인다. 파이프, 리다이렉션, 제어문, 스크립트 재사용 같은 것들은 typed tool 카탈로그로는 조합 폭발 없이 담기 어렵다. 반대로 bash를 쓰려면 임의 코드 실행을 감당할 샌드박스가 먼저 있어야 한다. 격리 수단이 없는 환경에서 이 논문을 근거로 쉘을 열어주는 건 결론을 절반만 읽은 것이다.

툴 스키마를 정의하고 버전을 관리하는 유지비용을 생각하면, 샌드박스가 이미 있는 팀에게는 검토해볼 만한 선택지다. 에이전트 스택 구성 전반은 [오픈소스로 AI 에이전트 구축하기]({{site.baseurl}}/tools/2026/09/11/opensource-agent-stack.html)에 정리해뒀다.

다만 벤치마크 두 종의 결과라는 점은 감안할 필요가 있다. TheAgentCompany와 APEX-Agents가 각자의 업무 환경을 얼마나 대변하는지는 별개 문제고, 21.8~24.5pp라는 격차도 특정 과제 구성에 기댄 수치다.

## 참고 자료

- 논문: [Is Bash All You Need? An Empirical Study of Tool Interfaces for Enterprise Digital Worker Agents](https://arxiv.org/abs/2609.11999){:target="_blank"}
- [DAIR.AI Academy 페이퍼 페이지](https://academy.dair.ai/papers/is-bash-all-you-need-an-empirical-study-of-tool-interfaces-for-enterprise-digita-2609.11999){:target="_blank"}
- 벤치마크: TheAgentCompany, APEX-Agents
