---
layout: post
comments: true
title: "생성 대신 채점 — SGLang /v1/score로 오픈 LLM을 분류 엔진으로 쓰기"
description: "Avi Chawla의 'Build your own Jev (100% local)' 해설. 고정된 선택지 분류에서 텍스트를 생성하지 않고 첫 next-token 로짓만 읽어 확률 분포를 얻는 방식, SGLang /v1/score 사용법, 단일 토큰 레이블 제약, 그리고 보정 없이는 확률이 정확도가 아니라는 한계를 정리한다."
img: sglang_scoring_title.webp
date: 2026-09-28 23:40:00 +0900
last_modified_at: 2026-09-29 22:30:00 +0900
tags: [sglang, scoring, classification, local-llm, qwen, inference, logits, jev, llm-serving, llm]
related: llm-serving
categories: dev
source_url: https://blog.dailydoseofds.com/p/build-your-own-jev-100-local
source_date: 2026-09-22
---

Avi Chawla(@_avichawla)가 [트윗](https://x.com/_avichawla/status/2101563610644496464){:target="_blank"}으로 소개한 글 [「Build your own Jev (100% local)」](https://blog.dailydoseofds.com/p/build-your-own-jev-100-local){:target="_blank"}을 읽고 정리했다. 분류처럼 답이 정해진 작업에서 LLM에게 텍스트를 만들게 하지 않고, 첫 출력 벡터의 로짓만 읽어 확률 분포를 얻는 방식이다. 재학습이 필요 없고 SGLang과 오픈 Qwen 모델만 있으면 된다. (이 글은 Claude가 원문을 직접 확인하고 작성했다.)

<!--more-->

> **TL;DR:** 선택지가 고정된 작업에서는 생성 대신 **채점(scoring)** 을 쓸 수 있다. SGLang의 `/v1/score`에 프롬프트와 레이블 토큰 ID를 넘기면, 모델은 포워드 패스 한 번을 돌고 지정한 어휘 위치의 로짓을 읽어 softmax를 씌운 확률 분포를 돌려준다. 자기회귀 디코딩 루프가 없다. 제약은 **레이블이 단일 토큰이어야 한다**는 것(Qwen2.5 기준 A=32, B=33, C=34). 한계도 분명하다. 이건 Jev의 **추론 경로만** 재현한 것이고 학습·보정은 빠져 있으며, 보정 없는 확률값은 정확도가 아니다.

## 1. 기존 LLM 사용과 무엇이 다른가

시험에 비유하면 차이가 분명해진다.

**기존 방식은 답을 "써내게" 한다.** 객관식 문제를 주고도 답안지에 글자를 적게 시키는 셈이다. 그래서 세 가지 문제가 따라온다. 글씨를 다 쓸 때까지 기다려야 하고("주차 민원"이면 여섯 글자를 순서대로 만들어야 한다), 표기가 흔들릴 수 있고("주차민원"인지 "주차 민원"인지), 보기에 없는 답을 적어 버릴 수도 있다. 그래서 받아 적은 답을 다시 파싱하고 목록과 대조하는 후처리가 필요하다.

**채점 방식은 보기마다 확신도를 매기게 한다.** 답을 쓰게 하는 대신 "A는 얼마나 그럴듯한가, B는, C는"만 묻는다. 결과는 `A 0.72 · B 0.21 · C 0.07` 같은 확률 분포다. 글자를 만들 필요가 없으니 한 번에 끝나고, 보기 밖으로 나갈 길이 없으니 파싱 실패가 구조적으로 불가능하다.

여기서 핵심은, **모델이 원래 이 점수를 늘 계산하고 있었다**는 점이다. 평소 생성에서도 모델은 다음 글자를 정하기 전에 후보마다 점수를 매긴다. 다만 그중 하나를 골라 이어 쓰고, 그 과정을 글이 끝날 때까지 반복할 뿐이다. 채점은 없던 능력을 새로 붙이는 게 아니라 **첫 단계에서 멈춰 그 점수를 그대로 읽어 오는 것**이다. 재학습이 필요 없는 이유가 여기 있다.

## 2. 용어 풀이

이 글에 반복해서 나오는 말들이다.

- **토큰(token)**: 모델이 글을 다루는 최소 단위. 글자도 단어도 아닌 그 중간쯤이다. 영어 `A`는 토큰 하나지만 한국어 "주차 민원"은 여러 개로 쪼개진다. 뒤에 나오는 "레이블은 단일 토큰이어야 한다"는 제약이 여기서 나온다.
- **어휘(vocabulary)**: 모델이 아는 토큰의 전체 목록. 모델마다 다르고 크기는 보통 십만 개 단위다. 토큰마다 고유 번호가 있어서 Qwen2.5에서는 `A`가 32번, `B`가 33번, `C`가 34번이다.
- **로짓(logit)**: 모델이 다음 토큰을 정할 때, 마지막 층이 **어휘의 모든 토큰에 대해 하나씩 매기는 원점수**. 아직 확률이 아니라 가공 전 숫자다. 이 글의 방식은 십만 개가 넘는 이 점수 중에서 32·33·34번 **세 자리만 꺼내 쓴다**.
- **softmax**: 원점수 묶음을 합이 1인 확률로 바꾸는 계산. 세 개만 꺼내서 softmax를 씌우면 그 셋 사이의 상대적 확률이 나온다. `apply_softmax: True`가 이 일을 한다.
- **포워드 패스(forward pass)**: 입력을 모델에 한 번 통과시키는 것. 생성은 토큰 수만큼 이걸 반복하고, 채점은 한 번으로 끝난다.
- **자기회귀(autoregressive)**: 방금 만든 토큰을 다시 입력에 넣어 다음 토큰을 만드는 방식. 지금 쓰는 LLM 대부분이 이렇게 동작하고, 답이 순차적으로 타이핑되듯 나오는 이유다.

## 3. 무엇을 하는 방식인가

원문이 재현 대상으로 삼은 Jev는 애플리케이션이 질의와 함께 **허용된 답 목록**을 넘기면 모델이 각 답의 점수를 돌려주는 구조다. 티켓 분류라면 티켓 본문과 세 개의 허용 답을 같이 주고, 모델은 세 답 각각에 점수를 매긴다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"the application provides the ticket/query and the three allowed answers to Jev. The model then returns a score for each answer."</blockquote></details>

여기서 중요한 건 **생성을 하지 않는다**는 점이다. 표준 생성은 없던 텍스트를 만들어 내지만, 채점은 미리 정한 위치의 어휘 벡터를 읽어 고정된 집합에서 고를 뿐이다.

## 4. 구조화 출력과 무엇이 다른가

JSON 스키마를 강제하는 구조화 출력(structured output)과 헷갈리기 쉽다. 둘의 차이는 결정적이다. 구조화 출력은 **여전히 토큰을 순차 생성한다**. 중괄호, 필드명, 값이 모두 디코딩 단계를 거친다. 채점은 자기회귀 생성 없이 특정 토큰 위치의 로짓을 읽는다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Structured output still generates tokens sequentially (braces, field names, values). Scoring reads logits at specific token positions without autoregressive generation."</blockquote></details>

답이 레이블 밖으로 나갈 수 없다는 점도 실무에서는 크다. 생성 방식은 카테고리명을 모델이 직접 써야 해서 표기가 흔들리거나 목록에 없는 값이 나올 수 있고, 그때마다 파싱과 후처리가 필요하다. 채점에는 그 실패 경로 자체가 없다.

## 5. SGLang `/v1/score` 사용법

엔드포인트가 하는 일은 네 단계다.

1. 프롬프트를 모델에 통과시킨다
2. 지정한 토큰 ID들의 로짓을 읽는다
3. 그 위치들에 대해서만 softmax를 적용한다
4. 확률 분포를 반환한다

```python
requests.post(f"{BASE_URL}/v1/score", json={
    "model": MODEL,
    "query": prompt,
    "items": [""],
    "label_token_ids": [32, 33, 34],   # A, B, C
    "apply_softmax": True,
})
```

`query`에는 답이 들어갈 자리 **직전까지**의 프롬프트를 넣는다. 모델이 그다음에 무엇을 낼지 예측한 첫 벡터에서 `label_token_ids`에 해당하는 위치만 읽기 때문이다.

## 6. 걸리는 제약: 레이블은 단일 토큰

모든 선택지가 **같은 출력 위치의 어휘 항목 하나**로 표현돼야 한다. 여러 토큰으로 쪼개지는 레이블은 한 위치에서 점수를 매길 수 없다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"single-token labels avoid that problem. Every option is represented by one vocabulary entry at the same output position."</blockquote></details>

원문은 Qwen2.5-0.5B-Instruct로 토큰 ID를 확인한다. `A` → 32, `B` → 33, `C` → 34다. 실제로 쓸 때는 **쓰려는 모델의 토크나이저로 직접 확인**해야 한다. 모델마다 어휘가 다르므로 이 숫자를 그대로 옮기면 안 된다.

한국어 업무에 붙일 때 이 제약이 바로 문제가 된다. "주차 민원"이나 "소음" 같은 카테고리명은 단일 토큰이 아니다. 그래서 실제 구성은 이렇게 된다.

- 레이블은 `A`/`B`/`C` 같은 단일 토큰 기호로 둔다
- 기호와 카테고리의 매핑은 **프롬프트 안에 적는다**
- 응답으로 받은 확률 분포를 애플리케이션에서 카테고리로 되돌린다

선택지가 많아지면 단일 토큰으로 남는 기호를 확보할 수 있는지부터 확인해야 한다.

## 7. 얼마나 빨라지나

원문은 Qwen3-4B-Instruct-2507로, 요청당 최대 32토큰을 생성하는 표준 생성과 채점을 동시 배치로 비교한다. 영상으로는 채점이 눈에 띄게 먼저 끝난다고 설명하지만, **지연 시간이나 처리량의 구체적인 수치는 제시하지 않는다**. 속도 우위의 근거는 벤치마크 수치가 아니라 구조다. 생성은 토큰 수만큼 포워드 패스를 반복하지만 채점은 한 번으로 끝난다.

직접 적용을 검토한다면 자기 환경에서 측정해 보는 편이 낫다. 모델 크기, 프롬프트 길이, 배치 크기에 따라 이득의 폭이 달라진다.

## 8. 한계 — 원문이 직접 밝힌 것

원문은 두 가지를 명시한다. 이 방식을 실무에 올릴 때 가장 중요한 대목이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"this article recreates the inference path, not the complete Jev system. Jev also includes training and calibration work that a scoring endpoint does not provide."</blockquote></details>

첫째, **추론 경로만 재현했다**. Jev의 학습과 보정(calibration) 작업은 스코어링 엔드포인트가 주지 않는다. 둘째, **라벨링된 평가 데이터 없이는 확률값이 실제 정확도를 뜻하지 않는다**. 0.9가 나왔다고 90% 맞는다는 보장이 없다.

이 두 가지를 합치면 "재학습 없이"라는 말의 범위가 분명해진다. *분류기를 새로 학습시킬 필요가 없다*는 뜻이지, *평가셋 없이 바로 운영할 수 있다*는 뜻이 아니다.

## 9. 적용 관점 — 고정 분류 업무

민원 분류처럼 **분류 체계가 고정돼 있고 라우팅·에스컬레이션 판단에 확신도가 필요한 업무**에 형태가 잘 맞는다. 확률 분포가 나오므로 "상위 확률이 임계값 미만이면 사람에게 넘긴다" 같은 운영 규칙을 바로 얹을 수 있다.

다만 순서를 틀리면 안 된다. 임계값을 정하려면 그 확률이 얼마나 믿을 만한지 알아야 하고, 그건 라벨링된 평가셋으로 보정한 뒤에야 나온다. 실제 작업량의 대부분은 스코어링 연동이 아니라 **평가셋 구축과 보정**이 될 가능성이 크다. 벤치마크 점수와 실제 운영 성능이 어긋나는 문제는 [토스의 자체 LLM 벤치마크 사례]({{site.baseurl}}/llm/ai-evaluation/2026/09/21/toss-benchmark.html)에서도 같은 형태로 나타났다.

로컬 실행 쪽 준비는 [RTX PC 로컬 LLM 가이드]({{site.baseurl}}/dev/2026/09/09/rtx-local-llm-guide.html)나 [오픈소스 에이전트 스택 정리]({{site.baseurl}}/tools/2026/09/11/opensource-agent-stack.html)를 참고할 수 있다.

## 10. 참고 자료

- 원문: [Build your own Jev (100% local)](https://blog.dailydoseofds.com/p/build-your-own-jev-100-local){:target="_blank"} — Avi Chawla, 2026-09-22
- 소개 트윗: [@_avichawla](https://x.com/_avichawla/status/2101563610644496464){:target="_blank"}
- [SGLang 공식 문서](https://docs.sglang.ai/){:target="_blank"}
- 관련 글: [토스 자체 LLM 벤치마크]({{site.baseurl}}/llm/ai-evaluation/2026/09/21/toss-benchmark.html), [RTX PC 로컬 LLM 시작하기]({{site.baseurl}}/dev/2026/09/09/rtx-local-llm-guide.html), [오픈소스로 AI 에이전트 구축하기]({{site.baseurl}}/tools/2026/09/11/opensource-agent-stack.html)
