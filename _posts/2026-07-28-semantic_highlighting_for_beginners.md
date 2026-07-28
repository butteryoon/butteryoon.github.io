---
layout: post
comments: true
title: "시맨틱 하이라이트란? — RAG 초보자를 위한 5분 요약"
description: "RAG에서 왜 검색은 똑똑한데 하이라이트는 멍청한지, 시맨틱 하이라이트가 그 간극을 어떻게 메우는지 비유와 그림으로 설명. 심화 글의 쉬운 입문 버전"
img: command-title.webp
date: 2026-07-28 19:10:00 +0900
last_modified_at: 2026-07-28 19:10:00 +0900
tags: [rag, semantic highlighting, beginner, tutorial, llm, nous research] # add tag
related: llm
categories: dev
---

"RAG"라는 말은 들어봤는데, 정작 **검색 결과에서 왜 하이라이트가 이상하게 되는지** 의아했던 적이 있다면 이 글이 답이 될 것이다. 심화 글([시맨틱 하이라이트 심층 분석]({{site.baseurl}}/dev/2026/07/28/semantic_highlighting_rag.html))의 **쉬운 입문편**으로, 용어를 최소화하고 비유 중심으로 5분 안에 읽히게 정리했다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** 보통의 "검색 강조"는 글자 그대로 맞는 단어만 노랗게 칠한다(키워드). 근데 RAG는 **뜻**으로 문서를 찾는다. 그래서 "iPhone 성능"으로 찾았는데 정작 답인 "A15 칩·벤치마크"는 노란색이 안 칠해진다. **시맨틱 하이라이트**는 뜻이 통하는 부분을 강조하니, 사용자가 "왜 이 문서가 나왔나"를 바로 안다.

## 1. 먼저 RAG가 뭔지 한 줄

RAG = "질문할 때, AI가 답을 지어내지 않고 **관련 문서를 먼저 찾아서** 그걸 보고 대답하는 방식".

찾는 건 벡터 검색이라 **뜻(의미)**으로 찾는다. "iPhone 성능"이라고 해도, 내용이 "A15 Bionic 칩이 빠르다"면 관련 있다고 판단해서 가져온다.

## 2. 문제 — 찾기는 똑똑한데, 강조는 멍청함

문서를 찾아서 보여줄 때, 대부분의 화면은 **키워드 하이라이트**를 쓴다. 즉 "질문에 쓴 단어와 똑같은 글자"만 노랗게 칠한다.

```
질문: "iPhone 성능"
키워드 강조: iPhone, 성능 ← 이 글자가 있는 곳만 노란색
```

그런데 찾아온 문서는 이렇게 생겼다 치자.

> "A15 Bionic 칩은 벤치마크 100만 점을 넘고, **lag 없이 부드럽게** 돈다."

이게 정확히 답인데, "iPhone"이나 "성능"이라는 글자가 안 들어가 있다. 그래서 **아무것도 강조 안 됨**. 사용자는 3,000자 문서를 처음부터 훑어야 "아, 여기서 성능 얘기하네"를 안다.

## 3. 그림으로 비교

키워드 방식 vs 시맨틱 방식을 한눈에 보여주는 그림 (Zilliz 모델 카드 발췌):

![키워드 vs 시맨틱 하이라이트 비교]({{site.baseurl}}/assets/img/semantic_vs_keyword_highlight.png)

- **왼쪽(키워드)**: "iPhone battery life strong"이라는 질문 단어와 똑같은 글자가 없어서 **아무것도 강조 못 함** (Failure)
- **오른쪽(시맨틱)**: 글자는 달라도 뜻이 통하는 "5000mAh large battery", "fast charging", "lasts two days on a single charge"가 노란색 (Success)

차이가 한 장에 다 보인다.

## 4. 시맨틱 하이라이트란

**시맨틱(semantic) = "의미의"**. 즉 뜻이 통하는 부분을 강조하는 것.

```
질문: "iPhone 성능"
시맨틱 강조: A15 Bionic 칩, benchmarks, lag ← 뜻이 같은 부분
```

단순히 예쁜 게 아니다. 두 가지 실익이 있다.

**① 사용자 신뢰**
"왜 이 문서가 나왔지?"를 바로 안다. 신뢰가 생기면 도구를 계속 쓴다.

**② 토큰 절약 (중요)**
RAG는 찾은 문서 전체를 AI에게 준다. 그런데 진짜 답은 몇 문장뿐, 나머지는 노이즈. 시맨틱 하이라이트로 **관련 문장만 골라서** AI에 주면, 토큰 비용이 **70~80% 줄어든다**(Zilliz 실측). 즉 "강조"가 "압축"이 된다.

## 5. 왜 그냥 ChatGPT한테 시키지 않나

"그럼 하이라이트할 때마다 LLM에게 '여기서 중요한 거 골라줘' 하면 되잖아?" — 안 된다.

- 하이라이트는 **검색할 때마다, 문서마다** 일어난다.
- LLM 호출 하나당 시간·돈이 든다. 매 검색마다 호출하면 느리고 비싸짐.
- 그래서 **작고 빠른 전용 모델**(Zilliz는 0.6B, 밀리초 추론)이 필요하다.

쉽게 말해: "매번 박사에게 물어보는" 게 아니라, "작은 전문가를 상시 대기시켜둔" 느낌.

## 6. 어디에 쓰이나

- **법률 문서**: 긴 계약서에서 관련 조항만 강조
- **고객지원**: 지식베이스에서 문제 해결 문장만 강조
- **쇼핑**: "오래가는 노트북" 물었을 때 명세서의 "18시간 배터리" 강조
- **에이전트**: AI가 여러 문서를 뒤질 때, 진짜 쓸모 있는 문장만 골라냄

## 7. 더 깊이 알고 싶다면

이 글은 입문편이다. 아래는 심화 버전:

- [시맨틱 하이라이트 심층 분석]({{site.baseurl}}/dev/2026/07/28/semantic_highlighting_rag.html) — Zilliz 모델(`semantic-highlight-bilingual-v1`) 스펙, 파이프라인, 벤치마크, OpenSearch/Provence 비교
- [AI Factory — 오픈소스 에이전트 인프라]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html) — RAG가 들어가는 bigger picture
- [OKF 지식 번들]({{site.baseurl}}/tools/2026/07/25/hermes_okf_knowledge.html) — 벡터 RAG의 약점을 어떻게 보완하는지

## 마무리

RAG의 숨은 병목은 "잘 찾는가"가 아니라 **"찾은 걸 어떻게 보여주나"** 다. 키워드 강조는 뜻을 못 읽어서 멍청해 보이고, 시맨틱 하이라이트는 뜻을 읽어서 신뢰+절약을 동시에 준다. 다음에 RAG 결과에서 하이라이트가 안 된 문서를 보면, "아, 이건 검색은 됐는데 강조가 키워드라 그런 거구나" 하고 바로 알 수 있을 것이다.

## 참고

- [Zilliz Semantic Highlight Model (HuggingFace)](https://huggingface.co/zilliz/semantic-highlight-bilingual-v1){:target="_blank"}
- [시맨틱 하이라이트 심층 분석 (관련글)]({{site.baseurl}}/dev/2026/07/28/semantic_highlighting_rag.html)
- [Reddit r/Rag 토론](https://www.reddit.com/r/Rag/comments/1qbjr5n/we_built_a_semantic_highlighting_model_for_rag/){:target="_blank"}
- [SkelterLabs — 하이브리드 검색 (한글 배경)](https://www.skelterlabs.com/blog/hybrid-search){:target="_blank"}
