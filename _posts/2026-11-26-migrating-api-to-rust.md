---
layout: post
comments: true
title: "운영 중인 API를 검증기로 쓰기: Anthropic의 Messages API Rust 이전에서 가져올 검증 루프"
description: "Anthropic이 Messages API를 Python에서 Rust로 옮기면서 쓴 검증 방법을 정리한다. 테스트 통과율과 운영 트래픽 일치율이 왜 벌어졌는지, 본문을 읽지 않고 차이를 기록하는 체크포인트와 지문, 에이전트에게 맡긴 안쪽·바깥쪽·메모리 루프, 사람 검수를 어디에 두었는지를 원문 기준으로 적는다."
img: migrating-api-to-rust_title.webp
date: 2026-11-26 19:00:00 +0900
last_modified_at: 2026-11-26 19:00:00 +0900
tags: [llm, llm-agent, anthropic, claude, rust, migration, verification, shadow-traffic, agent-loop]
related: llm-agent
categories: dev
source_url: https://www.anthropic.com/research/migrating-api-to-rust
source_date: 2026-11-25
---

Anthropic이 Messages API를 Python에서 Rust로 옮긴 과정을 공개했다. 코드를 누가 썼는지는 이 글의 중심이 아니다. Claude가 핵심 요청 경로의 첫 버전을 약 일주일 만에 썼다고 하고 나머지 분량은 전부 그 결과를 어떻게 믿을 수 있게 만들었는지에 쓴다. 운영 중인 기존 API를 정답지로 삼아 새 구현과 비교하고 차이가 나오면 에이전트가 고치게 한 구조다. 에이전트에게 코드 수정을 맡기는 환경이라면 규모와 상관없이 가져올 수 있는 설계가 여럿 있어 그 부분을 중심으로 정리한다. (이 글의 초안은 Orca Bots 가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 원문의 주장은 병목이 코드 생성에서 검증으로 옮겨 갔다는 것이다. Rust 구현은 4월에 테스트의 약 98%를 통과했지만 운영 트래픽에서는 입력의 약 4분의 1만 기존 구현과 일치했다. Anthropic은 운영 API 자체를 검증기로 삼았다. 모델은 한 번만 호출하고 그 출력을 Rust 쪽에 재생했고, 요청 본문은 읽지 않은 채 차이의 위치와 크기만 기록했다. 그 기록을 에이전트에게 넘기고 샌드박스에서 초 단위로 도는 안쪽 루프, 섀도 트래픽으로 확인하는 바깥쪽 루프, 사람의 교정을 규칙으로 남기는 메모리 루프를 겹쳤다. 사람 검수는 두 루프 사이의 PR 리뷰에 두었다. 결과는 요청당 CPU 약 70% 감소, 첫 토큰까지의 중앙값 시간 약 20% 단축이다.

원문은 2026년 11월 25일 Anthropic이 올린 [Migrating our Messages API to Rust](https://www.anthropic.com/research/migrating-api-to-rust){:target="_blank"}이다. 저자는 Lance Martin, Jake Eaton, Arvin Aminpour, Emma Webb, Bharat Mediratta, Ravi Trivedi로 적혀 있다.

## 왜 옮겼고, 왜 한 번에 옮겼나

Messages API는 2023년에 Python으로 만들어졌다. 원문에 따르면 2024년 초부터 2026년 가을까지 API 트래픽이 약 1,800배 늘었고 요청마다 검증, 프롬프트 변환, 토큰화, 안전 분류기 검사를 거치고 응답 토큰을 텍스트, thinking, 도구 호출로 분류하는 단계마다 Python의 오버헤드가 붙었다.

처음에는 통상의 조언대로 느린 부분만 다시 썼다. 모델 출력을 파싱하는 코드를 Rust로 옮기고 Python에서 호출했더니 API의 CPU 사용량이 약 8%에서 10% 줄었다. 대신 응답의 토큰 하나가 Python과 Rust 사이를 세 번 건너게 되어 새 오버헤드가 생겼고 원문은 이 이득이 늘어난 복잡도를 정당화하지 못했다고 평가한다.

전면 재작성을 피하던 이유는 두 가지였다. 테스트가 운영 동작을 전부 덮는다는 보장이 없고 들어가는 노력이 너무 크다는 것이다. 원문은 뒤쪽 이유가 빠르게 약해졌다고 쓴다. 2026년 5월 기준으로 Anthropic이 병합한 코드의 80% 이상을 Claude가 쓰고 있었다. 남은 것은 앞쪽 이유, 즉 옮긴 결과가 맞는지 어떻게 확인하느냐였다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"This allowed us to ship an API on Rust that uses about 70% less CPU per request and cuts median time to first token by about 20% without ever taking the service offline or freezing feature development."</blockquote><blockquote>"Verification, not generation, had become the bottleneck."</blockquote></details>

## 테스트 통과율을 믿을 수 없었던 이유

원문은 "맞다"를 정의하는 방법을 셋으로 나눈다. 규칙으로 쓸 수 있는 경우(수학 정리와 증명 보조기), 예시 묶음으로 정의하는 경우(테스트), 기존 시스템을 오라클로 삼는 경우다. Messages API 이전에는 앞의 둘이 충분하지 않았다.

기존 통합 테스트는 약 700개였다. 문제는 두 가지였다.

- **테스트가 Python 구현에 묶여 있었다.** 테스트는 Python API를 테스트 프로세스 안에서 띄우고 과금, 속도 제한 같은 서비스를 Python 메모리 안에서 목으로 대체했다. 구현을 옮기려면 테스트도 같이 바꿔야 했고 그 일도 Claude가 도왔다. 그러면 통과는 코드가 맞다는 뜻일 수도, 틀린 코드를 통과시키도록 테스트가 바뀌었다는 뜻일 수도 있다. 실제로 옮긴 테스트를 감사했더니 약해진 단언이 100개 넘게 나왔다.
- **테스트가 불완전했다.** 원문의 계산으로 이진 옵션 20개만으로도 조합이 100만 개를 넘고 건당 1초로 잡아도 모델 하나에 12일이 걸린다. 운영 서비스에서 무엇이 맞는 동작인지는 테스트와 문서만큼이나 실제 동작에 달려 있다고 원문은 쓴다.

그 결과가 통과율과 일치율의 간격이다. Rust API는 4월에 테스트의 약 98%를 통과했지만 운영 트래픽에서는 입력의 약 4분의 1만 일치했다. 원문이 드는 예는 JSON 도구 호출의 이스케이프다. Python은 이모지 같은 일부 문자를 텍스트 코드로 바꿔 쓰는데 Rust는 문자를 그대로 써서 두 프롬프트가 몇 바이트 달랐다. 또 테스트 묶음에서는 사례 하나가 한 번만 세어지지만 실제 트래픽에서는 흔한 버그 하나가 걸리는 요청마다 불일치를 만든다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"We saw this happen: when we audited the ported tests, we found more than 100 weakened assertions."</blockquote><blockquote>"Our Rust API passed about 98% of its tests in April, yet only about a quarter of inputs matched on production traffic (Figure 6)."</blockquote></details>

에이전트에게 구현과 테스트를 함께 고치게 하는 환경이라면 이 대목이 먼저 점검할 부분이다. 통과율만 보면 약해진 단언은 보이지 않는다.

## 운영 API를 검증기로 만들기

같은 요청을 옛 코드와 새 코드에 모두 흘려 결과를 비교하는 방식을 원문은 섀도잉이라고 부른다. 흔한 섀도잉 도구는 요청을 두 번 실행해도 안전하다거나 서비스가 결정적이라고 가정하는데, 여기서는 둘 다 성립하지 않았다. 요청은 고객의 한도와 요금에 한 번만 잡혀야 하고 Claude는 같은 입력에도 응답이 달라질 수 있다.

Anthropic의 해법은 모델을 한 번만 호출하고 그 출력을 Rust 쪽에 재생하는 것이다. 두 구현이 같은 모델 출력을 놓고 일하게 만든 다음, 모델 호출 앞뒤의 모든 단계를 비교했고 결과는 정확히 같아야 했다. 원문이 이유로 드는 것은 Claude가 거의 어떤 프롬프트든 받아들인다는 점이다. 보통의 시스템에서는 잘못된 입력이 어딘가에서 오류를 내지만 여기서는 프롬프트 일부가 빠져도 요청이 성공하고 응답만 조금 달라진다.

두 번째 제약은 프라이버시다. 사용자 요청과 응답을 읽지 않고 작업해야 했으므로 텍스트 대신 차이의 형태를 기록했다.

- **체크포인트**: 두 구현은 같은 단계(요청 수락 또는 거부, 설정 해석, 프롬프트 구성)를 거친다. 단계마다 결과를 비교하고 프롬프트 안에서는 조각별로 비교한다. 처음 달라진 지점이 이전 코드가 운영 코드와 어긋난 대략의 위치다.
- **지문(fingerprint)**: 차이가 프롬프트의 어디서 시작하는지와 두 버전의 길이 차이를 기록한다. 체크포인트로 얻은 단계, 조각 정보와 합치면 버그 하나가 영향을 준 요청마다 같은 지문을 남기는 경향이 있었고 지문으로 묶어 수정 순서를 정했다.

원문이 보여 주는 기록 한 건은 아래와 같다. 값은 원문이 지어낸 예시다.

```text
step: build prompt ← where in the code
piece: 2 ← where in the prompt
first diff: byte 17 ← fingerprint
python: 190 bytes ← fingerprint
rust: 190 bytes ← fingerprint
```

<details class="evidence"><summary>원문 근거</summary><blockquote>"To address this, we ran the model once and replayed the model output to Rust."</blockquote><blockquote>"So instead of looking at text, we recorded the shape of each difference, such as where it happened in the service and how big it was."</blockquote></details>

## 에이전트에게 맡긴 세 겹의 루프

버그 하나가 거의 같은 차이 기록을 수십만 건 만들 수 있다. 그래서 기록을 건건이 에이전트에게 보내지 않았다. 10분에서 30분마다 스크립트가 어떤 필드가 달랐는지로 기록을 묶고 새로운 종류가 나왔을 때만 조정 역할의 Claude 에이전트에게 알렸다. 조정 에이전트는 새 종류마다 작업 에이전트 하나를 배정하고 공유 Slack 채널에 스레드를 만들어 엔지니어가 지켜보다가 필요할 때 방향을 잡을 수 있게 했다.

**안쪽 루프.** 4월에 처음 시도했을 때는 에이전트가 기록만 보고 원인을 추측해 PR을 열었고 리뷰와 병합을 거쳐 스테이징에 배포한 뒤에야 차이가 사라졌는지 알 수 있었다. 가설 하나에 리뷰와 배포가 필요해 피드백에 몇 시간에서 며칠이 걸렸다. 그래서 에이전트마다 두 코드베이스와 로컬 검사를 갖춘 클라우드 샌드박스를 줬다. 입력 검사는 요청 하나를 받아 두 버전이 모델에 보낼 프롬프트를 각각 만들고 차이를 보여 준다. 모델을 호출하지 않으므로 비교가 1초 안에 끝난다. 나중에 가짜 모델을 상대로 도는 출력 검사를 더했다. 도구 호출 같은 각본 응답을 두 버전에 통과시켜 결과를 비교한다.

에이전트는 지문만 볼 수 있고 그 뒤의 요청은 볼 수 없다. 그래서 에이전트는 원인을 가정하고 차이를 재현할 요청이나 모델 응답을 스스로 써서 두 버전에 돌린다. 결과가 같으면 가정이 틀린 것이고, 기록과 같은 방식으로 달라지면 두 버전이 일치할 때까지 Rust를 고친 뒤 옛 코드에서 실패하고 새 코드에서 통과하는 테스트를 추가한다.

**바깥쪽 루프.** 수정은 PR 리뷰를 거친 뒤에만 바깥쪽 루프로 넘어간다. 바깥쪽 루프는 수정이 들어간 Rust 배포와 들어가지 않은 배포를 비교한다. 공정하게 비교하려고 요청 본문 대신 어떤 모델과 API 기능(프롬프트 캐싱, 도구, extended thinking 등)을 썼는지를 기록했고 에이전트는 같은 종류의 요청끼리 놓고 그 버그의 지문이 얼마나 자주 나오는지를 본다. 샌드박스에서 통과한 수정이 운영에서 여전히 차이를 보이면 작업은 안쪽 루프로 돌아간다. 안쪽 루프는 에이전트가 직접 쓴 요청으로만 돌고 로컬 검증기 설정이 운영과 늘 같지는 않기 때문이다.

**메모리 루프.** 엔지니어가 같은 교정을 두 번 하지 않도록 리뷰 결과와 에이전트의 작업 결과를 기록해 뒤따르는 모든 에이전트에게 적용했다. 원문이 드는 사례는 세 가지다.

- 한 에이전트가 Python의 동작을 추측으로 옮긴 뒤, 엔지니어가 Python이 한 그대로 옮긴다는 것을 규칙으로 만들었다. 다른 에이전트가 기존 코드를 그 규칙에 비춰 보고 추측에 기댄 곳을 일곱 군데 더 찾았다.
- 6월에는 에이전트가 팀의 리뷰 댓글 약 60개를 훑어 반복되는 문제를 묶고 규칙을 제안했고 엔지니어가 고치고 승인해 CLAUDE.md에 넣었다.
- 일부 교훈은 영구 검사로 바꿨다. Rust가 Python에서 벗어나면 실패하는 테스트, 리뷰어가 계속 지적하던 패턴을 막는 린트, Python 코드를 바꾸는 사람은 Rust에도 대응하는 변경을 하거나 예외를 요청하게 하는 병합 검사다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Rather than send each to an agent, we ran a script every 10 to 30 minutes that grouped them by which fields differed, then alerted a coordinating Claude agent only when it saw a new kind of record."</blockquote><blockquote>"Because this check never actually called the model, comparing the versions took under a second."</blockquote><blockquote>"This included new tests that failed when Rust drifted from Python, lints that banned patterns reviewers kept flagging, and later a merge check that required anyone changing the Python code to make a matching change in Rust or request an exemption."</blockquote></details>

## 검증기가 답하지 못하는 질문

원문에서 가장 구체적인 실패 사례는 웹 검색이다. 5월 중순에 남아 있던 큰 차이 가운데 하나가 웹 검색에서 나왔다. 운영 API는 웹 검색 도구의 설명을 내부 서비스에서 가져오는데 Rust API는 아직 그 서비스를 호출하지 않아 프롬프트에서 도구 설명이 빠져 있었다. 에이전트는 그 텍스트를 Rust가 읽을 수 있는 파일로 복사하는 지름길을 제안했다. 프롬프트는 일치했고 로컬 검증기도 통과했다. 엔지니어는 이 수정을 거절했다. 출시 전에 Rust가 어차피 그 서비스를 호출해야 하고 그때 이 우회로는 버려지기 때문이다.

원문의 해석은 이렇다. 검증기는 출력의 일치에 보상을 줬고 에이전트는 일치를 만드는 가장 값싼 방법을 찾았다. 그래서 사람 검수를 안쪽 루프와 바깥쪽 루프 사이에 뒀다. 샌드박스 안에서는 틀린 추측의 비용이 몇 초이고 사용자에게 영향이 없으니 에이전트가 혼자 반복하게 두고 엔지니어는 추측이 아니라 제안된 수정만 본다. 그리고 검증기가 답할 수 없는 질문, 예를 들어 두 구현의 차이가 버그인지 의도한 변경인지를 판단한다.

거절은 메모리 루프로 이어졌다. 에이전트는 그 결정과 이유, 올바른 수정(Rust가 도구 설명을 제공하는 서비스를 호출하게 하는 것)을 메모로 남겼다. 그 호출을 만드는 데 몇 주가 걸렸고 그 사이 같은 버그가 새 지문의 기록을 계속 만들었지만, 에이전트들은 메모를 확인하고 알려진 버그로 연결했을 뿐 우회로를 다시 쓰지 않았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"A verifier can tell an agent whether the outputs matched, but not whether the fix is good enough to ship."</blockquote></details>

## 개발 환경에 옮길 때 가져올 것

원문의 수치는 Anthropic 규모의 이야기지만 구조는 작은 환경에도 그대로 들어간다. 원문이 끝에 정리한 교훈을 실행 순서대로 다시 놓았다.

1. **무엇이 정답지인지부터 정한다.** 규칙, 테스트, 기존 시스템 가운데 어느 것이 "맞다"를 정의하는지 정한다. 테스트를 정답지로 쓸 생각이라면 에이전트가 그 테스트를 고칠 수 있는지 확인한다. 고칠 수 있다면 통과는 증거로서 약해진다.
2. **검증 신호가 위치를 가리키게 만든다.** 원문은 이 프로젝트의 노력 가운데 많은 부분이 신호를 만드는 데 들어갔다고 쓴다. 맞다 틀리다만 알려 주는 검사는 에이전트에게 고칠 곳을 알려 주지 못한다. 단계별 비교로 처음 어긋난 지점을 찍고 같은 원인을 한 묶음으로 볼 수 있는 표식을 남긴다.
3. **빠른 검사를 에이전트 손에 쥐여 준다.** 리뷰와 배포를 거쳐야 결과를 아는 구조에서는 가설 하나에 몇 시간이 든다. 모델 호출 없이 초 단위로 끝나는 로컬 검사를 먼저 만든다.
4. **느린 검사로 빠른 검사의 구멍을 메운다.** 빠른 검사는 에이전트가 스스로 만든 입력으로만 돈다. 실제 트래픽이나 실제 데이터로 확인하는 단계를 따로 두고 실패하면 빠른 검사로 되돌린다.
5. **사람 검수는 두 검사 사이에 둔다.** 사람이 볼 것은 추측이 아니라 검사를 통과한 수정이다. 볼 질문도 일치 여부가 아니라 이 수정이 남길 만한 것인지다.
6. **교정을 규칙과 검사로 남긴다.** 리뷰에서 한 번 한 말은 에이전트가 읽는 규칙 파일에 넣고 가능하면 테스트나 린트로 바꾼다.

이 블로그의 초안 파이프라인도 같은 모양을 작게 쓰고 있다. 초안은 Orca Bots 가 쓰고 저장해 둔 원문과 초안의 수치, 식별자, 인용문을 문자열로 대조하는 검사를 통과한 뒤 Claude와 사람의 검수로 넘어간다. 검사가 잡는 것은 원문에 없는 문자열이고, 글이 실을 만한지는 검수 단계가 판단한다. 검수에서 나온 교정은 초안 작성 규칙 파일에 사고 사례와 함께 쌓인다. 추출 모델을 에이전트의 검증 단계로 쓰는 구성은 [추출 모델을 에이전트 검증기로 쓰기]({{site.baseurl}}/dev/2026/11/25/extraction-models-as-verifiers.html)에서 다뤘다.

원문은 사람의 위치가 바뀐 과정도 적는다. 처음에는 엔지니어가 안쪽 루프 안에서 직접 Claude를 돌리며 차이를 쫓았고, 그다음에는 루프 사이에서 Slack으로 에이전트를 조정하고 수정을 리뷰했고, 시스템이 성숙하면서 에이전트가 따르는 규칙과 검사를 쓰는 쪽으로 옮겨 갔다. 공개 트래픽이 Rust로 넘어간 것은 2026년 9월이다.

## 읽을 때 감안할 것

- 수치는 전부 Anthropic이 자기 서비스에서 잰 값이다. CPU, 메모리, 첫 토큰 시간의 추이는 그래프로 제시되고 원문 캡션에 따르면 CPU는 한 리전, 메모리는 운영 전체, 첫 토큰 시간은 더 짧은 기간의 Claude Code 요청을 기준으로 한다. 그래프 안의 개별 값은 읽지 못했으므로 본문에 적힌 약 70%와 약 20%만 옮겼다.
- 섀도 비교 시스템, 샌드박스, 조정 에이전트의 코드나 설정은 공개되지 않았다. 원문에서 가져올 수 있는 것은 구조와 판단 기준이다.
- 원문은 일치율 그래프에서 비교 대상이 시간에 따라 바뀌었고 오해를 부를 수 있는 엿새를 뺐다고 캡션에 적는다.

## 참고

- [Migrating our Messages API to Rust](https://www.anthropic.com/research/migrating-api-to-rust){:target="_blank"} (Anthropic, 2026-11-25)
- [Messages API 문서](https://platform.claude.com/docs/en/build-with-claude/working-with-messages){:target="_blank"}
- [Claude Code 메모리(CLAUDE.md) 문서](https://code.claude.com/docs/en/memory){:target="_blank"}
- [Scientist: Measure Twice, Cut Over Once](https://github.blog/2016-02-03-scientist/){:target="_blank"} (원문이 운영 데이터가 "유일하게 참된 테스트"라는 대목에 링크한 GitHub 글)
- [Diffy: Testing services without writing tests](https://blog.x.com/en/engineering/diffy-testing-services-without-writing-tests){:target="_blank"} (원문이 흔한 섀도잉 도구로 링크한 글)
