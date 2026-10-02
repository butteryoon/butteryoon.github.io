---
layout: post
comments: true
title: "모델 아래층을 누가 고치고 있나 — Hugging Face × OS4Science Fund"
description: "ESM-2는 2022년에 나왔는데 지금도 월 수십만 회 내려받는다. 누군가 그 아래 소프트웨어를 계속 고치고 있기 때문이다. Hugging Face와 Open Source for Science Fund가 Hub의 의존 라이브러리 정보로 인프라 지도를 그려 유지보수자를 지원한다."
img: os4science_fund_title.webp
date: 2026-10-01 23:55:00 +0900
last_modified_at: 2026-10-01 23:55:00 +0900
tags: [huggingface, os4science, open-source, maintainership, esm-2, czi-eoss, science, llm-science, llm]
related: llm-science
categories: dev
source_url: https://os4science.org/news/hugging-face-open-source-for-science-fund/
source_date: 2026-09-30
---

Thomas Wolf(Hugging Face 공동창업자)가 9월 30일 올린 [트윗](https://x.com/Thom_Wolf/status/2105341455245996398){:target="_blank"}과 그가 링크한 [Open Source for Science Fund 발표문](https://os4science.org/news/hugging-face-open-source-for-science-fund/){:target="_blank"}을 읽고 정리했다. 과학용 AI 모델이 오래 쓰이는 이유를 모델이 아니라 그 아래 소프트웨어에서 찾는 이야기다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** ESM-2는 2022년에 나왔는데 지금도 월 수십만 회 내려받는다. Wolf는 그게 가능한 이유를 "누군가 그 아래 소프트웨어를 계속 유지보수하기 때문"이라고 짚었다. Hugging Face와 Open Source for Science Fund는 Hub에 쌓인 모델별 의존 라이브러리 정보를 모아 **과학 연구가 기대고 있는 인프라 지도**를 그리고, 거기서 드러난 라이브러리의 유지보수자를 지원하기로 했다. Hub에는 매달 약 200개의 과학 모델이 새로 올라오고 누적 다운로드는 4,000만 회를 넘는다. 펀드는 Renaissance Philanthropy가 운영하고 Biohub와 Wellcome이 시드를 댔으며, 6년간 230개 넘는 프로젝트에 5,800만 달러를 집행한 CZI EOSS를 잇는다.

## 1. 트윗이 짚은 지점

<details class="evidence"><summary>원문 근거</summary><blockquote>"ESM-2 came out in 2022. It's still downloaded hundreds of thousands of times a month. That only works because someone keeps maintaining the software underneath it. @huggingface 🤝 @os4science are teaming up to find those libraries and back the people behind them" — Thomas Wolf, 2026-09-30</blockquote></details>

ESM-2는 Meta AI가 공개한 단백질 언어 모델이다. 네 해 전 모델이 지금도 꾸준히 쓰인다는 사실 자체보다, Wolf가 그 원인을 **모델 바깥**에서 찾은 점이 이 트윗의 핵심이다. 모델 가중치는 한 번 올라가면 그대로지만, 그걸 불러오고 돌리는 라이브러리는 파이썬 버전이 올라가고 의존성이 깨질 때마다 누군가 손봐야 한다. 그 손길이 끊기는 순간 모델은 "받을 수는 있지만 돌릴 수 없는" 상태가 된다.

## 2. 무엇을 하기로 했나

발표문이 밝힌 협업 내용은 세 가지다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Together, we will work to identify the software libraries that scientific model contributors rely on most, explore concrete opportunities to support the maintainers behind them, and surface the critical bottlenecks facing the open source modeler community."</blockquote></details>

- 과학 모델 기여자들이 **가장 많이 의존하는 라이브러리를 식별**한다
- 그 뒤에 있는 **유지보수자를 지원할 구체적 방법**을 찾는다
- 오픈소스 모델러 커뮤니티가 겪는 **병목을 드러낸다**

## 3. 의존성 지도는 어떻게 그리나

방법은 이미 있는 데이터를 쓰는 것이다. Hugging Face Hub는 모델마다 어떤 라이브러리에 의존하는지를 보여 준다. 이걸 수천 개 과학 모델에 걸쳐 모으면 개별 모델의 의존성이 아니라 **분야 전체가 무엇 위에 서 있는지**가 드러난다. 발표문은 이것을 "과학 연구가 기대는 인프라 지도"라고 부른다.

규모는 이렇다.

| 항목 | 값 |
|---|---|
| 매달 새로 공유되는 과학 모델 | 약 200개 |
| 과학 모델 누적 다운로드 | 4,000만 회 이상 |

여기서 한 가지 구분해 둘 것이 있다. 모델 카드에 적힌 의존 라이브러리는 **직접 의존**이다. 예컨대 [ESM-2 모델 카드](https://huggingface.co/facebook/esm2_t33_650M_UR50D){:target="_blank"}는 `transformers`로 불러오는 예제를 싣고 PyTorch·TensorFlow를 지원 프레임워크로 표시한다. 실제로 깨지기 쉬운 지점은 그 아래 전이 의존성에 더 많으므로, 지도를 그린다고 곧바로 "고쳐야 할 곳"이 나오지는 않는다. 발표문도 식별을 첫 단계로, 지원 방법 모색을 그다음 단계로 분리해 적었다.

## 4. 펀드의 구조와 계보

Open Source for Science Fund는 Renaissance Philanthropy 산하 펀드이고, Biohub와 Wellcome이 시드를 댄 다중 기부자 구조다.

계보도 분명하다. 2019년 Chan Zuckerberg Initiative가 시작한 EOSS(Essential Open Source Software for Science)가 선행 프로그램이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"In 2019, the Chan Zuckerberg Initiative launched the Essential Open Source Software for Science (EOSS) program... Over six years, this program deployed $58M across over 230 software projects... The Open Source for Science Fund builds on this foundation to scale our investments and support open source for the AI era."</blockquote></details>

6년간 5,800만 달러를 230개 넘는 프로젝트에 집행했고, 새 펀드는 그 토대 위에서 AI 시대를 겨냥한다는 설명이다. 다만 **새 펀드의 규모는 발표문에 나오지 않는다.** 금액이 공개되지 않은 상태라 EOSS보다 크다고 단정할 근거는 아직 없다.

## 5. 읽고 나서

- **지속가능성을 측정 가능한 문제로 바꾸려는 시도다.** "오픈소스 유지보수자를 지원해야 한다"는 당위는 오래된 이야기인데, 어디를 먼저 지원할지 정하는 근거가 늘 약했다. 모델 허브의 의존성 메타데이터를 집계해 우선순위를 뽑겠다는 접근은 그 빈자리를 겨냥한다.
- **다운로드 수가 유지보수 부담을 대변하지는 않는다.** 많이 쓰이는 라이브러리가 반드시 취약한 건 아니고, 적게 쓰이지만 대체 불가능한 것이 더 위험할 수 있다. 지도는 출발점이지 답이 아니다.
- **같은 방법을 자기 스택에 적용해 볼 수 있다.** 서빙·파인튜닝 파이프라인이 기대는 라이브러리를 나열하고 각각의 릴리스 주기와 이슈 응답 속도를 보면, 어느 의존성이 끊길 때 가장 아픈지가 눈에 보인다. 모델을 고르는 기준에 "이걸 돌리는 코드가 계속 관리되고 있는가"를 넣자는 이야기이기도 하다.

모델 성능 경쟁 뒤편의 유지보수 문제를 다룬다는 점에서, 평가와 데이터 품질을 들여다본 [RIVER 논문]({{site.baseurl}}/dev/2026/09/24/river-verifier-quality.html) 같은 글들과 같은 결을 공유한다. 눈에 띄는 지표보다 그 지표를 떠받치는 쪽이 실제로 결과를 가른다는 이야기다.

## 6. 참고 자료

- 발표문: [Hugging Face 🤝 Open Source for Science Fund](https://os4science.org/news/hugging-face-open-source-for-science-fund/){:target="_blank"} (2026-09-30)
- 트윗: [Thomas Wolf](https://x.com/Thom_Wolf/status/2105341455245996398){:target="_blank"} (2026-09-30)
- [Open Source for Science Fund](https://os4science.org/){:target="_blank"} / [펀드의 내력](https://os4science.org/about/the-funds-history/){:target="_blank"}
- [ESM-2 모델 카드 (facebook/esm2_t33_650M_UR50D)](https://huggingface.co/facebook/esm2_t33_650M_UR50D){:target="_blank"}
- 관련 글: [검증기 품질이 환경 개수를 이긴다 — RIVER]({{site.baseurl}}/dev/2026/09/24/river-verifier-quality.html)
