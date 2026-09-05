---
layout: post
comments: true
title: "Talkie — 1931년 이전 텍스트만으로 학습한 13B 빈티지 언어모델"
description: "Alec Radford·Nick Levine·David Duvenaud 팀이 공개한 talkie-1930 해설. 1931년 이전 영어 텍스트 260B 토큰(전량 OCR)으로 학습한 13B 오픈웨이트 모델과, 같은 아키텍처·연산량의 FineWeb 쌍둥이 모델로 구성된 통제 실험 설계를 살펴본다."
img: talkie-vintage-title.webp
date: 2026-09-06 00:45:00 +0900
last_modified_at: 2026-09-06 00:45:00 +0900
tags: [talkie, vintage-model, llm, generalization, open-weight, radford, duvenaud] # add tag
related: llm
categories: dev
---

"1930년의 언어모델과 대화한다면 어떨까?" **talkie**는 이 질문을 실제로 구현한 연구다. Alec Radford(GPT 시리즈의 그 Radford), Nick Levine, David Duvenaud 팀이 2026년 4월 공개한 talkie-1930은 **1931년 이전 영어 텍스트만으로 학습한 13B 오픈웨이트 모델**로, 지식 단절(knowledge cutoff)을 거의 한 세기 뒤로 옮겨놓은 흔치 않은 실험 자산이다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 출처 검증·저자 정정 후 발행했다.)

<!--more-->

> **TL;DR:** talkie-1930-13b는 서적·신문·정기간행물·과학 저널·특허·판례 등 **1931년 이전 영어 텍스트 260B 토큰**으로 학습됐다 — 1930년엔 디지털 출판이 없었으므로 **전량 물리 매체를 OCR로 전사**했다. 백미는 통제 실험 설계: 같은 아키텍처·같은 학습 연산량으로 FineWeb(현대 웹)을 학습한 쌍둥이 모델 **talkie-web-13b**를 함께 공개해 "데이터 시대"라는 단일 변수의 효과를 분리 비교할 수 있다. 전부 Apache 2.0.

## 무엇이 공개됐나

| 모델 | 학습 데이터 | 용도 |
|------|-------------|------|
| [talkie-1930-13b-base](https://huggingface.co/talkie-lm/talkie-1930-13b-base){:target="_blank"} | 1931년 이전 영어 260B 토큰 | 빈티지 베이스 모델 |
| [talkie-1930-13b-it](https://huggingface.co/talkie-lm/talkie-1930-13b-it){:target="_blank"} | 베이스 + **1931년 이전 참고 문헌에서 추출한 지시-응답 쌍** | 대화형 — "과거와의 대화" |
| [talkie-web-13b-base](https://huggingface.co/talkie-lm/talkie-web-13b-base){:target="_blank"} | FineWeb (현대 웹) — 동일 아키텍처·동일 FLOPs | **통제군** — 시대 변수 분리용 |

지시 튜닝 데이터까지 1931년 이전 자료(사전·백과류 참고 문헌)에서 뽑았다는 점이 인상적이다 — 현대 어시스턴트 말투가 스며들 통로를 학습 전 단계에서 차단했다.

## 왜 흥미로운가 — 세 가지 연구 축

1. **분포 이동 하의 일반화**: 프로그래밍 언어도, 현대 과학도, 인터넷 용어도 본 적 없는 모델이 프롬프트만으로 현대적 개념을 얼마나 습득하는가. 스케일링 법칙 밖의 질문 — "무엇을 학습했는가"가 아니라 "학습 분포 밖으로 얼마나 나아가는가" — 를 정량화할 수 있는 드문 설계다.
2. **"미래 예측" 실험**: 1930년의 지식으로 멈춘 모델에게 이후의 발전을 추론시켜 볼 수 있다. 우리가 현재 모델의 "컷오프 이후"를 걱정하는 문제의 축소판을 100년 스케일로 재현한 셈이다.
3. **역사 언어 이해**: 당대 문체·어휘·지식 체계를 그대로 담은 모델은 역사·인문 연구의 도구도 된다.

프론티어 랩들이 비공개로 수행하는 데이터 ablation 성격의 실험을, 쌍둥이 통제군까지 갖춰 **오픈웨이트로 재현 가능하게** 공개했다는 점이 이 프로젝트의 가장 큰 기여다.

## 260B 토큰을 OCR로 — 데이터 파이프라인의 무게

1931년 이전 자료에는 디지털 원본이 없다. 즉 이 프로젝트의 260B 토큰은 서적, 신문, 정기간행물, 과학 저널, 특허, 판례를 **물리 매체 스캔에서 OCR로 전사**해 만들었다. 모델 학습보다 코퍼스 구축이 본체인 프로젝트라고 봐도 된다 — 저작권이 만료된 퍼블릭 도메인 자료라 라이선스도 깔끔하고, 결과물 전체가 Apache 2.0이다.

## 온프레미스 관점의 메모

- **13B 규모**는 단일 고급 GPU에서 추론 가능한 크기라, [OOD 일반화 평가]({{site.baseurl}}/dev/2026/08/24/rag_agent_weekly.html)의 대조 모델이나 데이터 필터링 효과 연구의 베이스라인으로 쓰기 좋다.
- 쌍둥이 모델 비교 프로토콜(같은 아키텍처·FLOPs, 데이터만 교체)은 사내 도메인 모델을 만들 때 **"데이터가 결과에 기여한 몫"을 분리 측정**하는 방법론으로 그대로 빌려올 수 있다.

## 참고

- [talkie 공식 사이트](https://talkie-lm.com/){:target="_blank"}
- [talkie-lm/talkie (GitHub, 추론 라이브러리)](https://github.com/talkie-lm/talkie){:target="_blank"}
- [Simon Willison의 talkie 노트](https://simonwillison.net/2026/Apr/28/talkie/){:target="_blank"}
- [MarkTechPost 소개 기사](https://www.marktechpost.com/2026/04/27/meet-talkie-1930-a-13b-open-weight-llm-trained-on-pre-1931-english-text-for-historical-reasoning-and-generalization-research/){:target="_blank"}
- [RAG & AI 에이전트 주간 동향 (관련글)]({{site.baseurl}}/dev/2026/08/24/rag_agent_weekly.html)
