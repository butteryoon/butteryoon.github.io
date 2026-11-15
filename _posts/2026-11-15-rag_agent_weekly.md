---
layout: post
comments: true
title: "에이전트 메모리에 원본 기록을 남겨야 하는 이유 — 에이전트·RAG 실무 메모 (11/08~11/15)"
description: "교훈만 뽑아 저장한 에이전트 메모리는 틀린 교훈을 걸러낼 방법이 없다. 이번 주 arXiv에서 고른 일곱 편으로 메모리의 원본 연결, 실패 로그에서 스킬 규칙 뽑기, 분류 체계 구축, 생성 중 조기 경보 평가, Text-to-SQL 테이블 검색, LLM 리랭킹 예산에서 바로 바꿀 것을 정리한다."
img: rag_weekly_20261115_title.webp
date: 2026-11-15 17:30:00 +0900
last_modified_at: 2026-11-15 17:30:00 +0900
tags: [rag, ai agent, llm, agent-memory, skills, evaluation, text-to-sql, reranking, arxiv, weekly, rag-weekly]
related: rag-weekly
categories: dev
---

Hermes 에이전트가 배치 작업으로 수집한 RAG·AI 에이전트 분야 주간 논문 보고서(arXiv cs.IR/cs.CL/cs.MA/cs.AI/cs.LG, 2026-11-08~11-15)에서 일곱 편을 골랐다. 기준은 지난주와 같다. 읽고 나서 내 파이프라인에서 오늘 바꿀 게 있는가. 다섯 편은 초록에 코드·모델 링크가 있고 작성 시점에 저장소가 모두 열린다.

<!--more-->

> **TL;DR:** ① 에이전트 메모리에 교훈만 저장하지 말고 그 교훈이 나온 원본 궤적을 링크로 달아 둔다. 실패한 과제에서 나온 교훈은 쓰기 전에 원본을 확인하게 한다. ② 대화 원문은 추출한 메모리와 따로 검색할 수 있게 남기고 검색 결과는 질의 의도에 맞춰 조립해서 넘긴다. ③ 실패한 대화에서 뽑은 규칙은 재시험을 통과한 것만 스킬에 넣는다. ④ 후보 목록이 정해진 분류 체계를 LLM으로 만든다면 자유 생성 대신 편집 도구를 쥐여 준다. 비용은 늘어난다. ⑤ 생성 중 모니터는 AUROC가 아니라 경보가 언제 울렸는지로 평가한다. ⑥ Text-to-SQL의 테이블 검색은 생성형 LLM 없이도 된다. ⑦ LLM 리랭킹은 전부 할지 말지 고르지 말고 몇 건에 쓸지를 예산으로 정한다.

수집과 서지 대조는 보고서 단계에서 끝났다. 후보 442편 가운데 코드·모델 공개나 수치 근거가 있는 50편이 남았고 그중 20편이 표에 올랐다. 아래 일곱 편을 고른 것과 "무엇을 바꿀 것인가"는 arXiv 초록을 직접 읽고 판단했다. 벡터 인덱스 내부의 탐색 종료 규칙과 디스크 페이지 배치, 양자화, 소프트 컨텍스트 압축, 추천 서빙, 교통 신호 강화학습, 로봇 내비게이션처럼 에이전트·RAG 개발 환경에서 바로 손댈 게 없는 행은 뺐다.

## 1. 받아서 바로 돌려 볼 것

### 메모리의 교훈에는 원본 궤적을 달아 둔다

외부 메모리를 쓰는 에이전트는 지난 과제의 궤적을 짧은 교훈으로 정리해 새 과제에 가져다 쓴다. 그런데 메모리가 오히려 해가 되는 경우가 있다. **WikiTrace** 저자들은 이를 "메모리 유발 성능 저하"라고 부른다. 메모리가 없었으면 풀었을 과제를 틀린 교훈을 따르거나 맞는 교훈을 잘못 적용해서 놓치는 현상이다. 원인은 교훈이 저장되는 방식에 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"This is because lessons are inferred from trajectories labeled only with task success or failure, yet are typically provided without the trajectories they came from, leaving the agent no way to check them."</blockquote></details>

WikiTrace는 교훈을 세 층으로 저장한다. 짧은 통찰, 그 궤적이 왜 성공하거나 실패했는지에 대한 요약, 그리고 원본 궤적이다. 새 과제에서 에이전트는 통찰 전체를 들고 시작한다. 통찰에 행동에 필요한 세부가 빠져 있거나, 다른 통찰과 충돌하거나, 실패한 과제에서 나온 것이면 터미널 도구로 링크를 따라 내려가 세부를 채우거나 통찰을 고친 뒤 행동한다. 에이전트 벤치마크 다섯 개에서 기존 메모리 방식과 같은 통찰만 받은 기준선을 모두 앞섰고 메모리 유발 성능 저하도 줄었다.

**바꿀 것**: 에이전트 메모리에 "이럴 땐 이렇게 하라" 식의 교훈 문장만 쌓고 있다면, 각 교훈에 원본 궤적 파일과 성공·실패 요약으로 가는 링크를 붙인다. 궤적을 요약한 뒤 지우는 구성이라면 지우지 않고 보관한다. 프롬프트에는 교훈이 세부가 부족하거나 서로 충돌하거나 실패 사례에서 나온 경우 원본을 열어 확인하라는 조건을 넣는다. 메모리를 켠 뒤 오히려 실패하게 된 과제가 있는지도 메모리를 끈 실행과 비교해 본다.

- [arXiv 2611.14320](https://arxiv.org/abs/2611.14320v1){:target="_blank"} · [yukyeongleee/wikitrace](https://github.com/yukyeongleee/wikitrace){:target="_blank"}

### 대화 원문을 남기고, 검색 결과는 질의 의도에 맞춰 조립한다

**MAPE**는 메모리 설계를 사람이 짜지 않고 프로그램 진화로 찾는다. 최소한의 씨앗 프로그램에서 출발해 진단과 제안을 반복하며 메모리 프로그램 전체를 진화시킨다. LoCoMo와 LongMemEval에서 사람이 설계한 기준선보다 정확도가 절대값으로 최대 12% 높았고 서빙 비용은 35% 수준이었다.

실무에서 가져갈 것은 진화 프레임워크보다 거기서 반복해서 나온 설계 패턴 두 가지다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"retaining raw conversations as retrievable evidence preserves details omitted from extracted memories, while composing retrieved evidence according to query intent helps the answering model use it effectively."</blockquote></details>

두 번째 패턴인 조립 모듈을 기존 메모리 시스템 다섯 개에 이식했더니 다섯 개 모두 정확도가 올랐다. 탐색 목표에 서빙 비용을 함께 넣으면 계산이 옮겨 간다는 관찰도 있다. LLM으로 메모리를 구축하는 단계의 계산이 줄고 질의 시점의 선택적 처리가 늘어나는데 정확도는 경쟁력을 유지했다.

**바꿀 것**: 대화에서 사실만 추출해 메모리에 넣고 원문은 버리고 있다면 원문도 검색 가능한 증거로 따로 남긴다. 추출 과정에서 빠진 세부를 원문이 채워 준다. 검색 결과를 순위대로 이어 붙여 답변 모델에 넘기고 있다면 질의 의도에 따라 증거를 조립하는 단계를 하나 둔다. 구체적인 조립 방식은 초록에 없으니 공개 코드에서 확인한다. 메모리 구축에 LLM 호출을 많이 쓰고 있다면 그 일부를 질의 시점 처리로 옮겼을 때 비용과 정확도가 어떻게 변하는지도 재 볼 만하다.

- [arXiv 2611.17068](https://arxiv.org/abs/2611.17068v1){:target="_blank"} · [ash-ding/MAPE](https://github.com/ash-ding/MAPE){:target="_blank"}

### 실패한 대화에서 뽑은 규칙은 재시험을 통과한 것만 스킬에 넣는다

소형 오픈 모델은 사내 고객 응대 배포에 맞지만 프런티어 모델에 두 가지로 밀린다. 주어진 상황에서 문서화된 정책이 무엇을 요구하는지 놓치고 아는 것도 일관되게 실행하지 못한다. **Knowledge in the Harness, Reliability in the Weights**는 더 강한 교사 모델 없이 배포 로그와 그 참조 결과만으로 두 격차를 줄일 수 있는지 본다.

출발점은 Gemma 4 12B 에이전트다. 도메인 정책을 시스템 프롬프트가 아니라 필요할 때 불러오는 스킬에서 읽는다. 라운드마다 새 과제 묶음을 처리하고 기본 모델이 실패한 대화를 진단해 스킬에 규칙을 쓴다. 규칙이 그대로 들어가는 것은 아니다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"a rule survives only if it passes code checks, counterexample self-review, a paired retest, and a noise-calibrated validation gate."</blockquote></details>

그다음 강화학습(GRPO)으로 도구를 쓰는 다중 턴 대화 전체에 LoRA 어댑터를 학습한다. τ²-bench 리테일·항공 도메인에서 두 라운드 뒤 pass^1이 출발점보다 리테일 12.5점, 항공 19.5점 올랐고 pass^4는 18.4점, 22.0점 올랐다. 공식 과제로 학습하거나 선별하지 않은 결과다. 같은 모델에 정책 전체를 시스템 프롬프트로 넣은 벤치마크 참조 에이전트보다도 pass^1이 11.8점, 8.0점, pass^4가 22.8점, 16.0점 높았다. 2×2 절제 실험에서는 역할이 갈렸다. 규칙은 에이전트가 한 번도 풀지 못한 과제를 구해 냈고 가중치 학습은 이미 푸는 과제를 매번 풀게 만들었다. 규칙을 컨텍스트에 넣고 강화학습을 해도 규칙에 담긴 정책 지식이 가중치로 흡수되지는 않았다.

**바꿀 것**: 정책 문서 전체를 시스템 프롬프트에 넣고 있다면 상황별 스킬로 쪼개 필요할 때 읽게 하는 구성을 시험해 본다. 실패한 대화를 모델에게 진단시켜 스킬에 규칙을 추가하는 루프를 돌린다면, 새 규칙을 넣기 전과 후로 같은 과제를 다시 돌려 통과한 규칙만 남긴다. 한 번의 재시험 차이는 실행 간 잡음일 수 있으니 같은 설정을 반복 실행해 잡음 폭부터 재 둔다. 못 푸는 과제는 규칙으로, 풀다 말다 하는 과제는 학습으로 접근한다는 구분도 가져갈 만하다. 코드·데이터·실행 기록이 공개돼 있다.

- [arXiv 2611.15268](https://arxiv.org/abs/2611.15268v1){:target="_blank"} · [Brightendata/knowledge-in-the-harness](https://github.com/Brightendata/knowledge-in-the-harness){:target="_blank"}

### 분류 체계는 자유 생성 대신 편집 도구로 만들게 한다

분류 체계(taxonomy)는 시맨틱 검색, 상품 카탈로그 정리, RAG를 받치는 구조화된 지식이다. 정해진 후보 개체 집합으로 계층을 만들 때 LLM의 반복 자유 생성은 개체를 빠뜨리거나 없는 개체를 만들고, 잘못된 상하 관계를 넣고, 정리 원칙이 중간에 바뀌는 문제에 취약하다.

**TIGA**는 분류 체계 구축을 명시적 상태와 제한된 행동 공간 위의 반복 결정 과정으로 바꾼다. 실행 가능한 분류 체계 편집 도구, 도구 호출·근거·결과를 기록하는 메모리, 에이전트가 추론 중에 고쳐 쓰는 정리 정책을 묶었다. 선택 사항으로 전역 검토자가 완성된 체계를 후보 집합과 현재 정책에 비춰 평가한다. 벤치마크 분류 체계 네 개와 LLM 두 개로 평가했더니 대부분의 구성에서 구조 품질이 가장 좋았지만 모든 정밀도 지표에서 앞선 것은 아니었다. 대가도 초록에 적혀 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The gains are accompanied by substantially higher token consumption, API-call volume, and inference time."</blockquote></details>

절제 실험에서 정책과 상호작용 메모리는 일관되게 도움이 됐고 전역 검토의 효과는 데이터셋과 수정 예산에 따라 달랐다.

**바꿀 것**: 상품·문서 카테고리 트리를 LLM에게 통째로 생성시키고 있다면 노드 추가·이동 같은 편집 도구만 호출하게 하고 후보 개체가 빠짐없이 한 번씩 들어갔는지는 코드로 검사한다. 정리 원칙은 프롬프트에 고정하지 말고 별도 정책 문서로 두어 에이전트가 고치면 기록이 남게 한다. 전역 검토 단계는 효과가 데이터에 따라 달랐으니 먼저 빼고 시작해도 된다. 토큰과 호출 수가 크게 늘어난다는 점은 감안해서 한 번 만들고 오래 쓰는 체계에 적용한다.

- [arXiv 2611.13966](https://arxiv.org/abs/2611.13966v1){:target="_blank"} · [bschaarmann/TIGA](https://github.com/bschaarmann/TIGA){:target="_blank"}

### 생성 중 모니터는 경보가 언제 울렸는지로 평가한다

언어 모델의 실패는 특정 토큰에서 시작되는 경우가 많다. 저자들은 그 지점을 onset이라고 부른다. 생성 도중에 모델을 읽는 모니터는 세 가지 일을 할 수 있다. 끝난 응답을 분류하거나, 실패가 진행 중일 때 탐지하거나, 시작되기 전에 경고한다. 생성을 멈추거나 방향을 돌릴 시간이 남는 것은 마지막뿐이다. 그런데 모니터는 보통 AUROC 같은 응답 단위 지표로 평가되고 이 지표는 경보가 언제 울렸는지를 보지 않는다.

**PREWARN**은 onset이 주석된 실패라면 어디에나 쓸 수 있는 평가 절차다. 검증 데이터에서 용도에 맞게 고른 오류율로 모니터의 임계값을 보정하고 모니터를 돌린 뒤 경보를 onset 기준으로 채점한다. 사례 연구는 오픈 모델 세 개의 반복 퇴화(같은 내용을 되풀이하는 실패)다. 활성값 위의 선형 프로브와 학습 없는 기준선 여섯 개를 비교했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Response-level metrics saturate on this failure, and a rule that reads only the response length scores almost perfectly, while probes that AUROC ranks as equal warn very differently under PREWARN."</blockquote></details>

프로브는 세 모델 중 두 모델에서 퇴화한 응답 대부분에 onset 전에 경고했다. 학습 없는 기준선 가운데 무작위 경보보다 훨씬 자주 경고한 것은 없었다.

**바꿀 것**: 스트리밍 중에 반복 루프나 이상 출력을 잡아 생성을 끊는 모니터가 있다면, 응답 단위 AUROC만 보고 고르지 않는다. 실패 사례에 시작 토큰 위치를 표시해 두고 경보가 그보다 앞에서 울린 비율을 따로 잰다. 응답 길이만 보는 규칙을 기준선으로 함께 돌려 본다. 그 규칙이 응답 단위 지표에서 거의 만점이 나온다면 그 지표로는 모니터를 가릴 수 없다. 오픈 모델을 직접 서빙한다면 보정된 임계값과 함께 공개된 프로브를 받아 볼 수 있다.

- [arXiv 2611.16803](https://arxiv.org/abs/2611.16803v1){:target="_blank"} · [HF 프로브 컬렉션](https://huggingface.co/collections/marcodene/prewarn-degeneration-probes){:target="_blank"} · [swiss-ai/degeneration-probe](https://github.com/swiss-ai/degeneration-probe){:target="_blank"}

## 2. 코드는 없지만 파이프라인에서 바로 점검할 것

### Text-to-SQL의 테이블 검색은 생성형 LLM 없이도 된다

여러 데이터베이스를 합친 큰 카탈로그 위의 Text-to-SQL은 질문에 필요한 테이블부터 찾아야 한다. **FERN**(Feature-Engineered Ranking with No generative LLM)의 비교 대상은 RASL의 공개 재현율이다. RASL은 Claude 3.5 모델로 키워드를 뽑고 테이블을 고른다. FERN은 고정 검색기 세 개의 결과를 합쳐 후보 100개를 만들고 후보마다 최대 120개 특성을 붙인 뒤 BIRD 학습 질문으로 한 번 학습한 LightGBM 랭커로 순서를 다시 매긴다.

학습에 쓰지 않은 Spider에서 후보 풀만으로 테이블 15개 기준 RASL의 공개 재현율(98.0)에 닿았다. 랭커는 정답 테이블이 모두 상위 5개 안에 든 질문 비율(complete@5)을 83.9%에서 95.6%로 올렸다. 테이블 5개 기준 재현율은 Spider에서 96.7 대 97.0, 힌트(evidence)가 있는 BIRD에서 96.6 대 97.5로 RASL과 1점 이내였다. 약한 곳도 있다. 힌트 없는 BIRD에서는 8.2점 낮았고(89.3), Fiben에서는 테이블 15개 기준 54.2 대 69.2로 크게 못 미쳤다(질문 세트가 다르다). 실제 BEAVER 질문 121개에서는 complete@15가 BEAVER로 학습한 검색기 둘보다 낮았다(49.6 대 54.5, 62.0). 지연은 이렇다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"On BIRD dev, median latency is 208 ms per question on one A100, without hosted model calls."</blockquote></details>

**바꿀 것**: 테이블이 많은 카탈로그에서 스키마 선택을 LLM 호출로 하고 있다면 검색기 여러 개의 결과를 합친 후보 풀과 가벼운 학습 랭커 조합을 같은 질문으로 비교해 본다. 평가는 재현율과 함께 **필요한 테이블이 전부 상위 k개 안에 들었는지**(complete@k)로 한다. 테이블 하나만 빠져도 SQL은 틀린다. BIRD dev에서는 검색 점수 외 특성이 complete@5를 7.7점 올렸으니 랭커에 검색 점수만 넣지 않는다. 다만 Fiben 결과처럼 도메인이 달라지면 크게 떨어질 수 있다. 내 카탈로그로 재 보기 전에 LLM 단계를 걷어 내지는 않는다.

- [arXiv 2611.14477](https://arxiv.org/abs/2611.14477v1){:target="_blank"}. 초록에 공개 자원 링크는 없다

### LLM 리랭킹은 몇 건에 쓸지를 예산으로 정한다

키워드 매칭 기반 어휘 검색기는 빠르고 에너지를 적게 쓴다. LLM은 길고 맥락이 많은 질의를 잘 해석하지만 계산 비용이 크다. **LAAM**(LLM Active Approximation Model)은 둘 사이에 가벼운 학습 예측기를 둔다. LLM이 소수의 문서에만 라벨을 달고 능동 학습 루프가 그 예측기를 다듬는다. LLM 평가는 정보량이 큰 문서에 몰아 쓴다.

TREC Robust 2004에서 Lucene BM25와 DeepSeek v3로 질의 장황도 세 수준을 평가했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"For verbose queries, LAAM achieves 87% of average P@10 achieved by DeepSeek's reranking (0.459 vs. 0.526), while requiring only 3.3% of the per-query energy consumption."</blockquote></details>

통계 검정에서 BM25보다는 유의하게 좋았다. 저자들은 값싼 어휘 검색과 전수 LLM 리랭킹 중 하나를 고르는 대신 평가 예산으로 정확도를 직접 조절하는 연속적인 운영점을 얻는다고 정리한다.

**바꿀 것**: 긴 질의에 대해 상위 후보 전부를 LLM으로 리랭킹하고 있다면, LLM에 보내는 문서 수를 줄여 가며 P@10이 어디서부터 꺾이는지 곡선을 그려 본다. LLM 판정을 로그로 남겨 두면 그 판정을 흉내 내는 가벼운 예측기의 학습 데이터가 된다. 실험은 말뭉치 하나와 LLM 하나에서 이뤄졌고 P@10 격차(0.459 대 0.526)가 남는다는 점은 감안한다.

- [arXiv 2611.16488](https://arxiv.org/abs/2611.16488v1){:target="_blank"}. 초록에 공개 자원 링크는 없다

## 이번 주에 손댈 것

일곱 편 중 세 편이 같은 곳을 가리킨다. **요약본만 남기고 원본을 버리면 나중에 확인할 길이 없다**는 것이다. 교훈에는 그것이 나온 궤적이 붙어 있어야 하고, 추출한 메모리 옆에는 대화 원문이 있어야 하고, 실패 로그에서 뽑은 규칙은 다시 돌려 본 뒤에 넣어야 한다. 당장 할 수 있는 건 셋이다.

**하나.** 에이전트 메모리의 교훈·사실 항목마다 원본 궤적이나 대화 원문으로 가는 링크를 붙이고 원본을 지우는 정리 작업이 있다면 멈춘다.

**둘.** 메모리나 스킬 규칙을 켠 실행과 끈 실행을 같은 과제로 비교해 켠 뒤에 새로 실패하는 과제를 찾는다.

**셋.** 모델이 제안한 스킬 규칙은 추가 전후 재시험을 통과해야 들어가도록 관문을 만든다.

분류 체계 구축, 생성 중 모니터, Text-to-SQL 테이블 검색, LLM 리랭킹은 그 기능을 쓰는 환경에서만 해당한다. 쓰고 있다면 각 절의 "바꿀 것"에서 측정 하나씩만 먼저 걸어 둔다.

## 참고

- [Don't Just Trust Your Memory: Evidence-Linked Memory Reduces Memory-Induced Degradation in LLM Agents](https://arxiv.org/abs/2611.14320v1){:target="_blank"}
- [Discovering Effective Agent Memory Designs through Program Evolution](https://arxiv.org/abs/2611.17068v1){:target="_blank"}
- [Knowledge in the Harness, Reliability in the Weights: Teacher-Free Co-Evolution of an LLM Agent](https://arxiv.org/abs/2611.15268v1){:target="_blank"}
- [Controlled Knowledge Engineering for Taxonomy Induction with Tool-Grounded LLM Agents and Runtime-Adaptive Policies](https://arxiv.org/abs/2611.13966v1){:target="_blank"}
- [Detection Is Not Early Warning: Evaluating Token-Level Alarms on Language Model Failures](https://arxiv.org/abs/2611.16803v1){:target="_blank"}
- [FERN: Cross-Benchmark Table Retrieval for Text-to-SQL with Frozen Retrievers and Learned Ranking](https://arxiv.org/abs/2611.14477v1){:target="_blank"}
- [Active Learning of Approximate LLM Ranking for Energy-Efficient Retrieval of Long Queries](https://arxiv.org/abs/2611.16488v1){:target="_blank"}
