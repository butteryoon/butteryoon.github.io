---
layout: post
comments: true
title: "RAG & AI 에이전트 주간 연구 동향 (2026-09-14 ~ 09-20)"
description: "이번 주 arXiv에 올라온 RAG, 벡터 검색, 리랭킹, 에이전트 하네스·평가 분야 주요 논문 요약"
img: ai_abstract_title.jpg
date: 2026-09-20 17:00:00 +0900
last_modified_at: 2026-09-20 17:00:00 +0900
tags: [rag, ai agent, llm, vector database, reranking, arxiv, weekly] # add tag
related: llm
categories: dev
---

Hermes 에이전트가 배치 작업으로 수집한 RAG·AI 에이전트 분야 주간 논문 보고서(arXiv cs.IR/cs.CL/cs.MA/cs.AI/cs.LG + Hugging Face Daily Papers, 2026-09-14~20)를 정리한다. 토픽별 대표성을 위해 기간 직전 등록분도 일부 포함했다.

<!--more-->

> **TL;DR:** 이번 주 RAG는 그래프를 만들 때 LLM을 아예 쓰지 않는 쪽으로 움직였다. 에이전트 쪽은 모델 자체보다 하네스 — 토큰 효율, 회귀 테스트, 검증기 — 에 논문이 몰렸고, 임베딩은 추론 능력 보존과 양자화 실측으로 갈렸다. 멀티 에이전트 조정은 게임 이론·정보 이론 쪽 논증이 붙기 시작했다.

## RAG 아키텍처: 그래프에서 LLM 호출을 걷어내다

- **[G³RAG](https://arxiv.org/abs/2609.19622)**: 엔티티 추출 없이 문서 임베딩만으로 그래프를 짠다. 기하학적 이득 점수(cosθ·sinθ)와 밀도 인식 위상 페널티로 허브 노드를 눌러주고 한 단계짜리 제어 확산으로 보완 증거를 찾는다. MuSiQue·2WikiMultiHopQA·HotpotQA에서 평균 F1 최대 +4.26, MuSiQue는 +5.76. 그래프 구성에 드는 토큰 비용이 0이 된다는 게 핵심이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"G$^3$RAG obtains the best average F1 and answer-document hit rate among the evaluated graph-based baselines in both embedding settings, with gains of up to 4.26 F1 points in average performance and 5.76 points on MusiQue." / "It also removes the graph-construction token cost incurred by entity-based graph methods."</blockquote></details>

- **[Algebraic Retrieval](https://arxiv.org/abs/2609.19482)**: 검색을 에이전트가 쿼리 타임에 조립하는 대수 연산으로 본다. 연관성 기준·자격 제약·랭킹 선호를 각각 수학적 쿼리로 쓰고 Programmatic Embedding Modulation으로 대조 점수·후보 풀 재랭킹·가중 랭킹을 합성한다. 11,429문서 Vaswani 픽스처에서 SQL/PyTerrier 구현과 점수 차이 1e-6 미만으로 실행 패리티를 확인했다.
- **[RAFT](https://arxiv.org/abs/2609.20754)**: 엔터프라이즈 트러블슈팅용 상태 보존 RAG. 종결된 케이스를 타임라인 엔트리의 방향성 체인으로 추상화해두고 엔트리 단위로 검색해 지금 진행 중인 케이스의 중간 상태와 맞춘 뒤 부모 케이스 궤적을 돌려준다. MS Learn 합성 벤치마크와 실제 Apache Jira 중복 라벨 데이터 양쪽에서 vanilla RAG·GraphRAG 대비 Case Hit이 전 단계 유의미하게 올랐다.
- **[MSS-Complement](https://arxiv.org/abs/2609.20050)**: 코딩 에이전트의 검색을 "다음 결정에 아직 부족한 최소 충분 증거를 복구하는 문제"로 다시 정의한다. SERBench(45개 레포, 500 상태) 기준 캘리브레이션 고정 구성에서 5개 항목 73.0%, 8개 항목 80.6% 완전 복구 — 임베딩+재랭킹의 61.4%/72.4%를 앞선다. AMA-Bench에서는 답변 프롬프트를 76.2% 줄이면서 정확도가 2.08pt 올랐다.

## 벡터 검색·임베딩: 추론을 죽이지 않고 임베딩을 학습하기

- **[CoFree](https://arxiv.org/abs/2609.20563)**: LLM으로 임베딩을 학습시키면 임베딩 목적에 특화되면서 추론 능력이 눌리거나 검색과 무관한 텍스트가 나온다. 저자들은 이걸 추론 붕괴(reasoning collapse)라고 부른다. 1단계 참조 가이드 SFT로 추론력을 되살리고 2단계에서 임베딩 지향·추론 지향 이중 보상으로 RL을 돌린다. CoFree-4B가 MTEB·BRIGHT 22개 데이터셋 평균 +2.8 nDCG@10.

<details class="evidence"><summary>원문 근거</summary><blockquote>"CoFree-4B achieving an average absolute improvement of 2.8 nDCG@10 points over Qwen3-Embedding-4B across 22 datasets from MTEB and BRIGHT."</blockquote></details>

- **[PTQ 실측 맵](https://arxiv.org/abs/2609.16391)**: BGE·E5·GTE·Jina 네 패밀리 24개 모델에 INT4/INT8 포스트-트레이닝 양자화를 걸어 nDCG@10 열화 패턴을 아키텍처·학습 목적·데이터 규모별로 매핑했다. 측정 레저와 분석 코드를 공개(ThakiCloud/skillret-ptq-measurements)해서, 벡터 DB에 어떤 임베더를 몇 비트로 올릴지 고를 때 바로 참고할 만하다.

## 시맨틱 서치: 궤적 하나에 기대지 않는 리랭킹

- **[MERIT-Rank](https://arxiv.org/abs/2609.20131)**: 재랭커가 단일 추론 궤적에 의존하면 흔들린다는 문제에서 출발해, 다중-궤적 추론 공간(MTRS)으로 쿼리-문서 연관성을 여러 각도에서 평가하고 공동 재랭커로 합친다. 점진적 랭크 정책 최적화(PRPO)가 궤적을 안정화한다. 추론 집약형과 전통 벤치마크 모두 SOTA를 넘었고, BRIGHT에서는 4B 모델이 여러 7B·32B 재랭커를 앞섰다.

## 에이전트 하네스: 모델이 아니라 껍데기를 고치는 주

- **[SoL-Pi](https://arxiv.org/abs/2609.20519)** (NVIDIA): 자동 연구 루프를 다양한 환경으로 재귀 확장해 재사용 가능한 개선점을 뽑아내고 액션 실행·컨텍스트 압축·관찰 처리·위임 읽기 네 가지 생존 메커니즘으로 하네스를 구성했다. EdgeBench 51개 태스크에서 Pi와 성능은 동등한데 토큰 트래픽은 44.7~49.0% 줄고 API 비용은 약 1/3로 떨어진다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"reducing recorded token traffic by 44.7-49.0% and API cost by about one third"</blockquote></details>

- **[Chronicle](https://arxiv.org/abs/2609.20625)**: 에이전트 비결정성 탓에 회귀 테스트가 어렵다는 고질병을 다룬다. 모델 호출·도구 호출 같은 비결정적 경계를 불변 엔벨로프로 기록해두고 컷-포인트 리플레이로 기록된 서브셋만 서빙하고 나머지는 새 코드로 실시간 실행한다. 기록 오버헤드는 경계당 23µs(0.008%), 완전 리플레이는 모델 호출 0회로 20회 반복까지 비트 단위 안정. 변이 연구에서 스텁 기준 잡음이 0% 대 100%로 갈렸다.
- **[하네스 설계 실증 연구](https://arxiv.org/abs/2609.20804)**: τ2-bench Retail/Airline에서 사전 작성 계획(Fixed)과 단어 수만 맞춘 셔플(Sham)을 비교해 계획 "내용"의 기여를 분리했다. 265개 매칭 셀에서 Fixed가 오라클 검증 성공률 +7.17%p(90% CI 1.15~13.36), 고복잡도 태스크에 몰려 있다. 읽기전용 터미널 검증기는 오라클-무효 에피소드의 61%를 거부하면서 정답은 17%만 보류했고 에피소드당 비용이 1센트 미만이다. [후속 분석](https://arxiv.org/abs/2609.20474)은 검증기 단독으로 전체 스택의 거짓통과 방지 이득을 대부분 흡수한다는 점까지 밀어붙인다.
- **[EvoSkill-GUI](https://arxiv.org/abs/2609.17653)**: GUI 에이전트 스킬을 정적 파일로 두지 않고 배포 시점에 학습 없이 진화시킨다. 스킬은 검색 메타데이터·실행 계획·백업 로컬라이제이션·실패 복구 규칙·접근성 유틸·실패 케이스를 묶은 다중 파일 패키지다. Reflect-Revise-Reuse 루프에서 실행기가 즉시 고치고 격리된 크리틱이 진단하고 제한된 도구 인터페이스로 스킬 파일을 편집한다. MobileWorld·AndroidWorld·OSWorld에서 베이스 모델별 최대 +16.2%/+6.0%/+10.5%.
- **[RetireOPD](https://arxiv.org/abs/2609.20784)**: 온-정책 증류에서 교사를 언제 버릴지를 다룬다. 기술 조건부 교사를 따로 최적화한 뒤 스킬-프리 학생을 RL+OPD로 공동 학습하고 학생-교사 불일치 수축이 멈추고 교사 성공률이 목표에 닿으면 교사를 드롭한 채 RL만 이어간다(적응적 은퇴). Qwen2.5 1.5B~7B에서 ALFWorld +14.1~18.8%, WebShop +11.8~19.0%으로 자체 교사를 넘어섰다.

## 멀티 에이전트: 조정에 이론이 붙는다

- **[Bilevel Coordinated Reflection](https://arxiv.org/abs/2609.02750)**: 오케스트레이터-워커 상호작용을 바이레벨 조정 게임으로 모델링한다. 워커 로컬 업데이트 게임을 근사 포텐셜 게임으로 놓고 분해 품질로 균형 슬랙을 통제한다. 반성은 시맨틱 메모리 상태 위의 확률적 이동으로 분석해 자유형 반성의 유한 시간 상한과 영속 피해 조건 하의 하한을 유도했다. 텍스트만 보는 게이트로는 텍스트-구별불가 환경에서 균일한 개선이 불가능하지만 환경 그라운딩 게이트는 가능하다는 걸 정보이론적으로 증명한다. SWE-bench 500개에서 Kimi 기반 시스템 72.2%(공개 mini-SWE-agent 70.8%).
- **[AgentGrad](https://arxiv.org/abs/2609.08572)**: MAS 프롬프트 최적화에서 순차 개입으로 실패를 해결한 에이전트를 찾아내고 수정된 출력을 에이전트 레벨 감독으로 삼아 그래디언트를 세밀하게 뽑는다. 의미적 그래디언트 추상화로 비슷한 그래디언트를 묶어 공유 교정 패턴을 일반화한다. 5개 MAS 벤치마크 SOTA에 차선 대비 벽시계 최적화 시간 평균 2.5배 단축.
- **조정 구조 실험들**: 하위 에이전트 결과를 상위가 다시 보는 [루프-백 권한](https://arxiv.org/abs/2609.14767)을 평면·계층 팀에서 페어 실험으로 비교한 연구, 태스크 난이도에 따라 협업 정도를 조절하는 [난이도 인식 토폴로지 선택](https://arxiv.org/abs/2609.13890), 평판을 커뮤니티 메모리로 쓰는 [에이전틱 웹 프레임워크](https://arxiv.org/abs/2609.19502)(eScience 2026 채택)가 나왔다.
- **[LearnActCoder](https://arxiv.org/abs/2609.19721)**: 임상 코딩 에이전트의 반복 실패 모드(미지원 코드·누락 조건·특이도 오류·프로시저 관례 불일치)를 소량 라벨 배치에서 구조화된 MistakeKDB로 바꾸고 위음성은 회상 지향 Coder로 위양성은 정밀도 지향 Judge로 라우팅한다. MIMIC-III 150노트에서 CPT F1 +5.9%p. 다만 MIMIC-IV ICD-10은 정밀도가 오르는 대신 재현율을 내주면서 F1은 통계적으로 변하지 않았다.

## 평가: 재현 가능한데 틀릴 수 있다는 문제

- **[Refuse, Decompose, Refresh](https://arxiv.org/abs/2609.20538)**: 폐루프 평가가 완전히 재현 가능해도 잘못된 클레임을 지지할 수 있다는 간극을 세 단계 계약으로 메운다. 깨끗한 참조 스트림이나 매칭 런타임 비교가 없으면 기권(Refuse)하고 프로토콜 실행·운영적 거짓 수용·구조적 가설을 단일 PASS/FAIL로 뭉개지 않고 따로 보고하며(Decompose) 분포 이동 알람은 결함 증거가 아니라 참조 맵 무효화·재계산 요청으로 처리한다(Refresh). 사전등록 헬드아웃 1,440케이스·21,600 파티션 행에서 안정적 거짓 수용 0/20(단측 95% 상한 0.1391).
- **[LLM 그룹의 합의 과대추정](https://arxiv.org/abs/2609.20543)**: 100개 held-out 인간 Wason 그룹을 LLM 에이전트 그룹으로 재현하고 동일 코드로 채점했다. 에이전트의 신념은 참가자 사전 답변으로 고정했다. 인간은 채점 정의에 따라 24.0~57.0%, 참가자 중 1/5쯤은 아예 글을 올리지 않는데 에이전트는 거의 항상 참여한다. 결과적으로 에이전트 그룹이 34.0~44.4%p 더 합의적이었고, 조기 중단과 암기 답변을 걷어낸 재파라미터화 뒤에도 격차가 남았다. 추론 모드에서는 거의 만장일치인데 대다수가 오답이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"the submit-based comparison (n = 98) yielded gaps of 34.0 and 43.9 percentage points for chat and reasoning modes, and the participation-matched comparison (n = 45) yielded gaps of 34.1 and 44.4 points."</blockquote></details>

## 정리

- **RAG**: 그래프 구성에서 LLM을 빼고 임베딩 기하만으로 가는 흐름, 검색을 조합 가능한 연산 인터페이스로 노출하려는 시도
- **벡터·임베딩**: 임베딩 특화가 추론을 갉아먹는다는 진단과 그 복구, 그리고 양자화 열화의 전수 실측
- **에이전트**: 모델보다 하네스 — 토큰 트래픽 절반 절감, 결정론적 리플레이, 1센트짜리 읽기전용 검증기
- **평가**: 재현성과 타당성을 분리하고, 시뮬레이션 합의가 집단 정확도를 따라가지 못한다는 반례

운영 중인 시스템에 당장 얹어볼 만한 건 세 편이다. 그래프 RAG의 인덱싱 비용이 부담이면 G³RAG, 에이전트 회귀 테스트가 없으면 Chronicle, 그리고 에이전트 출력을 사람이 다 볼 수 없다면 하네스 실증 연구의 읽기전용 터미널 검증기.

## 참고

- 각 논문의 arXiv 링크는 본문에 표기 (2609.xxxxx 시리즈)
- [지난 글: RAG & AI 에이전트 주간 연구 동향 (2026-09-06 ~ 09-13)]({{site.baseurl}}/dev/2026/09/13/rag_agent_weekly.html)
