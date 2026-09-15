---
layout: post
comments: true
title: "앤스로픽 레드팀 보고서 — AI는 이미 킬 체인의 어디까지 들어왔나"
description: "정보 수집·대상 고정·무기 최적화 각 단계에서 프런티어 모델이 인간 전문가를 앞지르기 시작했다는 앤스로픽 프런티어 레드팀 보고서를 수치 중심으로 정리한다."
img: ai-military-capabilities_title.webp
date: 2026-09-15 19:00:00 +0900
last_modified_at: 2026-09-15 19:00:00 +0900
tags: [anthropic, ai-safety, red-teaming, national-security, geolocation, open-weights, llm]
related: llm
categories: dev
---
앤스로픽 프런티어 레드팀의 [「Intelligence, Targeting, and Conventional Weapons Capabilities」](https://www.anthropic.com/research/intelligence-targeting-conventional-weapons-capabilities){:target="_blank"}(2026-09-10)를 읽고 핵심 수치만 추렸다. 모델의 위험을 환각이나 편향으로 이야기하던 단계는 지났다. 정보 분석가와 무기 엔지니어가 훈련받아 하던 일을 모델이 시뮬레이션 환경에서 해내기 시작했고, 사진 한 장으로 위치를 짚는 과제에서는 사람 최상위권을 이미 넘어섰다. 앞서 다룬 [NVIDIA × CrowdStrike SafeMind 글]({{site.baseurl}}/dev/2026/09/07/nvidia-crowdstrike-safemind.html)이 사이버 영역의 공수 균형을 다뤘다면, 이번 보고서는 같은 질문을 물리 세계로 옮겨 놓는다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 앤스로픽은 탐색–고정–추적–타격으로 이어지는 킬 체인의 노동 집약적 단계를 모델이 대신할 수 있는지 평가했다. 야외 사진 지오로케이션에서 Mythos Preview·Mythos 5는 중앙값 오차 37.0km·47.2km로, 상위 0.01% 인간 플레이어(151km)를 앞섰다. 오픈 웨이트 모델은 프런티어에 못 미쳤지만 적대자를 식별하고 무기 성능을 개선하기에는 충분한 능력을 보였다.

## 킬 체인의 어느 칸이 자동화되는가

킬 체인은 원래 사람 손이 많이 드는 작업이다. 흩어진 단서에서 대상이 누구인지 좁히고(Find), 언제 어디 있는지 특정하고(Fix), 그 상태를 유지하며 따라가는(Track) 과정 대부분이 분석가의 시간으로 채워진다. 보고서가 겨눈 지점이 바로 이 시간이다.

- **신원 상관관계**: 여러 플랫폼에 흩어진 가상 페르소나를 한 사람으로 묶는 작업.
- **지오로케이션**: 메타데이터가 지워진 사진에서 촬영 위치를 추정하는 작업.
- **무기 관련 엔지니어링**: 설계와 성능 개선에 필요한 계산·코딩.

보고서가 반복해 강조하는 건 이 능력이 최상위 모델만의 것이 아니라는 점이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Models well short of the frontier will have intelligence and military applications."</blockquote></details>

## 사진 한 장으로 위치를 짚는다

가장 눈에 띄는 숫자는 지오로케이션이다. 사진 6,000장을 놓고 잰 중앙값 거리 오차에서 Mythos Preview가 37.0km, Mythos 5가 47.2km를 기록했다. 비교 대상인 인간 챔피언 디비전(플레이어 상위 0.01%)은 151km, 마스터 디비전은 174km였다. 1km 이내로 맞힌 비율은 각각 23.7%, 23.1%다.

같은 평가에서 Opus 5는 181km, Sonnet 5는 384km, Kimi K3는 385km로 격차가 컸다. 지오로케이션은 모델 계층에 따라 결과가 크게 갈리는 과제인 셈이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Mythos Preview and Mythos 5 beat even the strongest human baseline on median distance error, scoring 37.0 km and 47.2 km across 6,000 photos (placing 23.7% and 23.1% within 1 km)."</blockquote></details>

텍스트 기반으로 거주지를 추정하는 과제에서는 검색을 붙였을 때 중앙값 오차가 Mythos Preview 20.1km, Mythos 5 20.9km, Opus 5 21.7km, Sonnet 5 31.3km로 모델 간 차이가 훨씬 좁았다. 사진 쪽 성능이 세계 지식과 비전 능력이 맞물려야 나온다는 뜻으로 읽힌다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Based on this comparison, we believe the frontier of LLM intelligence is now approaching superhuman capabilities for geolocating outdoor photos."</blockquote></details>

## 분석가 2.5시간 vs 모델 11분

신원 상관관계 과제의 샘플은 중앙값 기준 약 37,000단어 분량이다. 사람 분석가가 읽기만 해도 2.5시간이 걸리고, 체계적으로 분석하려면 그보다 훨씬 오래 걸린다. Claude Mythos Preview는 평균 11분에 끝냈다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Across difficulty levels, the median sample is about 37,000 words of content. This would take a human analyst about 2.5 hours to read, and much longer to systematically analyze. Claude Mythos Preview took about 11 minutes on average."</blockquote></details>

## 오픈 웨이트 모델이라는 변수

닫힌 모델에 안전장치를 붙이는 일은 그나마 방법이 있다. 문제는 가중치가 공개된 모델이다. 앤스로픽이 테스트한 중국 개발사의 오픈 웨이트 모델들은 프런티어에 미치지 못했지만, 적대자를 식별해 겨냥하고 무기 성능을 끌어올리는 데는 쓸 만한 수준이었다. 같은 평가 전반에서 오픈 웨이트 모델은 대체로 Sonnet급과 Mythos급 사이에 자리 잡았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Although open-weights models from PRC developers that we tested were behind the frontier, they also showed concerning ability to identify and target adversaries, and improve weapon performance."</blockquote></details>

국가 기관만 갖고 있던 분석 역량이 값싸게 퍼진다는 건, 비밀 유지와 비용이라는 두 진입장벽이 동시에 낮아진다는 뜻이다.

## 앤스로픽이 제안하는 완화책

보고서 말미의 제언은 네 갈래다.

1. 닫힌 가중치 모델 쪽에서는 안전 조치를 실제로 배포할 것. 앤스로픽 세이프가드 팀은 무기 개발 관련 요청을 탐지·차단하는 분류기를 이미 도입했다.
2. 그 분류기가 완벽할 수 없다는 점은 인정한다. 기반 엔지니어링 능력 자체가 이중 용도이기 때문인데, 그래도 아무것도 안 하는 것보다는 일단 넣고 개선하는 편이 낫다는 입장이다.
3. 오픈 웨이트 모델의 안전 확보 연구가 시급하다. 개방에 따르는 이점은 분명하지만, 군사적으로 쓸 수 있는 전문성이 함께 퍼지는 문제는 따로 따져봐야 한다.
4. 정책 입안자는 이 확산에 대한 회복력을 높이거나, 법 집행·규제·안보 당국이 대응할 수 있게 뒷받침할 방안을 검토해야 한다.

여기에 하나가 더 붙는다. 사이버 보안에서 그랬듯 프런티어 모델의 지능이 방어하는 쪽에도 이점이 될 수 있다는 것이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"As we have seen in cybersecurity, frontier model intelligence can be an advantage for defenders."</blockquote></details>

## 정리

읽고 나서 가장 오래 남은 건 23.7%라는 숫자였다. 사진 네 장 중 한 장꼴로 1km 안쪽을 짚는다면, 우리가 막연히 기대해온 온라인상의 익명성은 이미 상당 부분 유효기간이 지난 셈이다.

모델이 똑똑해질수록 정렬과 거버넌스 이야기가 따라붙는 게 당연한 수순이라고 여겨왔는데, 이 보고서는 그 순서를 조금 앞당긴다. 능력은 이미 배포돼 있고, 통제 쪽이 뒤를 쫓는 상황이다.

## 참고

- [Intelligence, Targeting, and Conventional Weapons Capabilities — Anthropic](https://www.anthropic.com/research/intelligence-targeting-conventional-weapons-capabilities){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/VBNb52J8Trk){:target="_blank"}
