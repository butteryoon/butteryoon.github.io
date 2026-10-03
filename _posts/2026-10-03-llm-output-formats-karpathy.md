---
layout: post
comments: true
title: "LLM 출력을 어떻게 읽을까 — Karpathy의 형식 사다리 네 단계"
description: "Karpathy가 제안한 LLM 출력 형식 네 단계(ASD-STE100 통제 언어 → 다이어그램 → HTML → 설명 영상)를 정리하고, 형식을 올리면 이해는 쉬워지지만 검증은 오히려 어려워진다는 반대편을 실제 사고 사례로 덧붙인다."
img: llm_output_formats_title.webp
date: 2026-10-03 12:40:00 +0900
last_modified_at: 2026-10-03 13:25:00 +0900
tags: [karpathy, asd-ste100, controlled-language, prompting, documentation, llm-agent, llm]
related: llm-agent
categories: dev
source_url: https://x.com/karpathy/status/2105819303471976479
source_date: 2026-10-02
---

Andrej Karpathy가 10월 2일 올린 [트윗](https://x.com/karpathy/status/2105819303471976479){:target="_blank"}을 정리했다. LLM에게 무엇을 시킬지가 아니라 **그 결과를 어떻게 읽을 것인가**에 관한 이야기다. 출력 형식을 네 단계로 올려 가며 제안하는데, 각 단계 사이에 "But even better:"를 붙여 사다리를 만든다. (이 글은 Claude가 원문을 직접 확인하고 작성했다.)

<!--more-->

> **TL;DR:** 네 단계는 글(ASD-STE100 통제 언어로 요청) → 다이어그램 → HTML 웹페이지 → 설명 영상이다. 전제는 따로 있다. LLM이 실무를 더 많이 가져갈수록 사람의 일은 **감독과 이해** 쪽으로 올라가고, 그래서 "출력을 읽는 비용"이 새 병목이 된다는 것이다. 여기에 한 가지를 덧붙여 둘 만하다. 형식을 올리면 이해는 쉬워지지만 **검증은 어려워진다.** 틀린 내용이 깔끔한 도표에 담기면 더 그럴듯해 보인다.

## 1. 네 단계

<details class="evidence"><summary>원문 근거</summary><blockquote>"We'll be spending a lot more time trying to understand the outputs of language models."</blockquote></details>

**① 글 — ASD-STE100으로 요청하기.** 항공 정비 문서용으로 만들어진 통제 언어 규격 이름을 그대로 부른다. 규격이 꽤 가혹해서, Karpathy는 완충 장치도 함께 제안한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Ask your LLM to explain something in ASD-STE100... Sometimes I've tried to soften it a bit e.g. ask for '80% of the way to ASD-STE100' because the spec is quite stringent."</blockquote></details>

**② 다이어그램.** 글 대신 그림을 요청한다. 처리하고 뜯어보고 이해하기가 훨씬 쉽다는 이유다.

**③ 웹페이지.** "HTML로 출력해 줘"라고 하면 인터랙티브한 페이지가 나온다. 프런트엔드 쪽 실력이 올라와서 애니메이션까지 붙는다.

**④ 설명 영상.** 가장 기대를 거는 형식이다. "3b1b 스타일로 X 설명 영상 만들어 줘, 내레이션은 내 ElevenLabs API 키 써서"처럼 요청한다. API 키가 없으면 로컬 연산을 쓰는 무료 대안을 LLM에게 물어보라고 덧붙인다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The output format I am most bullish on is fully custom / bespoke explainer videos generated on any arbitrary topic... This is actually starting to work!"</blockquote></details>

## 2. ASD-STE100이 무엇이고 왜 통하나

유럽 항공업계 요청으로 AECMA가 1979년에 착수해 1986년 첫 가이드를 낸 **통제 언어(controlled language)** 규격이다. 지금은 ASD 산하 STEMG(1983년 결성)가 관리하고, 2025년 1월판 기준 **작성 규칙 53개와 승인 단어 약 900개**로 이뤄진다.

| 항목 | 내용 |
|---|---|
| 핵심 원칙 | 한 단어는 한 뜻, 한 품사 |
| 문체 | 능동태, 짧은 문장 |
| 길이 제한 | 절차문 20단어 · 설명문 25단어 · 문단 6문장 |
| 금지어 처리 | 대체어가 지정돼 있다 (`commence`→`START`, `prior to`→`BEFORE`) |
| 동사형 | 진행형(`-ing`)과 완료형은 비승인 |

여기서 재미있는 건 **이게 왜 프롬프트로 잘 먹히는가**다. 스타일 가이드를 직접 써서 프롬프트에 욱여넣는 대신, **모델이 이미 학습한 공개 규격의 이름 하나**를 부르는 방식이다. 토큰을 적게 쓰면서 규칙은 일관되게 적용된다. "80%만"이라는 표현이 통하는 것도 같은 이유다. 모델이 규격의 강도를 알고 있으니 그 강도를 조절해 달라는 요청이 성립한다.

## 3. 트윗에 붙은 이미지가 그 자체로 논증이다

트윗에는 3200×1600 크기의 이미지가 한 장 붙어 있다. **ASD-STE100을 설명하는 다이어그램**이다.

![ASD-STE100 규격을 공학 도면 양식으로 정리한 다이어그램. 문서 구조, 문장 해부, 동사형 표, 사전 항목, 규칙 상한값, 연혁 등 여섯 개 패널로 구성돼 있다]({{site.baseurl}}/assets/img/karpathy_asd_ste100_diagram.webp)

*출처: [@karpathy 트윗](https://x.com/karpathy/status/2105819303471976479){:target="_blank"}에 첨부된 이미지. 본문 설명을 위해 인용했고, 가로 2000px로 줄여 실었다.*

읽는 순서대로 보면 이렇게 짜여 있다.

| 패널 | 내용 |
|---|---|
| A 문서 구조 | 규격이 작성 규칙(9개 절)과 사전 두 부분으로 나뉜다는 것 |
| B 문장 해부 | 같은 지시를 비규격 문장과 STE 문장으로 나란히 놓고, **비승인 단어를 빨간색으로** 표시 |
| C 동사형 | 명령형·현재·과거·미래·부정사·과거분사는 승인, **진행형과 완료형은 비승인** |
| D 사전 항목 | `commence`→`START`, `prior to`→`BEFORE`처럼 금지어와 대체어를 짝지어 제시 |
| E 규칙 상한 | 절차문 20단어, 설명문 25단어, 문단 6문장, 명사 묶음 3단어를 막대로 |
| F 연혁 | 1979년 AECMA 착수 → 1986년 첫 가이드 → 2005년 ASD-STE100 → 현재 무료 배포 |

전체를 테두리 좌표(세로 A~D, 가로 1~8)로 감싼 것까지 항공·기계 도면의 관례를 그대로 따랐다. 규격의 출신을 형식으로 드러낸 셈이다.

여기서 눈여겨볼 건 **②번 제안을 글로 설명하지 않고 그 형식 자체로 보여줬다**는 점이다. "다이어그램이 글보다 낫다"는 주장을 다이어그램으로 제시했으니, 트윗 자체가 자기 주장의 실물 예시가 된다.

다만 한 가지는 짚어 둘 만하다. 이 그림은 **밀도가 높아서 작게 보면 읽히지 않는다.** 위 이미지도 블로그 본문 너비에서는 글자가 뭉개져서, 제대로 보려면 이미지를 따로 열어야 한다. 다이어그램이 글보다 빠른 건 **한 화면에 들어올 때**의 이야기다.

## 4. 진짜 본론은 형식이 아니다

<details class="evidence"><summary>원문 근거</summary><blockquote>"As LLMs get better, they will do more and more of the legwork autonomously, and a lot more of our work will rise up the abstractions into oversight and understanding."</blockquote><blockquote>"as intelligence and code are increasingly abundant, you can ask for large, custom, discardable software artifacts (e.g. web apps, video explainers) that would have never made sense to create before"</blockquote></details>

두 문장에 두 개의 주장이 들어 있다.

하나는 **일의 무게중심 이동**이다. 만드는 일은 모델이 가져가고 사람에게는 감독과 이해가 남는다. 출력 형식에 공을 들이라는 제안이 여기서 나온다. 읽는 쪽이 병목이 됐으니 읽기 쉽게 만들라는 것이다.

다른 하나는 **쓰고 버리는 산출물(discardable artifacts)** 이다. 한 번 쓰고 버릴 도구를 만드는 비용이 임계점 아래로 내려갔다는 관찰인데, 이쪽이 더 실감 난다. 이 블로그를 운영하면서 모델 응답 속도를 재는 벤치마크 스크립트, 색인 상태를 조회하는 진단 스크립트, 사이트맵 전후를 비교하는 스크립트를 썼다. 전부 한 번 쓰고 버렸고, 예전 같으면 만들지 않고 수작업으로 때웠을 것들이다.

## 5. 덧붙일 것 — 형식을 올리면 검증은 어려워진다

사다리를 올라가면 이해는 쉬워진다. 그런데 **틀린 내용도 같이 올라간다.** 매끄러운 도표나 영상에 담긴 오류는 글에 담긴 오류보다 알아채기 어렵다. 형식이 주는 완성도가 내용의 신뢰도로 착각되기 때문이다.

이 블로그에서 실제로 겪은 일이다. 9월 22일, 에이전트가 쓴 초안이 벤치마크 수치를 통째로 지어냈다. 원문에서 그 수치는 **차트 이미지 안에만** 있었고 본문 텍스트에는 없었다. 에이전트는 읽지 못한 값을 채워 넣었는데, 표로 정리돼 있으니 그럴듯해 보였다. 실제 차트를 열어 보니 결론 방향까지 반대였다. 자세한 경위는 [transformers GGUF 지원 글]({{site.baseurl}}/dev/2026/09/22/transformers-gguf-support.html)에 정리해 뒀다.

텍스트였다면 더 빨리 걸렸을 것이다. "표에 숫자가 있다"는 사실 자체가 검증을 건너뛰게 만든다.

그래서 이 사다리는 **내용이 검증된 뒤에** 올라가는 게 맞다. 순서를 바꾸면 오류의 설득력만 키운다. Karpathy의 제안은 *이해를 돕는 방법*이지 *정확성을 보장하는 방법*이 아니고, 원문도 그렇게 주장하지는 않는다.

## 6. 한국어 작업에는 그대로 쓸 수 없다

ASD-STE100은 영어 규격이라 한국어 출력에는 적용되지 않는다. 같은 효과를 보려면 제약을 직접 명시해야 한다. 평어체, 한 문장에 한 주장, 피동 최소화, 문단당 한 주제 같은 것들이다. 이름 하나로 끝나지 않으니 프롬프트가 길어지는 건 감수해야 한다.

다이어그램·HTML·영상 쪽은 언어와 무관하게 그대로 적용된다. 이 글의 타이틀 이미지도 ②번을 따라 만들었다.

Karpathy의 이전 글들과 묶어 읽으면 방향이 보인다. [자율 실험 루프]({{site.baseurl}}/dev/2026/08/30/karpathy-autoresearch-loop.html)는 모델이 실무를 가져가는 쪽이고, [microgpt와 컴파일레이션 패러다임]({{site.baseurl}}/dev/2026/09/01/karpathy-microgpt-compilation-ir.html)은 추상화 층위를 다시 긋는 이야기였다. 이번 트윗은 그래서 사람에게 남는 일이 무엇인지를 다룬다.

## 7. 참고 자료

- 원문 트윗: [@karpathy](https://x.com/karpathy/status/2105819303471976479){:target="_blank"} (2026-10-02)
- [ASD-STE100 공식 사이트](https://www.asd-ste100.org/){:target="_blank"} — 규격 PDF 무료 배포
- [ASD 소개 페이지 — Simplified Technical English](https://www.asd-europe.org/standards-specifications/simplified-technical-english/){:target="_blank"}
- 관련 글: [Karpathy의 Autoresearch Loop]({{site.baseurl}}/dev/2026/08/30/karpathy-autoresearch-loop.html) · [microgpt와 컴파일레이션 패러다임]({{site.baseurl}}/dev/2026/09/01/karpathy-microgpt-compilation-ir.html) · [transformers GGUF 지원]({{site.baseurl}}/dev/2026/09/22/transformers-gguf-support.html)
