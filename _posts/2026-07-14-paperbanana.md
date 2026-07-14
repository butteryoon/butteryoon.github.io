---
layout: post
comments: true
title: "PaperBanana - 문장으로 아키텍처 다이어그램 그리기"
description: "텍스트 설명만으로 논문 수준의 다이어그램을 생성해주는 AI 서비스"
img: tools_title.jpg
date: 2026-07-14 22:30:00 +0900
last_modified_at: 2026-07-14 22:30:00 +0900
tags: [llm, diagram, architecture, PaperBanana, ai agent] # add tag
related: llm
categories: tools
---

기술 문서나 블로그 글을 쓸 때 아키텍처 다이어그램을 그리는 일이 은근히 손이 많이 간다. draw.io나 [Python Diagrams]({{site.baseurl}}/tools/2021/05/13/python_diagrams.html) 같은 도구를 써왔는데, 이번에는 문장으로 설명만 하면 다이어그램을 만들어주는 **PaperBanana**라는 서비스를 소개한다.

<!--more-->

## PaperBanana

[PaperBanana](https://paper-banana.org/#generator)는 텍스트 설명을 입력하면 논문에 바로 쓸 수 있는 수준의 학술 일러스트를 생성해주는 AI 서비스다. 방법론 다이어그램, 시스템 아키텍처, 통계 차트, 플로우차트, 교육용 인포그래픽까지 지원한다.

북경대(Peking University)와 Google Cloud AI Research가 공동으로 발표한 연구([arXiv:2601.23265](https://arxiv.org/abs/2601.23265))가 기반이고, 코드도 [GitHub](https://github.com/dwzhu-pku/PaperBanana)에 공개되어 있다.

## 동작 방식

단순히 이미지 생성 모델에 프롬프트를 넣는 방식이 아니라, **5개의 전문 에이전트가 협업하는 closed-loop 구조**가 특징이다.

1. **Retriever**: 참고 자료 수집
2. **Planner**: 텍스트 설명을 구조화된 시각 레이아웃으로 변환
3. **Visualizer**: Nano-Banana-Pro 모델로 도형, 커넥터, 아이콘을 정밀하게 렌더링
4. **Critic / Refiner**: 결과물을 스스로 평가하고 반복 개선

같은 입력을 Nano-Banana-Pro 단독으로 돌리면 색감이 촌스럽고 내용이 장황한 다이어그램이 나오는데, PaperBanana를 거치면 훨씬 간결하고 보기 좋은 결과가 나온다고 한다.

특히 통계 차트는 픽셀로 그리는 게 아니라 **실행 가능한 Python Matplotlib 코드를 생성**하는 방식이라, 막대 높이나 축 눈금 같은 수치가 수학적으로 정확하고 숫자 환각(hallucination)이 없다는 점이 인상적이다.

## 사용 방법

1. [paper-banana.org](https://paper-banana.org/#generator)의 Generator에 원하는 그림을 자연어로 설명
2. 스타일, 비율(aspect ratio), 품질, 포맷 선택
3. 생성된 결과를 바로 다운로드

Transformer 아키텍처, RAG 시스템 같은 템플릿 예제도 제공하므로 처음에는 템플릿을 변형해가며 감을 잡는 것이 편하다.

## 요금

구독제로 운영되며 연간 결제 시 할인이 있다 (2026년 7월 기준).

| 플랜 | 월 요금 | 생성량 |
|------|--------|--------|
| Hobby | $13.50 | 720~1,440장/년 |
| Basic | $22.50 | 1,560~3,120장/년 |
| Pro | $45.00 | 4,200~8,400장/년 |

오픈소스 코드가 공개되어 있으므로 API 키만 준비하면 직접 돌려보는 것도 가능하다.

## 마무리

발표 자료나 기술 블로그용 아키텍처 그림을 문장 몇 줄로 뽑을 수 있다는 점에서 활용도가 높아 보인다. 특히 통계 차트를 코드 기반으로 생성해서 수치 정확성을 보장하는 접근은 다른 이미지 생성 서비스와 차별화되는 부분이다.

## 참고

- [PaperBanana 서비스](https://paper-banana.org/){:target="_blank"}
- [논문: PaperBanana: Automating Academic Illustration for AI Scientists](https://arxiv.org/abs/2601.23265){:target="_blank"}
- [GitHub: dwzhu-pku/PaperBanana](https://github.com/dwzhu-pku/PaperBanana){:target="_blank"}
