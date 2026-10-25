---
layout: post
comments: true
title: "반복 검색을 붙이면 '모른다'가 사라진다 — 에이전트·RAG 실무 메모 (10/18~10/25)"
description: "반복 RAG를 붙이면 정확도는 그대로인데 답이 없는 질문에 '모른다'고 하는 비율이 떨어진다. 온도 0 LLM 심판을 그대로 다시 돌려도 판정의 21.2%가 바뀐다. 이번 주 arXiv에서 고른 일곱 편으로 검색·평가·자기 개선 에이전트에서 바로 바꿀 것을 정리한다."
img: ai_abstract_title.jpg
date: 2026-10-25 22:15:00 +0900
last_modified_at: 2026-10-25 22:15:00 +0900
tags: [rag, ai agent, llm, embedding, evaluation, llm-as-judge, abstention, arxiv, weekly, rag-weekly]
related: rag-weekly
categories: dev
---

Hermes 에이전트가 배치 작업으로 수집한 RAG·AI 에이전트 분야 주간 논문 보고서(arXiv cs.IR/cs.CL/cs.MA/cs.AI/cs.LG, 2026-10-18~25)에서 일곱 편을 골랐다. 고른 기준은 하나다. 읽고 나서 내 파이프라인에서 오늘 바꿀 게 있는가. 일곱 편 모두 초록에 코드나 데이터 링크가 있고 그중 여섯 편은 작성 시점에 저장소가 실제로 열린다.

<!--more-->

> **TL;DR:** ① 영어 코퍼스에서 dense와 late-interaction을 따로 돌리고 있다면, 인덱스 하나로 합치는 179M 모델이 나왔다. ② 좁은 기술 도메인에서는 상용 API 임베딩이 BM25에 크게 밀릴 수 있고 작은 개발 컬렉션에서 고른 설정은 코퍼스가 커지면 뒤집힌다. ③ 다중 사실 질문에서는 "관련 청크 하나"가 아니라 "필요한 청크 전부"를 가져왔는지 재야 한다. ④ 반복 검색·쿼리 재작성·리랭킹을 붙이면 정확도는 그대로인데 "모른다"고 답하는 비율이 떨어진다. ⑤ LLM 심판은 같은 설정을 다시 돌리기만 해도 판정이 21.2% 바뀐다. ⑥ 자기 개선 에이전트에는 스스로 고칠 수 없는 핵심을 코드로 박아 둔다.

수집과 서지 대조는 보고서 단계에서 끝났다. arXiv API로 받은 후보 475편 가운데 코드·모델 공개나 수치 근거가 있는 65편이 남았고 선정분은 초록 페이지의 `citation_title`과 맞춰 봤다. 아래 일곱 편을 고른 것과 "무엇을 바꿀 것인가"는 초록을 직접 읽고 판단했다.

## 1. 받아서 바로 돌려 볼 것 — 임베딩과 코퍼스

### dense 인덱스와 late-interaction 인덱스를 하나로

dense bi-encoder와 late-interaction(ColBERT 계열) 검색기는 서로 다른 질문에서 틀린다. 벡터 하나는 글이 무엇에 관한 것인지를 잡고 토큰 단위 매칭은 정확한 개체와 구절을 잡는다. 그래서 둘을 같이 쓰려면 모델 둘, 인덱스 둘에 점수 융합 단계까지 붙여야 했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Using both today means two models, two indexes and a score-fusion step."</blockquote></details>

**UIBK-Embedding-v1**은 179M 파라미터 영어 인코더 하나로 두 표현을 한 번에 뽑는다. dense 임베딩을 토큰 벡터 목록에 벡터 세 개로 끼워 넣고 1차원 태그로 두 종류가 서로 매칭되지 않게 막는다. 그러면 평범한 MaxSim이 곧 "토큰 MaxSim + 가중 dense 코사인"이 된다. 기존 late-interaction 인덱스에 손대지 않고 그대로 올라가며 저자들은 PLAID로 확인했다. BEIR nDCG@10은 59.79로, 같은 체급 최강인 LateOn보다 2.57점 높고 15개 데이터셋 전부에서 앞섰다.

**바꿀 것**: 영어 문서에 dense와 late-interaction을 따로 돌리고 융합하고 있다면, 이 모델로 PLAID 인덱스 하나를 만들어 기존 융합 결과와 나란히 비교해 본다. 인덱스와 융합 코드가 하나씩 줄어든다. 다만 **영어 인코더**다. 한국어 코퍼스라면 이 모델 자체보다 "dense 벡터를 토큰 벡터 목록에 끼워 넣는다"는 구조를 참고하는 정도가 맞다.

- [arXiv 2610.27195](https://arxiv.org/abs/2610.27195v1){:target="_blank"} · [DataScienceUIBK/uibk-embedding](https://github.com/DataScienceUIBK/uibk-embedding){:target="_blank"} · [HF 모델](https://huggingface.co/DataScience-UIBK/uibk-embedding-v1){:target="_blank"}

### 좁은 도메인에서는 API 임베딩부터 의심하라

**FormBench**는 배합 특허 문서에서 화학 정보를 정밀하게 찾는 검색 벤치마크다. 질의 55,352개에 약 100만 패시지짜리 코퍼스 두 벌을 붙였는데, C0은 방해 문서를 무작위로 뽑았고 C1은 구조가 비슷한 문서를 방해 문서로 넣었다.

결과가 극단적이다. 구조가 비슷한 방해 문서가 섞인 C1에서 OpenAI-3-large는 **0.009**(C0에서는 0.213)를 받았다. BM25는 C1 0.375, C0 0.467로 버텼고 이 벤치마크 트리플렛으로 추가 학습(TAPT)한 Qwen3-Embed만 BM25를 넘었다(C1 0.4085, C0 0.4835). 저자들은 API 실패의 원인을 도메인 안에서 임베딩 공간이 한데 뭉쳐 버리는 현상(within-domain geometric collapse)으로 짚었다. 그런데 분류와 군집화에서는 API가 TAPT 변형보다도 나았다. 군집화를 잘한다고 정밀 검색까지 잘하는 건 아니다.

**바꿀 것**: 특허·사양서처럼 용어가 비슷한 문서가 빽빽한 코퍼스에서 API 임베딩만 쓰고 있다면, **BM25 단독 결과를 같은 질의에 한 번 돌려** 비교한다. 비교 세트를 만들 때는 무작위 방해 문서 말고 구조가 비슷한 문서를 섞어야 이 차이가 드러난다.

- [arXiv 2610.30175](https://arxiv.org/abs/2610.30175v1){:target="_blank"}. 초록에 적힌 `olivettigroup/formbench` 저장소는 **작성 시점(10/25)에 404**이고 HF 데이터셋 `vinven7/FormBench`도 로그인 없이는 열리지 않는다(401). 당장은 설계만 참고할 수 있다

### 작은 컬렉션에서 고른 설정은 커지면 뒤집힌다

생의학 RAG를 대상으로 한 통제 실험이다. 정답 문서는 그대로 두고 검색 컬렉션만 점점 키우면서 초록·평문 전문·계층형 전문 세 가지 표현을 비교했다. 결론은 둘이다. 작은 컬렉션에서 보이던 성능 패턴이 컬렉션이 커지면 일관되게 유지되지 않았고 리랭킹이 도움이 되는지도 코퍼스와 검색 설정에 따라 달랐다. 계층형 표현은 대부분의 종단 비교에서 평문 전문과 같거나 나았는데, 인덱싱 단위는 67% 적었고 저장 공간은 44~56% 덜 들었다.

**바꿀 것**: 청킹·리랭커 설정을 몇천 건짜리 개발 컬렉션에서 골랐다면, **정답 문서는 고정하고 운영 규모에 가깝게 컬렉션을 불린 뒤** 다시 비교한다. 전문을 인덱싱하고 있다면 계층형 표현을 후보에 올린다. 생의학 도메인에서 나온 결과라 다른 도메인에서도 같은 방향일지는 직접 확인해야 한다.

- [arXiv 2610.26339](https://arxiv.org/abs/2610.26339v1){:target="_blank"} · [hitz-zentroa/biomedical-rag-scaling](https://github.com/hitz-zentroa/biomedical-rag-scaling){:target="_blank"}

## 2. RAG 평가 — 맞혔는데 근거가 빠졌거나, 모르는데 답했거나

### "관련 청크 하나"가 아니라 "필요한 청크 전부"

이 논문은 답과 함께 **줄 단위 인용**을 내게 하는 evidence-based generation(EBG)을 정의하고 정답 여부와 근거를 한꺼번에 채점하는 Ans+Evid 지표를 만들었다. 프런티어 모델 열 개와 오픈 웨이트 모델 세 개를 돌려 보니, 긴 컨텍스트 모델은 피하는데 RAG만 걸리는 한계가 둘 나왔다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"similarity retrieval surfaces any relevant chunk far more reliably than all required chunks for multi-fact queries, and chunking destroys document meta-information needed to resolve identity-bearing queries."</blockquote></details>

하나는 유사도 검색이 관련 청크를 *하나쯤* 가져오는 데는 강하지만 다중 사실 질문에 *필요한 청크 전부*를 가져오는 데는 훨씬 약하다는 것이다. 다른 하나는 청킹이 문서 메타정보를 날려서 "어느 문서·누구의 것이냐"를 가려야 하는 질문을 풀지 못한다는 것이다. 답만 채점하면 이런 실패는 보이지 않는다. 바늘이 둘인 설정에서 인용이 빠지는 경우가 그렇고 컨텍스트가 모델의 사전 지식과 충돌할 때 Ans와 Ans+Evid 사이 격차가 벌어지는 경우도 그렇다.

**바꿀 것**: 검색 평가가 hit@k처럼 "관련 청크가 하나라도 들어왔나"만 보고 있다면, 다중 사실 질문에는 **필요한 청크가 전부 들어왔는지**를 따로 센다. 청크마다 출처 문서 제목 같은 식별 정보가 남아 있는지도 확인한다. 답변 평가에는 인용을 요구하고 답과 근거를 같이 채점한다.

- [arXiv 2610.26951](https://arxiv.org/abs/2610.26951v1){:target="_blank"} · [debobanerjee/LC-EBG](https://github.com/debobanerjee/LC-EBG){:target="_blank"}

### 반복 검색을 붙이면 "모른다"가 줄어든다

**DeclineCraft**는 답할 수 없는 질문을 일부러 만든다. 코퍼스 기준일 이후에 추가된 마인크래프트 기능을 묻는 식이다. 27개 LLM을 놓고 답이 없는 질문에서 "모른다"고 인정한 비율(CAR), 답할 수 있는 질문에 시도한 비율, 정확도를 쟀다.

단발 검색을 붙이면 검색이 없을 때보다 CAR이 40개 구성 중 38개에서 유의하게 올랐다. 여기까지는 예상대로다. 문제는 그다음이다. 모델이 컨텍스트를 스스로 판단해 검색을 최대 두 번 더 요청하는 **반복 RAG**는 단발 RAG보다 CAR을 40개 중 30개에서 유의하게 **떨어뜨렸다.** 정확도는 그사이 거의 움직이지 않았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Iterative RAG, in which the model self-assesses its context and can request up to two further retrieval rounds, significantly lowers CAR versus single-shot RAG in 30 of 40 configurations (mean 46.4%), with little change in Answer Accuracy (68.5% to 69.3%)."</blockquote></details>

하이브리드(dense+sparse) 검색에 쿼리 재작성과 리랭킹을 더했을 때도 시험한 12개 모델 전부에서 CAR이 떨어졌다. 검색 없이 같은 가중치에서 thinking을 켜면 13개 중 8개 모델에서 CAR이 평균 14%p 내려갔고 응답이 "불확실하다"에서 "질문의 전제가 틀렸다"로 바뀌었다. 모델 크기와 출시 시점은 정확도와는 상관이 있었지만 CAR과는 유의한 관계가 없었다.

**바꿀 것**: 반복 검색·쿼리 재작성·리랭킹을 붙이거나 thinking을 켜는 변경은 **정확도만 보고 통과시키지 않는다.** 회귀 세트에 답이 없는 질문 묶음을 넣고 "모른다"고 답한 비율을 별도 지표로 기록한다. 이번 결과대로라면 정확도는 그대로인데 이 지표만 떨어진다. 더 큰 모델로 바꾼다고 이 문제가 풀린다고 기대하지도 않는다.

- [arXiv 2610.28340](https://arxiv.org/abs/2610.28340v1){:target="_blank"} · [Thakur-Atharva/declinecraft-benchmark](https://github.com/Thakur-Atharva/declinecraft-benchmark){:target="_blank"}

## 3. LLM 심판 — 집계 점수가 같아도 판정은 바뀐다

Arena-Hard v2.0 프롬프트 500개에 대해 두 모델의 답을 고정해 두고 GPT-4.1 심판을 기준·출력 형식·과제 프레이밍 조합 여덟 가지로 돌렸다. 각 설정은 버전을 고정한 모델에 온도 0으로 **그대로 한 번 더** 돌렸다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Rerunning the default setup moves the aggregate score by about one point but changes the win/tie/loss outcome on 21.2% of prompts."</blockquote></details>

기본 설정을 다시 돌리면 집계 점수는 1점 남짓 움직이는데, 프롬프트별 승·무·패는 21.2%가 바뀐다. 지시문을 바꾸면 더 흔들린다. 추론 없이 판정만 내게 하고 다른 지시도 몇 개 뺀 설정에서는, 같은 설정을 다시 돌렸을 때의 평균보다 판정이 28.2%p 더 많은 프롬프트에서 바뀌었고 집계 점수는 4.09점 떨어졌다. 반대로 정확성을 우선하도록 기준을 바꾼 설정에서는 판정이 6.7%p 더 많이 바뀌었는데도 집계 점수는 거의 그대로였다.

**바꿀 것**: LLM 심판의 개별 판정을 회귀 게이트나 선호 라벨로 쓰고 있다면, **같은 설정을 최소 두 번 돌리고 프롬프트 단위 일치율을 집계 점수와 함께 기록한다.** 저자들이 내린 권고도 이것이다. 일치율이 낮은 설정이라면 개별 판정 하나로 어떤 응답을 떨어뜨리는 게이트는 다시 생각해야 한다.

- [arXiv 2610.30512](https://arxiv.org/abs/2610.30512v1){:target="_blank"} · [Yorkel/llm-judge-release](https://github.com/Yorkel/llm-judge-release){:target="_blank"}

## 4. 자기 개선 에이전트 — 스스로 못 고치는 핵심을 코드로 박아라

**Steward**는 실시간 웹 데이터로 투자하는 9개 역할의 멀티 에이전트 시스템이다. 투자 자체보다 볼 만한 건 **자기 개선의 경계를 어떻게 그었나**다. 매일 밤 코드가 모든 거래와 예측을 채점해 메모리에 쓴다. 분기마다 reflector가 프롬프트 개선안을 내면, 결정적(deterministic) 게이트가 이를 되돌릴 수 있게 적용하고 기록된 입력을 재생해 검증한다. 주문 계산, 리스크 규칙, 규율 담당 에이전트의 프롬프트로 이루어진 "불변 핵심"은 어떤 자기 수정도 건드릴 수 없게 타입 시스템으로 막아 두었다.

5개월 실운영(거래일 102일, LLM 호출 1,632회) 동안 섹터 낙폭 24.6%를 손실 0.3%로 막았다. 기록된 입력을 재생한 실험에서는 주입된 지시를 하나도 따르지 않았다.

**바꿀 것**: 에이전트가 자기 프롬프트나 메모리를 고치는 구조라면, **자기 수정이 닿지 않을 영역(리스크 규칙, 감시 역할 프롬프트)을 목록으로 정하고 프롬프트가 아니라 코드로 막는다.** 학습된 변경은 바로 반영하지 말고 되돌릴 수 있는 게이트를 거치게 한다. 반영 전에는 지난 입력을 재생해서 확인한다.

- [arXiv 2610.30557](https://arxiv.org/abs/2610.30557v1){:target="_blank"} · [yebof/quant-agent](https://github.com/yebof/quant-agent){:target="_blank"}

## 이번 주에 손댈 것

일곱 편에서 하나만 뽑자면 **"집계 숫자가 그대로라고 안심하지 마라"**다. 반복 RAG는 정확도를 지키면서 "모른다"를 줄였고 LLM 심판은 집계 점수를 지키면서 판정 다섯 개 중 하나를 바꿨다. 당장 할 수 있는 건 셋이다.

**하나.** RAG 회귀 세트에 답이 없는 질문 묶음을 넣고 "모른다" 비율을 따로 센다. 검색 단계를 늘리는 변경이 이 숫자를 조용히 깎는다.

**둘.** LLM 심판 설정을 그대로 한 번 더 돌려 프롬프트 단위 일치율을 본다. 비용은 한 번 돌리는 값이고 개별 판정을 얼마나 믿어도 되는지가 바로 나온다.

**셋.** 다중 사실 질문에서 필요한 청크가 전부 들어왔는지 센다. hit@k만 보고 있었다면 여기서 새는 게 보일 것이다.

임베딩 교체와 코퍼스 규모 재평가는 인덱스를 다시 만들어야 해서 시간이 든다. 위 셋을 먼저 하고 영어 코퍼스에서 인덱스 두 개를 운영 중이라면 UIBK-Embedding 비교부터 붙인다.

## 참고

- [UIBK-Embedding: Effective Text Retrieval by Unifying Dense and Late-Interaction Retrieval](https://arxiv.org/abs/2610.27195v1){:target="_blank"}
- [FormBench: A Benchmark for Fine-Grained Retrieval in Formulation Patents](https://arxiv.org/abs/2610.30175v1){:target="_blank"} (저장소 404·데이터셋 비공개, 10/25 기준)
- [Large-Scale Biomedical RAG: Corpus Scaling, Document Representation, and Efficiency Trade-offs](https://arxiv.org/abs/2610.26339v1){:target="_blank"}
- [The Bottleneck Is Retrieval, Not Reading: Diagnosing Evidence-Based Generation with RAG for Long Context Question Answering](https://arxiv.org/abs/2610.26951v1){:target="_blank"}
- [DeclineCraft: A RAG Abstention Benchmark for Jointly Measuring Combined Retrieval and Parametric Knowledge](https://arxiv.org/abs/2610.28340v1){:target="_blank"}
- [Behind Aggregate Scores: What Systematically Varying Arena-Hard's LLM Judge Reveals](https://arxiv.org/abs/2610.30512v1){:target="_blank"}
- [Governed Self-Evolution for Autonomous Investing: A Multi-Agent LLM System with an Invariant Core](https://arxiv.org/abs/2610.30557v1){:target="_blank"}
