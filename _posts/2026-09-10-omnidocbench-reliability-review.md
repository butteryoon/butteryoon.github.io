---
layout: post
comments: true
title: "OmniDocBench — 문서 파싱 벤치마크, 상용 성능 검증에 써도 될까?"
description: "문서 파싱 평가의 사실상 표준으로 떠오른 OmniDocBench를 CVPR 2025 논문·리더보드·LlamaIndex 비판까지 근거로 검토한다. 순위 비교 지표로는 훌륭하지만 형식 패널티·포화·단일 정답 특성 때문에 상용 합격 기준으로 단독 사용하기엔 한계가 있다."
img: ai_abstract_title.jpg
date: 2026-09-10 14:00:00 +0900
last_modified_at: 2026-09-10 14:00:00 +0900
tags: [omnidocbench, document-parsing, ocr, benchmark, evaluation, llm-ops, llm] # add tag
related: llm
categories: dev
---

문서 파싱 모델을 고르다 보면 성능 검증을 어떻게 할지가 늘 걸린다. 파서마다 "저희는 OmniDocBench에서 96점"이라고 광고하는데, 이 숫자를 상용 성능 품질 기준으로 곧이곧대로 받아들여도 되는지 의문이었다. RAGAS로 RAG 파이프라인을 정량 검증하는 과정을 정리했을 때와 같은 맥락에서, 이번엔 **문서 파싱 파이프라인의 기준이 되는 OmniDocBench라는 벤치마크의 신뢰도**를 원문 기반으로 따져본다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 사실 검증 후 발행했다.)

<!--more-->

> **TL;DR:** OmniDocBench(CVPR 2025, arXiv:2412.07626)는 현재 문서 파싱 분야의 **사실상 표준 참조 벤치마크**다. 1651 PDF 페이지, 10개 문서 유형, 수식·표·읽기 순서까지 어노테이션이 촘촘하고 EvalScope·CodeSOTA 등 여러 프레임워크가 갖다 쓴다. 그러나 ① 최근 모델 점수가 **포화**돼 남은 점수차는 엣지 케이스 잡기에 불과하고, ② BLEU/편집거리 지표가 **의미가 같은 출력을 형식이 다르다는 이유로 깎으며**, ③ 문서마다 **단일 정답(ground-truth)**만 인정해 도메인 맞춤형 파서를 저평가할 수 있다. 결론: **후보 선정용 순위 비교로는 적합하지만, 상용 성능 보증의 절대 기준으로 단독 사용은 부적합**하다. 실제 납품 문서 샘플로 직접 검증하는 2차 관문이 있어야 한다.

## 1. OmniDocBench는 무엇인가

OmniDocBench는 OpenDataLab(상하이 AI 랩 계열)이 공개한 문서 파싱 평가 벤치마크다. CVPR 2025에 채택됐고 arXiv:2412.07626으로 나왔다.

| 특성 | 내용 |
|------|------|
| 데이터 규모 | 1651 PDF 페이지 |
| 문서 다양성 | 10개 문서 유형(학술·금융·신문·교과서·필기 등) + 5개 레이아웃 + 5개 언어 |
| 어노테이션 | 28개 block-level + 4개 span-level 요소, 수식 LaTeX·표 HTML 주석, **읽기 순서**까지 |
| 평가 모드 | 엔드투엔드 / 레이아웃 / 표 / 수식 / 텍스트 OCR |
| 메트릭 | Normalized Edit Distance, BLEU, METEOR (+표 TEDS, 수식 CDM) |
| 라이선스 | 코드 저장소는 Apache-2.0 (데이터셋 이용 조건은 HuggingFace/OpenDataLab 배포 페이지에서 별도 확인) |

리더보드는 지금도 갱신되고 있다. 2026-04-30 v1.6→v1.7 업데이트에서 Qianfan-OCR 결과가 들어왔고, 최고점은 PaddleOCR-VL-1.6(0.9B)의 **96.34**다.

```text
| Model           | Type            | Size | Overall↑ | TextEdit↓ | FormulaCDM↑ | TableTEDS↑ |
|-----------------|-----------------|------|----------|-----------|-------------|------------|
| PaddleOCR-VL-1.6| Specialized VLM | 0.9B | 96.34    | 0.0326    | 97.53       | 94.76      |
| PaddleOCR-VL-1.5| Specialized VLM | 0.9B | 94.93    | 0.038     | 96.89       | 91.67      |
| MinerU-2.5      | Specialized VLM | 1.2B | 93.04    | 0.045     | 95.77       | 87.88      |
```

([공식 리더보드](https://github.com/opendatalab/OmniDocBench){:target="_blank"} v1.6_full 표에서 상위 3개 발췌 — 이 숫자를 취급할 때 주의할 점은 3절에서 다룬다.)

## 2. 신뢰할 만한 이유 — 왜 사실상 표준이 됐나

1. **학술·관리 기관**: CVPR 2025 채택, 상하이 AI 랩 계열 OpenDataLab이 관리하며 손을 놓지 않고 있다 (최근 2026-07 커밋).
2. **정교한 어노테이션**: 단순 텍스트 유사도가 아니라 수식(LaTeX)·표(HTML)의 구조 어노테이션과 **읽기 순서**까지 평가해, 이전 벤치마크가 놓친 계층을 짚는다. 논문 자체가 "기존 벤치마크는 학술 논문 위주 · 메트릭 비일관 · 미세 평가 부재"라는 한계를 지적하며 나왔다.
3. **생태계 채택**: LlamaIndex, EvalScope, CodeSOTA, LLM Stats 등이 통합하거나 인용하는 기준 리더보드.

비판하는 글을 쓴 LlamaIndex도 벤치마크 자체의 가치는 인정하고 시작한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"To be clear, I think OmniDocBench has provided significant value as a document benchmark." — LlamaIndex, "OmniDocBench is Saturated"</blockquote></details>

## 3. 신뢰도를 낮추는 요소 — 상용 결정에서 반드시 봐야 할 것

### 3-1. 포화(Saturation): 남은 점수차는 "엣지 케이스 잡기"

LlamaIndex의 2026 분석("OmniDocBench is Saturated")이 이 문제를 정면으로 짚는다. 최신 모델(GL-OCR ~94~96%)은 이미 정확도가 충분히 높아서, **숫자가 더 올라간다고 파싱 품질이 실제로 좋아졌다고 보기 어렵다**. 1~2% 차이는 특정 엣지 케이스를 하나 더 잡았다는 뜻이지, "진짜 더 나은 파서"라는 뜻은 아니다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"From a pure quantitative perspective, additional increases in OmniDocBench could reduce to 'edge case fixing' and doesn't represent 'true' improvements in doc parsing accuracy. Even at 1355 pages, the benchmark covers 9 doc types and misses a lot more complex documents, particularly those relevant to specific domains."</blockquote></details>

### 3-2. 형식 패널티: 의미 동등한 출력을 "오답"으로

OmniDocBench는 BLEU/편집거리 계열의 연속 지표를 쓴다. 그래서 **의미는 완전히 같은데 형식만 다른 출력**이 점수를 깎인다. LlamaIndex가 든 실제 사례:

> "LlamaParse outputs the formula table without merged cells. It also outputs the scientific notation in HTML rather than LaTeX format. These are still semantically correct."

이처럼 뜻은 같은데 병합 셀을 다르게 쓰거나 HTML 대신 Markdown으로 표현하면, 표 TEDS나 편집거리 지표에서 점수가 크게 깎인다. **AI 에이전트든 사람이든 최종적으로 쓰기엔 똑같은 결과인데도** 숫자로는 실패가 되는 셈이다.

### 3-3. 단일 정답(ground-truth) 고정

같은 문서를 세 개의 불릿으로 쓸 수도, 두 개의 표로 나눌 수도 있다. **정당한 표현이 여럿**이라는 뜻이다. 그런데 OmniDocBench는 문서마다 **하나의 고정 정답**만 인정한다. 정답과 표현만 다를 뿐 뜻이 같은 출력도 전부 감점 대상이다.

### 3-4. 언어·도메인 바이어스

저장소 "Known Issues"는 텍스트 평가에 **중국어·영어만** 들어간다고 못 박아 뒀다. 게다가 데이터가 **학술 논문 위주**로 쏠려 있어서, 실무에서 정작 중요한 폼(form)/필드·비정형 필기·차트 해석 같은 까다로운 유형은 상대적으로 얇다.

## 4. 상용 "성능 검증"에 쓰기 위한 실무 판단

| 용도 | 판정 |
|------|------|
| 후보 파서 1차 **후보 선정**(순위) | ✅ 적합 |
| 리더보드 순위를 **참고 자료**로 | ✅ 적합 |
| 최종 상용 선택의 **유일 근거** | ❌ 부적합 |
| 내 데이터로 **직접 의미·정확도 검증** 대체 | ❌ 부적합 |

상용 검증은 3단계로 가는 걸 권한다.

1. **후보 압축**: OmniDocBench 리더보드로 관심 모델을 2~3개로 줄인다. (순위는 믿어도 된다)
2. **직접 검증**: 실제 납품할 문서 샘플을 모아 각 파서로 파싱한 뒤, **형식이 아니라 내용 손실·오독·표 구조 정합성**을 기준으로 정성·정량 평가한다. 이때 잣대는 OmniDocBench 지표가 아니라 의미 정확도 중심의 자체 체크리스트다.
3. **수락 기준 수립**: 숫자 커트라인 대신 "핵심 필드 누락률", "표 셀 정렬 오류율" 같은 **업무 중요 지표**로 합격 여부를 가른다.

## 마무리

OmniDocBench는 문서 파싱 평가의 **참조(reference) 표준**으로 쓸 값어치가 충분하다 — CVPR 2025 발표, 촘촘한 구조 어노테이션, 꾸준한 리더보드 관리, 업계 채택까지 다 갖췄다. 다만 **상용 성능 보증의 절대 기준으로 이것만 쓰기엔 한계**가 있다. 형식이 다르다고 깎는 점수, 포화, 단일 정답 고정, 언어 바이어스 탓에 "OmniDocBench 96점"이 곧 "우리 도메인에서 최고"는 아니다. 평가 지표가 성능을 100% 투명하게 비춰주지 않는다는 건 RAGAS로 RAG 평가 방법론을 정리한 [글]({{site.baseurl}}/dev/2026/08/29/ragas-evaluation-methodology.html)에서 이미 겪은 일이다. 벤치마크는 **후보를 걸러내는 필터**로 쓰고, 최종 결판은 **실제 데이터 검증**에서 내야 한다.

## 참고

- [OmniDocBench GitHub 저장소](https://github.com/opendatalab/OmniDocBench){:target="_blank"}
- [OmniDocBench 논문 (arXiv:2412.07626)](https://arxiv.org/abs/2412.07626){:target="_blank"}
- [OmniDocBench is Saturated, What's Next for OCR Benchmarks? (LlamaIndex)](https://www.llamaindex.ai/blog/omnidocbench-is-saturated-what-s-next-for-ocr-benchmarks){:target="_blank"}
- [RAGAS 평가 방법론 — RAG 시스템 정량 검증 가이드]({{site.baseurl}}/dev/2026/08/29/ragas-evaluation-methodology.html)