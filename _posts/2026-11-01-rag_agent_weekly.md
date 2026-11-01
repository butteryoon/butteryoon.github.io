---
layout: post
comments: true
title: "검색한 것과 읽히는 것은 다르다 — 에이전트·RAG 실무 메모 (10/25~11/01)"
description: "청크 대신 제목·섹션 경로·표 머리글을 붙인 근거 단위로 리더 컨텍스트를 다시 채우면 같은 토큰 예산에서 정답률이 5~6점 오른다. 이번 주 arXiv에서 고른 일곱 편으로 리더 입력, 에이전트 컨텍스트 관리, 코드 품질 기준선, llms.txt 입력 신뢰에서 바로 바꿀 것을 정리한다."
img: ai_abstract_title.jpg
date: 2026-11-01 20:05:00 +0900
last_modified_at: 2026-11-01 20:05:00 +0900
tags: [rag, ai agent, llm, chunking, context-management, evaluation, prompt-injection, arxiv, weekly, rag-weekly]
related: rag-weekly
categories: dev
---

Hermes 에이전트가 배치 작업으로 수집한 RAG·AI 에이전트 분야 주간 논문 보고서(arXiv cs.IR/cs.CL/cs.MA/cs.AI/cs.LG, 2026-10-25~11-01)에서 일곱 편을 골랐다. 기준은 지난주와 같다. 읽고 나서 내 파이프라인에서 오늘 바꿀 게 있는가. 다섯 편은 초록에 코드 링크가 있고 그중 세 편은 작성 시점에 저장소가 실제로 열린다.

<!--more-->

> **TL;DR:** ① 검색 단위와 리더가 읽는 단위를 분리하고 리더에 넘기는 조각마다 페이지 제목·섹션 경로·표 머리글을 붙인다. 인덱스와 검색기는 그대로 둔다. ② 검색된 출처끼리 충돌할 때 충돌 유형 분류는 LLM 생성 대신 확률을 내는 분류 모델에 맡기고 확신이 낮은 4분의 1만 LLM으로 넘긴다. ③ 다중 홉 질문은 개체 이름을 다 적은 하위 질문을 하나씩 만들고 그 답을 캐시해 재사용한다. ④ 장기 실행 에이전트에 "확인된 사실을 체크포인트하고 컨텍스트를 비우는" 도구를 준다. ⑤ 코드 생성 품질 기법은 같은 호출 예산의 best-of-n과 먼저 비교한다. ⑥ llms.txt 본문을 모델에 그대로 넣지 않는다.

수집과 서지 대조는 보고서 단계에서 끝났다. 후보 493편 가운데 코드·모델 공개나 수치 근거가 있는 59편이 남았고 그중 20편이 표에 올랐다. 아래 일곱 편을 고른 것과 "무엇을 바꿀 것인가"는 arXiv 초록을 직접 읽고 판단했다. 추천 시스템, 하드웨어 설계, MoE 사후 학습처럼 에이전트·RAG 개발 환경에서 바로 손댈 게 없는 행은 뺐다.

## 1. 받아서 바로 돌려 볼 것

### 리더에 넘기는 단위를 검색 단위와 떼어 놓는다

웹 RAG는 보통 페이지를 고정 길이 청크(시각 파이프라인이면 스크린숏 타일)로 자르고 그 조각 하나를 인덱싱·랭킹·읽기에 모두 쓴다. **Tessera** 저자들은 영어 위키백과 전체에서 정답 근거가 원본 HTML에서 리더 입력까지 가는 길을 추적했다. 원본에 있던 근거가 리더에 온전히 도착하지 못하는 경우가 많았고 특히 표와 인포박스가 그랬다. 청킹이 표를 그 의미를 정해 주는 머리글·맥락과 떼어 놓기 때문이다. 더 강한 추출기를 쓰거나 컨텍스트를 넓히거나 리더 예산을 늘려도 손실은 그대로였다.

Tessera는 학습이 필요 없는 근거 전달 계층이다. 랭킹 결과 뒤의 페이지를 HTML부터 다시 쪼개 잘게 나누고 조각마다 제목과 위치 정보를 붙여 리더 컨텍스트를 다시 채운다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Given the pages behind any ranked list, Tessera rebuilds their HTML into fine-grained evidence units, each completed with its page title, section path, and table headers, selects the units relevant to the query, and packs them into the reader context at an unchanged token budget."</blockquote></details>

인덱스·검색기·리더는 건드리지 않는다. 질의응답 벤치마크 세 개에서 텍스트·스크린숏 파이프라인 모두 정답률이 5~6점 올랐고 리더·검색기·예산을 바꿔도 같은 방향이었다. 다단계 에이전트 검색에서도 효과가 이어졌다.

**바꿀 것**: HTML이나 표가 많은 문서를 청크 그대로 리더에 넘기고 있다면, 검색은 지금처럼 하되 **리더에 넣기 직전에 원본 페이지로 돌아가 조각마다 페이지 제목·섹션 경로·표 머리글을 붙여** 다시 넣어 본다. 인덱스를 다시 만들 필요가 없어 실험 비용이 낮다. 평가는 같은 토큰 예산에서 한다.

- [arXiv 2610.37080](https://arxiv.org/abs/2610.37080v1){:target="_blank"} · [sarendis56/Tessera](https://github.com/sarendis56/Tessera){:target="_blank"}

### 출처가 서로 다를 때, 왜 다른지는 싼 분류기로 가린다

검색된 출처끼리 말이 다르면 이유에 따라 대응이 달라야 한다. 정답이 여럿인 질문인지, 논쟁적인 주제인지, 낡은 정보인지, 거짓 출처인지에 따라서다. 앞선 연구(Cattan et al., 2025)는 생성기에 충돌 유형을 알려 주면 응답이 약 24점 좋아지지만 그 유형을 맞히기는 프런티어 LLM에게도 어렵다고 보고했다.

이 논문은 텍스트를 생성하는 대신 고정된 선택지 위의 확률을 한 번의 forward pass로 돌려주는 분류형 결정 모델(decision model)을 같은 CONFLICTS 벤치마크·같은 프롬프트로 돌렸다. 정확도는 50.0~67.0%였고 11개 중 10개가 원 논문의 Gemini 2.5 Flash(61.3%)와 같거나 나았다. 과금 비용은 4~39배, 지연은 2.6~6.0배 낮았다. 최신 Gemini 3.8 Flash(70.0%)는 그중 8개보다 유의하게 정확했지만 결정 모델이 가장 확신하지 못하는 4분의 1만 넘기는 캐스케이드로 비용을 크게 줄일 수 있었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The current Gemini 3.8 Flash (70.0%) is significantly more accurate than 8 of them, but sending each decision model's least confident quarter of queries to it gives a cascade statistically tied with it for 8 of the 11, at 25--37% of its cost."</blockquote></details>

분류 체계를 예/아니오 질문 여러 개로 쪼개면 모든 모델에서 정확도가 떨어졌다. 출처 순서에 대한 민감도는 모델마다 크게 달랐다.

**바꿀 것**: RAG에서 출처 충돌을 처리하느라 LLM에게 "왜 다른지 판단하라"를 매번 생성시키고 있다면, **확률을 내는 분류 모델로 1차 판정하고 확신이 낮은 하위 25%만 LLM으로 넘기는** 캐스케이드를 시험한다. 유형 판정을 예/아니오 질문 여러 개로 쪼개는 방식은 피한다. 출처 순서를 바꿔 넣어 판정이 흔들리는지도 한 번 확인한다.

- [arXiv 2610.36915](https://arxiv.org/abs/2610.36915v1){:target="_blank"} · [Parv17k/decision-models-conflict-typing](https://github.com/Parv17k/decision-models-conflict-typing){:target="_blank"} (코드·프롬프트·모델 출력 전체 공개)

### 검색 에이전트의 판단은 타입 객체로 받고, 검증은 코드가 한다

**AgenticDS**는 수백만 건 규모 저장소에서 학습용 데이터셋을 찾는 문제를 다룬다. 질의에는 "무엇을 학습하려는가" 같은 의미 목표와 함께 과제·라이선스·학습 단계 같은 하드 제약이 붙는데, 텍스트 검색은 이런 제약을 강제하지 못한다. 자유 형식 tool-calling 에이전트는 유연하지만 데이터셋을 고른 모델이 스스로 "맞다"고 선언하고 제약이 실제로 지켜졌는지는 아무도 확인하지 않는다.

AgenticDS에서는 언어 모델의 모든 결정이 먼저 타입이 정해진 객체로 만들어지고 그다음에 결정적(deterministic) 코드가 그 객체를 소비한다. 질의를 타입이 있는 검색 계획으로 컴파일하고 허용된 결정적 연산자(하이브리드 검색, 범위를 제한한 그래프 확장)로 실행한 뒤, 출처에 근거한 증거를 검증하고 나서야 순위를 받아들인다. 검증에 실패하면 국소 수정만 허용하는 타입 진단을 낸다. 새 벤치마크 HuDS-Bench(질의 200개)에서 NDCG@10이 가장 강한 dense–sparse 하이브리드보다 25.7%, 상용 LLM 에이전트보다 18.9% 높았다.

**바꿀 것**: 에이전트가 라이선스·버전·권한 같은 하드 제약이 있는 검색을 하고 있다면, 모델이 결과를 고르고 스스로 "조건 충족"이라고 말하게 두지 않는다. **모델 출력은 스키마가 있는 계획 객체로 받고, 제약 충족 여부는 메타데이터를 읽는 코드가 판정한다.** 판정에 실패하면 전체 재시도가 아니라 실패한 부분만 고치게 한다.

- [arXiv 2610.39608](https://arxiv.org/abs/2610.39608v1){:target="_blank"} · [nju-websoft/AgenticDS](https://github.com/nju-websoft/AgenticDS){:target="_blank"}

## 2. 코드는 아직 안 열렸지만 구조는 가져올 것

### 다중 홉 질문: 자기완결형 하위 질문 하나씩, 답은 캐시

반복형 RAG는 증거 사슬을 한 단계씩 복원하지만 LLM에게 시키는 일이 너무 많다. 하위 질문 계획, 의존관계 결정, 답이 나올 때마다 질문 고쳐 쓰기, 무엇을 알게 됐는지 추적하기까지 맡긴다. **DeFT-RAG**는 이 일을 덜어 낸다. 원래 질문과 지금까지의 질문–답 쌍만 보고 **자기완결형 하위 질문을 한 번에 하나** 만든다. 하위 질문은 묻는 개체의 이름을 전부 적기 때문에 계획·의존관계 추적·재작성이 필요 없다. 무엇을 알게 됐는지는 규칙 기반으로 갱신하는 QA 표가 맡고 예/아니오 판단은 파인튜닝한 경량 결정 모델이 맡는다.

MuSiQue·2WikiMultiHopQA·HotpotQA에서 gpt-4o-mini와 Qwen3.6 두 백본 모두 13개 비교 방법 중 가장 정확했고 다른 반복형 방법보다 LLM 입력 토큰이 적었다. 같은 의미의 하위 질문이 다시 나오면 캐시한 답을 재사용한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"With a QA cache shared across 1,000 MuSiQue questions, 23.1% of sub-questions are answered from the cache, which reduces LLM calls by 12.9% without lowering accuracy."</blockquote></details>

**바꿀 것**: 다중 홉 질의를 LLM이 계획부터 추적까지 한 프롬프트로 처리하게 두고 있다면, 하위 질문 생성 프롬프트에 **"묻는 개체의 이름을 모두 적어라"** 를 넣고 지금까지의 질문–답 쌍을 코드가 관리하는 표로 넘긴다. 하위 질문과 답은 질의 사이에서 공유하는 캐시에 쌓는다. 같은 코퍼스에 비슷한 질문이 반복되는 사내 RAG라면 캐시 적중률부터 재 본다.

- [arXiv 2610.37826](https://arxiv.org/abs/2610.37826v1){:target="_blank"}. 초록에 적힌 `Dingyi-Kang/DeFT-RAG` 저장소는 **작성 시점(11/01)에 404**다

### 장기 실행 에이전트에 "체크포인트하고 비우기" 도구를 준다

장기 과제에서 에이전트가 흔들리는 이유로 저자들은 작업 기억의 과부하를 든다. 관련 없는 정보가 쌓이면 주의가 흐려지고 계획과 추론이 무너진다. **CORAL**은 이를 에이전트가 직접 호출하는 작업 기억 관리 도구 세트로 푼다. 에이전트가 진행 상황과 검증된 사실을 체크포인트하고 어지러워진 컨텍스트를 비운 뒤 체크포인트만 들고 새 에피소드를 시작한다. 체크포인트 품질은 다중 에피소드 강화학습(MARPO)으로 더 끌어올렸다.

GAIA에서 8B 모델로 평균 42.8%를 냈고 Level 2·3처럼 긴 과제에서 개선 폭이 컸다. 입력 토큰은 ReAct 에이전트보다 6~7배 적게 썼다. BrowseComp에서는 강한 상용 에이전트를 8.0점 끌어올렸다.

**바꿀 것**: 에이전트 루프에서 컨텍스트를 대화 이력째로 끝까지 끌고 가고 있다면, **"지금까지 확인된 사실과 남은 할 일을 저장하고 컨텍스트를 초기화한다"는 도구를 하나 추가**하고 초기화 뒤에는 시스템 프롬프트와 체크포인트만 넣어 재개한다. 강화학습 없이 도구만 붙여도 효과가 나는지는 초록만으로는 알 수 없으니, 긴 과제 몇 개로 입력 토큰과 성공률을 먼저 비교한다.

- [arXiv 2610.40350](https://arxiv.org/abs/2610.40350v1){:target="_blank"}. 초록에 적힌 `JIA-Lab-research/CORAL` 저장소는 **작성 시점(11/01)에 404**다

## 3. 평가와 입력 신뢰

### 코드 품질 기법은 best-of-n부터 이겨야 한다

LLM이 기능 정확성 벤치마크는 대부분 통과하지만 테스트를 통과했다는 사실은 코드가 읽기 쉬운지·효율적인지·안전한지에 대해 별로 말해 주지 않는다. 이 논문은 1차 연구 628편을 체계적으로 검토하고 기법들을 같은 연산 예산에서 다시 돌렸다. 대부분의 연구는 자기가 겨냥한 품질 차원만 쟀고 정확성 변화는 그대로이거나 보고되지 않았으며 자동 품질 신호 하나만 쓰면 그 신호가 쉽게 게이밍됐다.

재평가 결과는 이렇다. 호출 한 번짜리 "품질 신경 써라" 프롬프팅은 분석기가 잡는 이슈 수를 늘렸고 복잡도와 심판 점수는 거의 그대로였다. 같은 예산에서는 **분석기 가중치 랭커를 붙인 best-of-n 샘플링**이 프롬프팅·계획·자기 수정·리뷰어 에이전트·검색 보강과 같거나 더 나았다. 랭커가 직접 최적화하지 않는 지표에서도 그랬다. Gemma 4 31B에서 66개 기법을 직접 비교했더니 주 목표에서 best-of-n을 95% 구간 밖으로 이긴 기법은 하나뿐이었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"In the head-to-head, 1 of 66 techniques beats best-of-n on its primary target beyond the 95% interval, a rate consistent with chance."</blockquote></details>

**바꿀 것**: 코딩 에이전트에 리뷰어 에이전트나 자기 수정 루프를 붙이려 한다면, **같은 호출 수로 n개를 샘플링하고 정적 분석기 결과로 고르는 기준선**을 먼저 만들어 비교한다. 개선 여부는 그 기법이 쓰는 신호(예: 분석기 이슈 수)와 독립된 지표로 판단한다. 저자들의 권고도 예산을 맞춘 샘플링 기준선과 기법 자체 신호와 무관한 지표로 이득을 보고하라는 것이다.

- [arXiv 2610.38977](https://arxiv.org/abs/2610.38977v1){:target="_blank"}

### llms.txt 본문은 신뢰할 수 없는 입력이다

`/llms.txt`는 사이트가 AI 검색에 읽어 달라고 내놓는 매니페스트다. 이 논문은 먼저 그 효과를 따로 떼어 측정한 외부 근거가 없다는 점을 확인했다. 도구 15개를 대상으로 한 목적 표집 조사에서는 한 엔드포인트에 대한 45번의 시도에서 자동 요청이 관찰되지 않았고 로그 조사에서는 파일이 감지된 도메인의 97%가 2026년 5월 동안 기록된 요청을 하나도 받지 않았다.

보안 쪽 파일럿이 더 실무적이다. 8개 사이트 24개 과제를 다섯 조건에서 돌린 실제 모델 응답 120개를 고정된 자동 채점으로 봤더니, 적대적 매니페스트 본문을 받은 원시 응답 24개 중 21개가 공격자가 노린 대상을 채택했다(1개는 미판정). 매니페스트에서 URL만 넘긴 적대 조건에서는 확인된 채택이 없었다(미판정 1개). 다만 이 파일럿은 근거를 고정해 넣었을 때의 출력 변화만 쟀고 실시간 검색이나 외부 행동은 포함하지 않았다.

**바꿀 것**: 웹을 읽는 에이전트가 `/llms.txt`를 가져와 본문을 그대로 컨텍스트에 넣고 있다면, **본문은 버리고 URL 목록만 남기는** 쪽으로 바꾼다. 모델에 넣기 전 단계에 결정적 파서 검사를 둔다. 반대로 블로그·문서 사이트 운영자라면, 이 조사 기준으로는 llms.txt를 올려도 실제로 읽힐 가능성이 낮다는 점을 감안한다.

- [arXiv 2610.38361](https://arxiv.org/abs/2610.38361v1){:target="_blank"}

## 이번 주에 손댈 것

일곱 편을 하나로 묶으면 **"모델에게 넘기는 것과 모델이 판정하는 것을 줄여라"**다. Tessera는 리더에 넘기는 조각을 다시 짰다. DeFT-RAG와 AgenticDS는 상태 추적과 제약 검증을 코드로 옮겼고 CORAL은 컨텍스트를 비우는 시점을 에이전트에게 맡겼다. 당장 할 수 있는 건 셋이다.

**하나.** 리더 입력 직전에 조각마다 페이지 제목·섹션 경로·표 머리글을 붙인다. 인덱스를 다시 만들지 않아도 되는 가장 싼 실험이다.

**둘.** 리뷰어 에이전트나 자기 수정 루프를 평가할 때 같은 호출 수의 best-of-n 기준선을 옆에 둔다. 이 기준선을 못 넘는 단계는 지연과 비용만 늘린다.

**셋.** 에이전트가 웹에서 가져온 `/llms.txt` 본문을 컨텍스트에 넣고 있는지 확인한다. 넣고 있다면 URL만 남긴다.

출처 충돌 캐스케이드와 체크포인트 도구는 분류 모델 선택이나 루프 구조 변경이 필요해 시간이 조금 더 든다. 위 셋을 먼저 하고 다중 홉 질의가 많은 환경이라면 하위 질문 캐시부터 붙인다.

## 참고

- [Tessera: Structure-Completed Evidence Units for Web Retrieval-Augmented Generation](https://arxiv.org/abs/2610.37080v1){:target="_blank"}
- [Do We Need an LLM to Know Why Sources Conflict? Decision Models for Knowledge-Conflict Typing in RAG](https://arxiv.org/abs/2610.36915v1){:target="_blank"}
- [From Intent to Improvement: Agentic Dataset Search for Better Model Training](https://arxiv.org/abs/2610.39608v1){:target="_blank"}
- [DeFT-RAG: Incrementally Decomposing Multi-Hop Questions into Self-Contained Sub-Questions with Reusable Answers](https://arxiv.org/abs/2610.37826v1){:target="_blank"} (저장소 404, 11/01 기준)
- [Don't Lose the Thread: Empowering Long-Horizon LLM Agents with Cognitive Resource Self-Allocation](https://arxiv.org/abs/2610.40350v1){:target="_blank"} (저장소 404, 11/01 기준)
- [Beyond Passing Tests: A Systematic Review and Controlled Re-evaluation of Techniques for Improving the Quality of LLM-Generated Code](https://arxiv.org/abs/2610.38977v1){:target="_blank"}
- [Manifest-Assisted Retrieval in Artificial Intelligence Search: Evidence Gaps and a Security Evaluation](https://arxiv.org/abs/2610.38361v1){:target="_blank"}
