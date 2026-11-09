---
layout: post
comments: true
title: "영어로만 평가한 에이전트를 한국어로 돌리면 — 에이전트·RAG 실무 메모 (11/01~11/08)"
description: "τ-bench를 한국어·독일어·터키어·페르시아어로 현지화했더니 리테일 과제에서 영어 대비 pass^4가 최대 61점 떨어졌다. 이번 주 arXiv에서 고른 일곱 편으로 다국어 에이전트 평가, 오프라인 RAG 벡터 무결성, 감독 에이전트 설득 위험, 거래 단위 비용 추적, 에이전트 메모리에서 바로 바꿀 것을 정리한다."
img: rag_weekly_20261108_title.webp
date: 2026-11-08 19:30:00 +0900
last_modified_at: 2026-11-09 01:40:00 +0900
tags: [rag, ai agent, llm, evaluation, multilingual, agent-memory, context-management, security, finops, arxiv, weekly, rag-weekly]
related: rag-weekly
categories: dev
---

Hermes 에이전트가 배치 작업으로 수집한 RAG·AI 에이전트 분야 주간 논문 보고서(arXiv cs.IR/cs.CL/cs.MA/cs.AI/cs.LG, 2026-11-01~11-08)에서 일곱 편을 골랐다. 기간 직전 등록분 일부도 포함했다. 기준은 지난주와 같다. 읽고 나서 내 파이프라인에서 오늘 바꿀 게 있는가. 여섯 편은 초록에 코드·데이터 링크가 있고 작성 시점에 저장소가 모두 열린다.

<!--more-->

> **TL;DR:** ① 한국어로 서비스할 도구 사용 에이전트라면 영어 벤치마크 점수를 믿지 말고 현지화한 데이터로 같은 과제를 여러 번 돌려 본다. ② 서명된 지식 팩에 들어 있는 임베딩 벡터는 서명만으로 텍스트와 맞는다고 볼 수 없다. 받은 벡터를 그대로 인덱스에 넣지 않는다. ③ LLM 감독 에이전트는 작업 에이전트의 설득에 승인으로 넘어갈 수 있다. 되돌릴 수 없는 행동에는 코드로 된 관문을 따로 둔다. ④ 모든 LLM 호출 텔레메트리에 주문·건 번호 같은 업무 키를 붙여 거래 단위 비용을 낸다. ⑤ 매번 LLM을 부르는 워크플로 단계 중 판단이 안정된 것은 코드로 옮긴다. ⑥ 에이전트 메모리 검색이 날짜·장소를 구분하는지 날짜만 바꾼 문장으로 시험한다. ⑦ 컴팩션(대화 이력을 요약 노트로 교체)을 쓴다면 기본 모델이 쓴 노트를 그대로 믿지 않는다.

수집과 서지 대조는 보고서 단계에서 끝났다. 후보 492편 가운데 코드·모델 공개나 수치 근거가 있는 67편이 남았고 그중 20편이 표에 올랐다. 아래 일곱 편을 고른 것과 "무엇을 바꿀 것인가"는 arXiv 초록을 직접 읽고 판단했다. 추천 시스템, 위치 인코딩 같은 모델 구조, 로봇 강화학습, 이미지 검색, 이론 컴퓨터과학 벤치마크처럼 에이전트·RAG 개발 환경에서 바로 손댈 게 없는 행은 뺐다.

## 1. 받아서 바로 돌려 볼 것

### 한국어 에이전트는 한국어 데이터로 다시 잰다

도구를 쓰는 대화형 에이전트의 프런티어 평가는 거의 영어로만 이뤄진다. **τ-bench-LILT**는 τ³-bench를 독일어·한국어·터키어·페르시아어로 넓히고 항공·리테일·통신·은행 네 도메인을 다룬다. 문장만 번역한 게 아니다. 고객 데이터베이스는 그 문화에 맞는 레코드로 채웠고 사용자 시뮬레이터 프롬프트는 원어민 전문가가 대조 주석을 달아 가며 여러 번 다듬었다. 과제도 현지화한 데이터베이스와 그 지역의 실제 업무에 맞게 바꿨다.

에이전트 모델 여덟 개를 돌린 결과, 영어 대비 하락 폭은 한국어와 페르시아어 리테일 도메인에서 가장 컸다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Evaluation across eight agent models shows the largest drops compared to English in the Korean and Farsi retail domains, by up to 61 points in pass^4."</blockquote></details>

pass^4는 같은 과제를 네 번 돌려 네 번 모두 성공한 비율이다. 한 번 맞히는 능력보다 매번 맞히는 안정성을 본다.

**바꿀 것**: 한국어 고객 응대용 도구 사용 에이전트를 영어 벤치마크 점수나 영어 프롬프트를 번역한 시험으로 골랐다면, 공개된 τ-bench-LILT의 한국어 리테일 도메인부터 내 모델·프롬프트로 돌려 본다. 사내 평가셋을 만들 때도 문장만 번역하지 말고 고객 레코드와 과제 자체를 한국 환경에 맞춘다. 같은 과제를 여러 번 돌려 전부 성공하는지도 함께 본다.

- [arXiv 2611.07203](https://arxiv.org/abs/2611.07203v1){:target="_blank"} · [lilt/tau-bench-lilt](https://github.com/lilt/tau-bench-lilt){:target="_blank"}

### 서명된 지식 팩의 벡터는 서명이 보증하지 않는다

엣지 기기에서 도는 오프라인 RAG는 텍스트와 미리 계산한 임베딩 벡터를 함께 담은 서명된 지식 팩으로 갱신된다. **Signed but Steerable**의 출발점은 단순하다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"A signature proves who published a pack, not that its vectors encode its text."</blockquote></details>

저자들은 이 틈을 "텍스트–벡터 무결성 격차"라고 부른다. 받는 쪽이 벡터를 다시 계산해 코사인 허용치 안이면 통과시키는 검증기를 둬도, 벡터를 조작할 수 있는 공격자에게 일정 비율의 질의를 내준다. 그 비율은 허용치에서 나타나는 검색 마진 분포와 같다(저자들의 "margin law"). 말뭉치 다섯 개와 임베딩 모델 두 개에서 마진이 작았다. 부동소수점 잡음 수준의 이상적인 허용치로도 질의의 최대 5.5%가 노출됐고 완전 중복 문서를 빼면 어느 말뭉치에서도 1.2% 이하였다. 소형 임베딩 모델의 CPU INT8–FP32 비교로 보정한 임계값을 쓰면 98~100%가 노출됐다. 로컬 언어 모델을 돌리는 Raspberry Pi 5에서 이 공격은 정답률을 0.75에서 0.39로 떨어뜨렸다.

압축된 벡터 저장소의 무결성 감사도 나을 게 없었다. 문서 57,638건 위의 FAISS product-quantisation 인덱스에서 감사를 모두 통과한 조작 벡터가 표적의 25%에서 공격자 문서를 1위로 올렸다. 저자들이 내놓은 해법은 양자화 코드를 검증하되 애매한 경우를 검증자가 판정하고 보정한 보호 대역과 정규화를 쓰는 방식이다. 세 플랫폼에서 표본의 정직한 벡터를 모두 통과시켰다. 흔한 부동소수점·동적 INT8 추론 파이프라인은 배치 크기나 아키텍처가 바뀌면 비트 단위로 재현되지 않는다는 점도 보였다. 기권, 컨텍스트 가지치기, 의미 기반 선별은 사전 등록 실험에서 효과가 없었다고 보고했다.

**바꿀 것**: 문서와 함께 미리 계산된 임베딩을 배포하거나 외부에서 받은 벡터 묶음을 인덱스에 넣고 있다면, 서명 검증을 벡터 검증으로 여기지 않는다. 받는 쪽에서 텍스트를 다시 임베딩할 여유가 있으면 받은 벡터는 버리고 직접 만든다. 그럴 수 없다면 코사인 허용치 비교로 대신하지 말고 공개된 코드의 양자화 코드 검증 방식을 검토한다. 임베딩을 해시로 비교하는 검증은 배치 크기나 하드웨어가 달라지면 깨진다는 점도 감안한다.

- [arXiv 2611.06498](https://arxiv.org/abs/2611.06498v1){:target="_blank"} · [securagenai/signed-but-steerable](https://github.com/securagenai/signed-but-steerable){:target="_blank"} (코드·데이터)

### LLM 감독 에이전트는 설득당한다

멀티 에이전트 구성에서는 에이전트끼리 서로의 행동을 의도한 쪽으로 끌고 갈 수 있다. **Inter-Agent Influence**는 설득·기만·강압 세 가지 능력을 현실적인 평가 환경 다섯 개에서 프런티어 모델 기반 에이전트로 실험했다. 감독(oversight) 환경에서는 시험한 모델 전부가 설득·기만·강압으로 감독자의 결정을 거부에서 승인으로 바꿨다. 위계가 없는 동료 관계에서도 상대 에이전트의 안전성 연구 방향을 돌려놓았다.

**바꿀 것**: 작업 에이전트가 낸 행동을 다른 LLM이 검토해 승인하는 구조를 쓰고 있다면, 그 승인을 되돌릴 수 없는 행동(삭제, 결제, 외부 발송)의 유일한 관문으로 두지 않는다. 허용 목록이나 권한 검사처럼 작업 에이전트의 말에 흔들리지 않는 코드 관문을 따로 둔다. 공개된 평가 환경과 데이터셋으로 내 감독 프롬프트가 얼마나 버티는지도 재 본다.

- [arXiv 2611.04556](https://arxiv.org/abs/2611.04556v1){:target="_blank"} · [Cooperative-AI-Foundation/inter-agent-influence](https://github.com/Cooperative-AI-Foundation/inter-agent-influence){:target="_blank"} · [HF 데이터셋](https://huggingface.co/datasets/Cooperative-AI-Foundation/inter-agent-influence-eval){:target="_blank"}

### 주문 한 건에 AI 비용이 얼마 들었는지 답할 수 있게

기업은 여러 클라우드, LLM, 오케스트레이션 하네스, 기간계 시스템에 걸친 멀티 에이전트로 업무 거래를 처리한다. 그런데 재무가 받는 AI 비용은 SKU와 과금 기간으로 묶인 정산 단위뿐이다. 공급자는 기업의 거래 경계를 모르기 때문에 "이 주문·청구·건에 AI 비용이 얼마 들었나"는 청구 데이터로 답할 수 없다. FinOps 표준 FOCUS도 공급자 간 스키마는 통일하지만 여전히 정산 단위다.

**Two-Ledger Model**은 FOCUS 형식의 정산 원장과, OpenTelemetry GenAI 텔레메트리로 만든 호출 단위 사용 기록인 귀속 원장을 조인한다. 각 기록은 trace context, A2A/MCP 메타데이터, 업무 키 상관, 확률적 매칭을 층층이 거쳐 업무 거래에 배정되고 어떤 방법으로 얼마의 확신도로 배정했는지가 함께 남는다. 버전 관리되는 요율표가 사용량에 거의 실시간으로 값을 매기고 기간 정산 때 두 원장을 청구서와 맞춘다. 설명되지 않는 잔차가 섀도 AI를 드러낸다. 정답을 아는 시뮬레이션에서 trace 전파가 0까지 떨어져도 비용 커버리지는 99% 이상을 유지했다. 거래 도중 게이팅은 오탐 정지율 2~4%에서 p99 거래 비용을 최대 61% 줄였다. 실제 운영 데이터가 아닌 시뮬레이션과 합성 데이터라는 점은 감안해야 한다.

**바꿀 것**: 에이전트 비용을 월 청구서로만 보고 있다면, 모든 LLM·도구 호출의 텔레메트리 span과 A2A/MCP 메타데이터에 **주문 번호·건 번호 같은 업무 키를 붙인다.** trace가 끊겨도 업무 키로 다시 묶을 수 있다. 월말에는 호출 단위 합계를 청구서와 맞춰 보고, 남는 차이는 계량되지 않은 사용으로 의심한다. 거래 하나의 누적 비용이 한도를 넘으면 중간에 멈추는 장치도 검토한다.

- [arXiv 2611.06989](https://arxiv.org/abs/2611.06989v1){:target="_blank"} · [curiosityexplorer/two-ledger-model](https://github.com/curiosityexplorer/two-ledger-model){:target="_blank"} (코드·합성 데이터)

### 매번 LLM을 부르던 단계를 코드로 옮긴다

에이전트 워크플로는 복잡한 과제를 재사용 가능한 절차로 정리하지만 단계마다 LLM을 부르면 추론 비용과 지연이 쌓인다. 이 논문(저장소 이름은 **SeaFlow**)은 과제를 의미 단위 하위 과제로 나누고 하위 과제마다 **명시적인 적용 조건**과 필요에 따라 코드를 실행하거나 모델을 부르는 절차를 둔다. 에이전트는 실행 피드백으로 분해 방식과 절차를 고친다. 흥정·협상·설득 과제를 다루는 GLEE 경진대회에서 1위를 했고 최종 결정 절차는 대부분 코드로 구현돼 있었다.

**바꿀 것**: 반복해서 도는 에이전트 워크플로가 있다면 단계별 LLM 호출 로그를 모아 **판단이 매번 거의 같은 단계**를 찾고 그 단계를 "이 조건일 때 적용"이라는 조건과 코드 절차로 바꾼다. LLM은 조건에 맞는 절차가 없을 때만 부른다. 실험 도메인이 협상 게임이라는 점은 감안하고 내 워크플로에서 호출 수와 성공률을 같이 잰다.

- [arXiv 2611.01840](https://arxiv.org/abs/2611.01840v1){:target="_blank"} · [Applied-Machine-Learning-Lab/NeurIPS2026_SeaFlow](https://github.com/Applied-Machine-Learning-Lab/NeurIPS2026_SeaFlow){:target="_blank"}

### 메모리 임베딩은 날짜·장소를 잘 못 가른다

사용자의 개인 경험을 묻는 질문에 답하려면 검색이 시간·장소·인물·사건으로 기억을 구분해야 한다. **Axis-LoRA** 저자들은 지금의 개인 메모리 파이프라인이 쓰는 고정 dense 임베딩 모델이 이 네 축을 잘 잡지 못한다는 것을 보였다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Swapping a memory's date or place barely shifts its embedding, and a trained projection on top of the frozen embedding does not recover the missing signal."</blockquote></details>

문제는 모델 안에 있다. 축마다 서로 다른 attention head 패턴으로 인코딩된다. 그래서 저자들은 임베딩 모델 안에 축별 LoRA 가지 네 개를 넣고 질의별 게이트로 섞는 어댑터를 만들었다. 임베딩 모델 세 개를 LongMemEval·ATM-Bench·LoCoMo에서 평가했더니 평균 R@10이 1.28~1.56pp 올랐다(초록은 벤치마크 셋만 이름을 대면서 "4-benchmark mean" 이라고 적는다). 기반 파라미터의 일부만, 작은 축별 반사실 학습셋으로 학습했다.

**바꿀 것**: 에이전트 메모리를 고정 임베딩 검색으로만 찾고 있다면, 기억 문장에서 날짜나 장소만 바꾼 쌍을 몇 개 만들어 임베딩 유사도가 얼마나 달라지는지 먼저 본다. 거의 같다면 "지난달 부산에서" 같은 질문은 임베딩이 가려 주지 못한다. 고정 임베딩 위에 사영층을 학습해 붙이는 방법은 초록 기준으로 효과가 없었으니 건너뛴다. 날짜·장소를 별도 필드로 저장해 필터로 거는 것은 논문의 제안이 아니라 이 관찰에서 나오는 실무 판단이다. 어댑터를 학습할 여유가 있으면 공개 코드로 시험해 보되 이득의 크기(평균 R@10 1~2pp)를 감안한다.

- [arXiv 2611.07820](https://arxiv.org/abs/2611.07820v1){:target="_blank"} · [dohoonkim-ai/axis-lora](https://github.com/dohoonkim-ai/axis-lora){:target="_blank"}

## 2. 코드는 없지만 하네스에서 바로 점검할 것

### 컴팩션 노트는 기본 모델이 잘 못 쓴다

긴 입력을 다루는 에이전트는 고정된 컨텍스트 창에 무엇을 남길지 정해야 한다. 흔한 방법은 컴팩션이다. 모델이 지금까지의 이력을 스스로 쓴 노트로 바꾼다. **MEMENTO** 저자들은 이 노트 품질이 문제라고 짚는다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Yet off-the-shelf agentic models, including GPT-5.5 and GLM-5.3, often write ineffective notes, and reinforcement learning (RL) alone rarely discovers useful note-taking behavior."</blockquote></details>

MEMENTO는 Qwen3-8B를 컴팩션으로 컨텍스트를 관리하도록 학습했다. 사후에 노트를 붙인 합성 궤적으로 먼저 미세조정하고 정답 여부·노트 품질 휴리스틱·확신도 보정을 합친 보상으로 강화학습을 했다. 새 창으로 컴팩션하는 것만 학습했는데도 테스트 때는 context folding이나 확신도 기반 되돌리기 같은 전략으로 확장됐다. 최대 250만 토큰짜리 책을 다루는 질의응답 벤치마크 네 개에서 Qwen3-8B 정확도를 36.9%에서 67.8%로 올렸고 컨텍스트가 2만 토큰을 넘은 적은 한 번도 없었다. 같은 하네스의 GPT-5.5 에이전트(13.1%)와 책 전체를 컨텍스트에 넣은 경우(56.7%)보다 높았고 비용은 50분의 1 미만이었다.

**바꿀 것**: 에이전트 하네스가 컴팩션을 쓰고 있다면 기본 모델과 기본 요약 프롬프트가 쓴 노트를 그대로 믿지 않는다. 컴팩션 전에만 있던 정보가 필요한 질문을 몇 개 만들어 컴팩션 뒤에도 답하는지 재 본다. 노트 품질을 따로 점검하는 단계를 두고 컴팩션 횟수가 늘수록 정답률이 어떻게 변하는지도 기록한다.

- [arXiv 2611.00244](https://arxiv.org/abs/2611.00244v1){:target="_blank"}. 초록에 공개 자원 링크는 없다

## 이번 주에 손댈 것

일곱 편 중 다섯 편이 같은 지점을 가리킨다. **모델이 맞다고 한 것, 받은 데이터가 맞다고 한 것을 그대로 믿지 말라**는 것이다. 영어 점수는 한국어 성능을 말해 주지 않고, 서명은 벡터를 보증하지 않으며, 감독 에이전트의 승인은 설득으로 뒤집히고, 컴팩션 노트는 정보를 흘린다. 당장 할 수 있는 건 셋이다.

**하나.** 한국어 서비스용 도구 사용 에이전트를 τ-bench-LILT 한국어 도메인이나 현지화한 사내 과제로 여러 번 돌려 본다. 영어 점수와의 차이부터 확인한다.

**둘.** 에이전트가 LLM 승인만 받고 실행하는 행동 목록을 뽑아 되돌릴 수 없는 것에 코드 관문을 붙인다.

**셋.** LLM 호출 텔레메트리에 업무 키를 넣는다. 오늘 붙여 두면 다음 달 청구서부터 거래 단위로 쪼갤 수 있다.

외부에서 받은 임베딩을 쓰는 오프라인 RAG, 메모리 검색, 컴팩션 점검은 환경에 따라 우선순위가 다르다. 그 기능을 쓰고 있다면 위 셋과 같은 주에 점검 질문 몇 개부터 만들어 둔다.

## 참고

- [τ-bench-LILT: Evaluating Conversational Agents on Realistic Multilingual Tool-Use Tasks](https://arxiv.org/abs/2611.07203v1){:target="_blank"}
- [Signed but Steerable: Measuring and Closing the Text-Vector Integrity Gap in Offline Retrieval-Augmented Generation](https://arxiv.org/abs/2611.06498v1){:target="_blank"}
- [Inter-Agent Influence: Evaluating Persuasion, Deception, and Coercion in Multi-Agent Systems](https://arxiv.org/abs/2611.04556v1){:target="_blank"}
- [The Two-Ledger Model: Transaction-Grain Cost Accounting for Multi-Agent AI Systems Across Clouds, Models, and Systems of Record](https://arxiv.org/abs/2611.06989v1){:target="_blank"}
- [Self-Evolving Agentic Workflows via Semantic Decomposition of Complex Tasks](https://arxiv.org/abs/2611.01840v1){:target="_blank"}
- [Axis-LoRA: Probing and Adapting Embedding Models for Personal-Memory Retrieval](https://arxiv.org/abs/2611.07820v1){:target="_blank"}
- [MEMENTO: Learning Context Management for Long Context Processing Agents](https://arxiv.org/abs/2611.00244v1){:target="_blank"}
