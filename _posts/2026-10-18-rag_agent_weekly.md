---
layout: post
comments: true
title: "평가가 통과했는데 틀렸다 — 에이전트 평가가 조용히 무너지는 네 가지 방식 (10/11~10/18)"
description: "에이전트 메모리를 다운스트림 QA로 재면 메모리 연산 정확도 20%가 안 보인다. 가드 벤치마크는 폴드를 바꾸면 F1이 0.901에서 0.495로 떨어진다. 이번 주 arXiv에서 고른 네 편으로 평가 설계에서 바로 바꿀 것을 정리한다."
img: ai_abstract_title.jpg
date: 2026-10-18 22:00:00 +0900
last_modified_at: 2026-10-18 22:00:00 +0900
tags: [rag, ai agent, llm, evaluation, guardrail, agent-memory, arxiv, weekly, rag-weekly]
related: rag-weekly
categories: dev
---

이번 주 arXiv에서 고른 네 편에는 공통점이 있다. 전부 **"평가가 통과했는데 실제로는 틀렸다"**는 이야기다. 모델을 더 좋게 만드는 연구가 아니라, 내가 지금 쓰고 있는 평가가 무엇을 못 보고 있는지를 지적하는 연구다. 이 중 둘은 저장소가 공개돼 바로 받아 돌려 볼 수 있다.

<!--more-->

> **TL;DR:** ① 에이전트 메모리를 다운스트림 QA로만 재면 메모리 연산 분류 정확도가 20%여도 모른다. ② 가드 벤치마크는 도메인 홀드아웃으로 바꾸면 F1이 0.901에서 0.495로 주저앉는다. ③ 에이전트는 입력이 오염돼도 알아채지 못하고 그대로 흘려보낸다. ④ 컨텍스트 압축에는 목표를 같이 넣어야 한다.

수집과 서지 검증은 주간 서베이 스크립트가 했다(arXiv API로 464편을 받아 코드·모델 공개 또는 수치 근거가 있는 61편만 남기고, 선정분은 초록 페이지의 `citation_title`과 대조했다). 아래 네 편의 선별과 "무엇을 바꿀 것인가"는 사람이 판단해 썼다.

## 1. 메모리를 질문으로 재면, 메모리가 깨진 걸 모른다

에이전트 메모리 벤치마크 대부분은 **최종 답이 맞았는지**로 메모리 품질을 매긴다. 문제는 그 점수에 검색 품질과 생성 품질이 섞여 들어간다는 것이다. 메모리가 깨졌는데 답이 맞아 통과하고(거짓 음성), 메모리는 멀쩡한데 답이 틀려 메모리 탓을 한다(거짓 양성).

<details class="evidence"><summary>원문 근거</summary><blockquote>"Agent memory benchmarks evaluate whether an agent's responses reflect correct knowledge, measuring downstream question-answering accuracy as a proxy for memory quality."</blockquote></details>

**FEB-30**은 사실이 시간에 따라 바뀌는 30개 궤적을 6개 진단 범주·3개 난도로 만들어, 매 수집 단계마다 **메모리 저장소를 직접 들여다보게** 한다. 문서를 알려진 사실 궤적에서 생성하므로 사람 주석 없이 정답이 확정된다.

숫자가 아프다. 한 대표 메모리 시스템에 적용했더니 직접 측정과 다운스트림 QA 측정이 **평균 16%p 벌어졌고**, 궤적의 **44%**가 유의미한 괴리 기준을 넘었다. 결정적으로 메모리 연산(생성·갱신·강화·대체)을 **올바르게 분류한 비율이 20%**였는데, 이 실패는 다운스트림 평가로는 **전혀 보이지 않았다.**

**바꿀 것** — 메모리를 쓰는 에이전트가 있다면 평가에 "저장소 직접 확인"을 한 줄 추가한다. 수집 단계마다 저장소를 덤프해 두고, 사실이 바뀌었을 때 갱신으로 처리했는지 새 항목으로 쌓았는지를 본다. QA 정확도만 보고 있었다면 그 수치는 메모리에 대해 거의 아무것도 말해 주지 않는다.

- [arXiv 2610.22209](https://arxiv.org/abs/2610.22209v1){:target="_blank"} — 저자들은 벤치마크·문서 생성 파이프라인·채점 프레임워크를 공개하겠다고 밝혔으나, 초록에 적힌 `amazon-science/feb-30-benchmark` 저장소는 **작성 시점(10/18)에 아직 404**다. 지금은 논문의 설계만 참고할 수 있다

## 2. 가드 벤치마크가 가드를 재지 않을 때

도구를 쓰는 에이전트에는 위험한 호출을 막는 가드를 붙인다. 그럼 **그 가드는 무엇으로 검증하나.**

<details class="evidence"><summary>원문 근거</summary><blockquote>"AI agents take actions through tools. Some actions are sensitive, so guards that supervise them must themselves be evaluated reliably."</blockquote></details>

이 논문은 가드 평가 데이터에 **라벨을 맞힐 수 있는 지름길 단서**가 섞여 있다고 지적한다. 예컨대 LinuxArena에서는 **셸 명령의 평균 길이만으로** 궤적 라벨이 상당히 예측된다. 가드가 안전성을 판단한 게 아니라 데이터의 버릇을 외운 것일 수 있다는 뜻이다.

평가 방식을 바꾸면 점수가 무너진다. AgentDojo-Traj에서 도메인을 섞어 폴드를 나눴을 때 F1이 **0.901**이었는데, 도메인 홀드아웃으로 바꾸자 **0.495**로 떨어졌다. AgentHarm-Traj에서는 가중 방식만 바꿨는데 TS-Guard의 F1이 **0.902에서 0.719**로 내려갔다(대신 균형 정확도는 올라갔다).

더 직접적인 실험도 있다. 64스텝 통제 실험에서 `<INFORMATION>` 마커의 표기만 바꿨더니 차단율이 **8.33%p 내려갔고**, 도구 이름을 의미 없는 별칭으로 바꾸자 **11.46%p 올라갔다.** 위험한 호출은 모두 계속 차단됐으니, 움직인 것은 거짓 양성 쪽이다. 저자들은 이 구간을 탐색적 결과로 표현한다.

**바꿀 것** — 가드 평가 폴드를 **도메인 홀드아웃**으로 바꾼다. 지금 점수가 섞인 폴드에서 나온 것이라면 그 수치는 실제 배포 상황을 과대평가하고 있을 가능성이 크다. 그리고 프롬프트의 마커 표기나 도구 이름을 바꿔 보는 **짝 지은 개입**을 한 번 돌려, 가드 판정이 내용이 아니라 표기에 반응하는지 확인한다.

- [arXiv 2610.22243](https://arxiv.org/abs/2610.22243v1){:target="_blank"}

## 3. 입력이 흔들리면, 에이전트는 모르고 그대로 넘긴다

**AgentPerturb**는 교란의 **대상**과 **유형**을 두 축으로 정리하고, APEX-Agents와 Toolathlon에서 프런티어 모델들을 돌렸다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Real-world tasks are often complex and noisy, posing substantial challenges for LLM agents. Yet existing evaluations lack a systematic framework for jointly characterizing potential perturbation targets and types."</blockquote></details>

결론 셋이다. 교란은 일관되게 성능을 떨어뜨리지만 **어떤 교란이 가장 아픈지는 환경마다 다르다.** 에이전트는 입력이 바뀐 것을 **좀처럼 알아채지 못하고**, 오염된 정보를 확인 없이 다음 단계로 넘긴다. 그리고 교란 속에서 성공한 실행은 **연산을 훨씬 많이 쓴다.**

초록에는 하락 폭의 구체적 수치가 없다. 여기서는 그 점을 그대로 둔다.

**바꿀 것** — 회귀 세트에 **교란 실행을 한 벌 더** 붙인다. 깨끗한 입력에서의 성공률만 보고 있다면, 운영에서 처음 만나는 노이즈가 어디를 부수는지 모른 채 배포하는 셈이다. 토큰·시간 사용량도 같이 기록한다 — 성공했는데 비용이 몇 배로 뛴 실행은 성공으로만 세면 안 된다.

- [arXiv 2610.21624](https://arxiv.org/abs/2610.21624v1){:target="_blank"} · [xuyj233/AgentPerturb](https://github.com/xuyj233/AgentPerturb){:target="_blank"}

## 4. 컨텍스트를 압축할 때 목표를 같이 넣어라

긴 세션을 돌리는 에이전트는 컨텍스트를 줄여야 한다. 문제는 **무엇을 남길지 정해야 하는 시점에, 앞으로 어떤 질문이 올지 모른다**는 것이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Context compression in long-running applications of large language models (LLMs) must preserve goal-relevant information, often before specific downstream questions are known."</blockquote></details>

**GACC-Bench**는 7개 컬렉션에서 문서·에이전트 트레이스 **281개 컨텍스트**를 모아, 같은 목적을 공유하는 질문들을 묶어 상위 목표로 바꿨다. 최종적으로 **목표 500개·질문 2,571개**이고 전수를 사람이 검토했다. 압축기에는 원문과 목표와 예산만 주고 **질문은 숨긴 채**, 고정된 리더가 압축 결과만 보고 답하게 한다.

결과는 방향이 분명하다. 최고 성능 압축기도 전체 컨텍스트를 그대로 쓴 경우와 **상당한 격차**가 남았고, 특히 한 목표에 딸린 질문을 빠짐없이 맞히는 데서 벌어졌다. **목표를 넣어 압축하면 목표 없이 압축할 때보다 일관되게 정확도가 올랐고, 엉뚱한 목표를 넣으면 떨어졌다.**

**바꿀 것** — 요약·압축 단계에 **그 세션의 목표 문장을 같이 넘긴다.** 지금 "앞 대화를 요약해라"만 보내고 있다면 한 줄 추가로 개선 여지가 있다. 다만 목표를 잘못 넣으면 오히려 손해라는 것도 같이 확인됐으니, 목표 문장을 자동 생성해 끼워 넣는 구조라면 그 생성부터 점검해야 한다.

- [arXiv 2610.21893](https://arxiv.org/abs/2610.21893v1){:target="_blank"} · [weiting24/GACC-Bench](https://github.com/weiting24/GACC-Bench){:target="_blank"}

## 이번 주에 손댈 것

네 편을 관통하는 한 문장은 **"통과했다는 사실이 맞았다는 뜻은 아니다"**이다. 당장 할 수 있는 것은 셋이다.

**하나.** 메모리를 쓴다면 저장소를 직접 들여다보는 검사를 평가에 넣는다. QA 정확도 뒤에 20%짜리 실패가 숨어 있을 수 있다.

**둘.** 가드 평가 폴드를 도메인 홀드아웃으로 바꾼다. 한 번만 돌려 보면 지금 점수가 얼마나 낙관적이었는지 바로 나온다. F1이 절반 가까이 빠지는 사례가 이미 보고돼 있다.

**셋.** 컨텍스트 압축 프롬프트에 목표 문장을 넣는다. 가장 싸고, 방향이 분명하게 확인된 변경이다.

교란 테스트는 회귀 세트를 한 벌 더 돌려야 해서 비용이 든다. 위 셋을 먼저 하고, 에이전트를 외부 입력에 노출하는 단계가 되면 그때 붙이는 편이 낫다.

## 참고

- [FEB-30 — Beyond Downstream QA: Rethinking How We Evaluate Agent Memory](https://arxiv.org/abs/2610.22209v1){:target="_blank"} (저장소 미공개)
- [Shortcut Sensitivity in Agent-Safety Evaluations](https://arxiv.org/abs/2610.22243v1){:target="_blank"}
- [AgentPerturb — Evaluating LLM Agent Robustness under Controlled Interaction Perturbations](https://arxiv.org/abs/2610.21624v1){:target="_blank"}
- [GACC-Bench — Benchmarking Goal-Adaptive Context Compression Ability across Large Language Models](https://arxiv.org/abs/2610.21893v1){:target="_blank"}
