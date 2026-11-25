---
layout: post
comments: true
title: "추출 모델을 에이전트 검증기로 쓰기: DAIR.AI가 본 Drex for Extraction과 Nace Extract·Ground API"
description: "DAIR.AI가 추출 모델을 에이전트의 검증 단계로 쓰는 구성을 정리했다. 답이 입력의 정확한 부분 문자열로 돌아오고, 없으면 빈 결과가 나온다는 두 성질을 어디에 끼워 넣는지, 그리고 지금 문서화된 경로인 Nace Extract·Ground API의 요청 형식, 결과 필드, 제한, 가격을 공식 문서로 확인해 적는다."
img: extraction_verifier_title.webp
date: 2026-11-25 19:00:00 +0900
last_modified_at: 2026-11-25 22:40:00 +0900
tags: [llm, llm-agent, dair-ai, nace, drex, extraction-model, verifier, grounding, citation, document-ai]
related: llm-agent
categories: dev
source_url: https://nlp.elvissaravia.com/p/extraction-models-as-verifiers-for
source_date: 2026-11-24
---

DAIR.AI의 Elvis Saravia가 추출 모델을 에이전트의 검증기로 쓰는 구성을 정리한 글을 올렸다. 소재는 Nace가 Drex Max에 추가한 Drex for Extraction이다. 추출 모델은 답을 새로 쓰지 않고 입력 안의 위치를 가리키기 때문에, 에이전트가 낸 주장이 원문에 실제로 있는지 확인하는 단계로 쓸 수 있다는 것이 요지다. 이 글은 Nace가 후원한 글이라고 원문 끝에 적혀 있다. 그래서 수치는 Nace 제품 페이지와 공식 문서를 따로 받아 대조했고 원문이 말하지 않는 불리한 수치도 함께 적는다. (이 글의 초안은 Orca Bots 가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 추출 모델의 쓸모는 두 가지 성질에서 나온다. 텍스트 답이 입력의 정확한 부분 문자열이라 기계적으로 대조할 수 있고 답이 없으면 지어내지 않고 빈 결과를 낸다. 원문은 이 성질을 도구 결과를 컨텍스트에 넣기 전, 메모리에 쓰기 전, 최종 답을 내기 전 세 지점에 끼워 넣자고 제안한다. 다만 Drex Max Locate와 Drex Max Box는 지금 Nace Console의 Experimental 탭에서만 써 볼 수 있고 원문이 API로 안내하는 경로는 Extract다. 인용문이 문서 어디에 있는지 찾는 일은 문서상 Ground가 맡는데, Ground는 위치만 찾을 뿐 주장이 참인지는 판단하지 않는다. 벤치마크는 전부 Nace가 낸 수치이고 박스 정밀도와 FUNSD에서는 VGSE-32B가 앞선다.

원문은 2026년 11월 24일 DAIR.AI 뉴스레터에 올라온 [Extraction Models as Verifiers for Agents](https://nlp.elvissaravia.com/p/extraction-models-as-verifiers-for){:target="_blank"}이다.

## 추출 모델이 검증기가 되는 이유

원문은 추출 모델을 결정 모델과 나란히 놓는다. 결정 모델은 선택지 가운데 무엇을 할지 고르고 추출 모델은 사실이 어디에 있는지 찾는다. 추출 모델이 돌려주는 답은 모두 출처를 가리킨다는 점이 다르다고 원문은 쓴다.

Nace 제품 페이지 기준으로 Drex for Extraction은 두 가지다.

- **Drex Max Locate**: 텍스트와 평문 설명을 받아 정확한 구간을 문자 위치, 신뢰도 점수와 함께 돌려준다. 요청 하나에 질문을 128개까지 넣을 수 있다. 모든 답은 입력의 정확한 부분 문자열이고 답이 텍스트에 없으면 아무것도 돌려주지 않는다.
- **Drex Max Box**: 페이지 이미지나 스캔을 받아 페이지 위의 박스를 신뢰도 점수와 함께 돌려준다. 좌표는 0에서 1,000까지의 페이지 척도다. 의미로 찾기, 정확한 텍스트로 찾기, 일치하는 것 전부 찾기를 지원하고 페이지에 없으면 빈 목록을 낸다.

원문은 이 둘을 Drex Locate, Drex Box라고 줄여 부르지만 제품 페이지의 이름은 Drex Max Locate, Drex Max Box다. 둘 다 Drex Max v1.1 위에 얹는 애드온이고 아래의 Drex Max v1.1은 고정돼 있어 결정 답이 바뀌지 않는다고 제품 페이지는 적는다. Locate를 올린 상태에서 결정 답 105개 중 105개가 같았다는 것이 근거로 제시된다.

검증기로 쓸 때 중요한 것은 "없음"을 돌려주는 동작이다. 원문은 송장의 납기일을 예로 든다. 빈 칸은 빈 채로 남아야 하고 에이전트가 납기일을 지어내면 그 틀린 날짜가 뒤의 모든 단계로 흘러간다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Every text answer is an exact substring of the input, so you can check it."</blockquote><blockquote>"A blank field should stay blank. If an agent invents a due date on an invoice, that wrong date flows into every step after it."</blockquote></details>

결정 모델을 이 블로그에서 다룬 흐름은 [결정 모델 다섯 편 묶어 읽기]({{site.baseurl}}/dev/2026/11/12/decision-model-lineage.html)에 정리돼 있다. 원문이 그리는 조합은 그 연장이다. 송장 하나를 놓고 Drex Box가 합계, 납기일, PO 번호의 위치를 찾고, Drex Max 결정 모델이 승인, 플래그, 사람에게 넘기기 가운데 하나를 선택지별 확률과 함께 고른다. 어느 단계에서든 점수가 낮거나 일치가 없으면 사람에게 간다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"A low score or a no-match at either step goes to a human. The LLM only sees fields that were found and checked."</blockquote></details>

## 하니스의 어디에 끼워 넣는가

원문이 드는 사용처는 네 가지다.

- 계약서 에이전트가 요약을 쓰면, 요약이 나가기 전에 Drex Locate가 각 주장을 계약서와 대조한다. 일치하는 구간이 없는 주장은 플래그된다.
- 리서치 에이전트가 논문을 인용하면, Locate가 인용된 텍스트가 그 논문에 있는지 확인한다. 지어낸 인용이 일찍 걸린다.
- 송장 에이전트가 결제 양식을 채우면, Drex Box가 각 값이 페이지에 있는지 확인한다. 페이지에 없는 필드는 비워 둔다.
- RAG 답변을 구간 하나씩 확인한다. 신뢰도가 낮은 구간이 나오면 에이전트가 답하기 전에 컨텍스트를 더 검색하러 돌아간다.

하니스 안에서는 추출을 에이전트 루프의 기본 단계로 넣자고 한다. 지점은 세 곳이다.

1. **도구 결과가 컨텍스트로 들어가기 전**: 도구가 긴 문서를 돌려주면 추출 모델이 에이전트가 요청한 구간만 컨텍스트로 끌어온다. 컨텍스트 창은 작게 유지되고 모든 사실에 출처 포인터가 붙는다.
2. **메모리에 쓰기 전**: 각 사실에 일치하는 원문 구간이 있어야 한다. 없는 사실은 저장되지 않는다.
3. **최종 답을 내기 전**: 검증 훅이 모든 주장에 같은 검사를 돌린다. 위치를 찾지 못한 주장은 no-match 결과와 함께 에이전트에게 돌아간다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Before the agent writes to memory, each fact needs a matching source span. Facts without one never get stored."</blockquote><blockquote>"Before the final answer, a verify hook runs the same check on every claim. Claims it cannot locate go back to the agent with a no-match result."</blockquote></details>

## 지금 호출할 수 있는 경로

여기서 원문과 제품 상태 사이에 간격이 있다. Drex Max Locate와 Drex Max Box는 [Nace Console의 Experimental 탭](https://console.nace.ai/dashboard/experimental){:target="_blank"}에 올라와 있다. 텍스트를 붙여 넣거나 페이지 이미지를 올리고 필요한 필드를 설명하면 구간이나 박스가 돌아온다. API로는 Extract가 지금 문서화된 경로라고 원문은 쓴다. 내려받은 Extract, Ground, 가격 문서에는 Locate와 Box를 직접 부르는 엔드포인트가 나오지 않는다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"For the API, Extract is the documented route today. You give it a document and a JSON schema of the fields you want. It fills each field and cites where in the document it came from."</blockquote></details>

아래 호출은 원문에 실린 예시 그대로다. API 키를 발급받아 실행해 본 것은 아니다.

```bash
curl "https://console.nace.ai/v1/documents/extract?wait_seconds=60" \
  -H "Authorization: Bearer $NACE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "source": {
      "type": "url",
      "url": "https://example.com/services-agreement.txt",
      "file_name": "services-agreement.txt"
    },
    "schema": {
      "type": "object",
      "properties": {
        "payer": { "type": "string" },
        "fee": { "type": "string" },
        "termination_notice": { "type": "string" }
      },
      "required": ["payer", "fee"]
    }
  }'
```

[Extract 문서](https://console.nace.ai/docs/perception/extract){:target="_blank"}에서 검증기로 쓸 때 필요한 부분만 추렸다.

- `url` 소스에는 `type`, `url`, `file_name`이 모두 있어야 한다. `file_name`이 빠지면 422 `invalid_request`가 난다.
- 결과의 `fields[].status`는 `found`, `not_found`, `ambiguous` 가운데 하나다. `not_found`는 문서에 대한 답이고 작업 자체는 성공으로 끝난다. 찾지 못한 값은 `data`에서 빠지지 않고 `null`로 들어간다.
- `fields[].citations`에 값이 읽힌 위치가 들어가고 `source_text`로 짧은 인용이 붙는다. `citations.enabled`와 `citations.include_source_text`의 기본값은 둘 다 `true`다.
- `fields[].confidence`는 최상위 필드와 최상위 배열의 각 항목에만 나오고 중첩 객체 안의 필드와 찾지 못한 필드에서는 `null`이다. 문서의 결과 예시에서는 `found`인 필드도 `confidence`가 `null`이다. 점수 임계값으로 분기를 짜려면 중첩 스키마를 피해야 한다.
- 스키마는 JSON Schema의 일부만 받는다. 속성은 중첩과 배열 항목을 합쳐 128개까지이고 `allOf`, 재귀·원격 참조는 422 `invalid_schema`로 거부된다. `format`, `pattern`, `minLength` 같은 제약은 검증되지 않고 필드 설명으로 옮겨질 뿐이라, 날짜 형식 같은 것은 애플리케이션에서 다시 검사해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"found, not_found or ambiguous (a value is there but couldn't be chosen uniquely). not_found is an answer about the document: the job still succeeded."</blockquote></details>

## 인용 검증에는 Ground가 더 가깝다

원문의 사용처 가운데 "인용한 문장이 그 논문에 있는가"는 Extract보다 [Ground](https://console.nace.ai/docs/perception/ground){:target="_blank"}의 일이다. 원문은 Ground를 언급하지 않지만 같은 문서 사이트에 Extract 다음 항목으로 올라와 있다. `POST /v1/documents/ground`는 대상 인용문마다 문서 안의 위치를 찾는다.

- `targets`는 1개에서 30개까지이고 각각 직접 정한 `id`와 `text`를 가진다. 같은 문구가 여러 번 나오면 `hint`나 `page_hints`로 고른다.
- 대상별 `status`는 `found`, `not_found`, `failed`다. `not_found`면 `matches`가 비고 작업은 성공으로 끝난다.
- `match_method`는 `exact`, `normalized`, `semantic` 가운데 하나다. `matched_text`는 실제로 찾은 텍스트라 정확히 일치하지 않으면 인용문과 다를 수 있다.
- `semantic: true`를 주면 `text`가 인용문이 아니라 찾을 내용의 설명이 된다.
- `options.max_matches`의 기본값은 10이고 1에서 50까지 줄 수 있다.
- 평문, HTML, 이메일에서는 위치가 `text_span`으로, PDF나 이미지에서는 `page_region`으로 나온다.

문서가 스스로 그은 선이 두 개 있고 검증 훅을 짤 때 둘 다 그대로 받아들여야 한다. 첫째, Ground는 텍스트의 위치를 찾을 뿐 주장이 참인지 확인하지 않는다. 인용문이 원문에 있다는 것과 그 인용문이 주장을 뒷받침한다는 것은 다른 문제다. 둘째, 정확 일치를 요청했다고 결과가 정확 일치인 것은 아니다. 스캔 페이지의 흐릿한 OCR 적중은 `semantic`으로 보고된다. `found`만 보고 통과시키지 말고 `match_method`와 `matched_text`를 함께 봐야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"POST /v1/documents/ground finds where each target quote appears in a document. It locates text; it doesn't check whether a claim is true."</blockquote><blockquote>"Inspect matched_text and the highlighted region; the query setting alone does not establish an exact match."</blockquote></details>

## 수치는 누가 어떻게 쟀는가

원문이 인용한 수치는 전부 Nace가 보고한 것이고 제품 페이지의 표와 일치한다.

- ExtractBench의 빈 양식 필드에서 Drex Box는 98%를 "not on page"로 답했고 VGSE-32B는 18.8%였다.
- PDF 양식 필드 세트인 ExtractBench에서 Drex Box 84.1, Liquid d1.2-ultra 65.7, VGSE-32B 64.1.
- 텍스트에서는 여섯 개 데이터셋 평균이 81.5, Perplexity Decider 1.3이 73.9. 차이가 가장 큰 곳은 답할 수 없는 질문으로 만든 SQuAD 2.0으로 84.2 대 70.0이다.

제품 페이지의 표에는 원문이 옮기지 않은 행이 있다.

- **박스 정밀도**: IoU 0.75 기준 Tight boxes에서 Drex Max Box는 54.9, VGSE-32B는 65.2다. 박스를 원본 위에 그려 사람에게 보여 주는 용도라면 이 행이 더 중요하다.
- **FUNSD**: 스캔 양식에서 Drex Max Box 77.8, VGSE-32B 80.9. VGSE public은 89.6 대 89.4로 사실상 같다.
- **"not on page"**: Liquid d1.2-ultra도 96.5다. 빈 필드를 비워 두는 동작이 Drex만의 것은 아니다.
- **텍스트 쪽 격차**: Nemotron-PII는 90.9 대 90.7이다. 평균 격차는 SQuAD 2.0과 Few-NERD(77.4 대 67.7), CUAD(85.4 대 76.7)에서 주로 생긴다.

측정 조건도 제품 페이지에 있다. 박스 쪽은 모든 시스템이 같은 고정 표본 8,477문항으로 2026년 11월 21일에서 22일 사이에 돌았다. 텍스트 쪽은 held-out 점수이고 각 모델이 자기 dev 세트에서 맞춘 컷오프를 썼다. CrossNER과 Kleister-NDA는 Drex 학습 데이터에 들어간 적이 없다고 한다. 청구 문서 행(VGSE billing)은 비공개 페이지라 Liquid AI와 TypeSafe에는 보내지 않았고 두 시스템은 그 행에서 "not run"이다. 제3자가 재현한 수치는 내려받은 자료 어디에도 없다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Every system ran on the same fixed sample of 8,477 questions, 21 to 22 November 2026."</blockquote><blockquote>"On blank form fields, Drex Box returned "not on page" 98% of the time, against 18.8% for VGSE-32B."</blockquote></details>

## 가격과 제약

[가격 문서](https://console.nace.ai/docs/perception/pricing){:target="_blank"} 기준이다. 단위는 USD이고 1,000 페이지 상당당 가격이다. Ground만 요청한 대상 1,000개당이다.

| 도구 | Low | Medium | High |
|---|---|---|---|
| parse | $3.5 | $5 | $7 |
| extract | $12 | $13.5 | $15.5 |
| ground | $12 | $12 | $12 |

- Extract는 도구 가격 $8.50에 Parse 부분이 더해진 값이다. 이미 끝난 Parse 작업을 `parse_result` 소스로 넘기면 Parse 부분은 다시 내지 않는다.
- Ground에는 Parse 부분도 effort 설정도 없다.
- 실패한 작업과 완료 전에 취소한 작업은 청구되지 않는다. 캐시 적중과 같은 요청의 정확한 반복은 $0이다.
- Drex Max Locate와 Drex Max Box 자체의 가격은 내려받은 문서에서 찾지 못했다.

확인하지 못한 것도 적어 둔다. 가중치 공개 여부와 라이선스는 원문과 제품 페이지에 언급이 없고 제공 형태는 콘솔과 API뿐이다. 온프렘 배포 가능 여부도 나오지 않는다. 한국어 문서에 대한 지원이나 평가 수치는 원문, 제품 페이지, 문서 어디에도 없다.

## 적용할 때 따져 볼 것

원문의 구조에서 제품과 무관하게 가져올 수 있는 것은 검증 지점 세 곳과 "없음"의 처리 방식이다. 아래는 원문과 문서를 읽고 정리한 판단이다.

- **부분 문자열 대조는 모델 없이 된다.** 에이전트가 인용문을 원문 그대로 내도록 강제할 수 있다면 일치 여부는 문자열 검색으로 끝난다. 이 블로그의 초안 흐름이 그렇게 돈다. Orca Bots 가 쓴 초안의 근거 인용을 스크립트가 저장해 둔 원문 추출본에서 문자열로 찾고 못 찾은 인용은 미확인으로 보고한다. 추출 모델이 필요한 구간은 주장이 원문과 다른 표현으로 쓰였을 때, 그리고 스캔이나 페이지 이미지처럼 텍스트가 없을 때다.
- **위치 확인과 참 여부 확인을 분리한다.** Ground 문서가 못박았듯 위치를 찾았다는 것은 주장이 참이라는 뜻이 아니다. 요약문처럼 원문을 바꿔 쓴 주장은 구간을 찾은 뒤에도 함의 판정이 따로 필요하다.
- **no-match를 통과·차단과 다른 상태로 둔다.** 원문은 no-match를 에이전트에게 되돌려 재검색하게 하거나 사람에게 넘긴다. `ambiguous`와 `failed`도 같은 갈래로 보내는 편이 안전하다.
- **임계값은 자기 문서로 다시 잡는다.** 신뢰도 점수는 `null`일 수 있고 벤치마크의 컷오프는 모델별 dev 세트에서 맞춘 값이다. 한국어 문서라면 처음부터 다시 재야 한다.
- **메모리 쓰기 전 검사는 비용이 싸다.** 세 지점 가운데 저장 직전 검사는 호출 빈도가 낮고 틀린 사실이 뒤의 세션으로 퍼지는 것을 막는다. 하나만 넣는다면 여기부터다.

## 참고 자료

- [Extraction Models as Verifiers for Agents (DAIR.AI, Elvis Saravia)](https://nlp.elvissaravia.com/p/extraction-models-as-verifiers-for){:target="_blank"}
- [Drex for Extraction (Nace)](https://www.nace.ai/drex/extraction){:target="_blank"}
- [Extract 엔드포인트 문서 (Nace Console)](https://console.nace.ai/docs/perception/extract){:target="_blank"}
- [Ground 엔드포인트 문서 (Nace Console)](https://console.nace.ai/docs/perception/ground){:target="_blank"}
- [Perception 가격 문서 (Nace Console)](https://console.nace.ai/docs/perception/pricing){:target="_blank"}
- [Nace Console 문서](https://console.nace.ai/docs){:target="_blank"}
