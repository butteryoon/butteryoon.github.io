---
layout: post
comments: true
title: "Prompt Caching 실전 가이드 — 원리·프롬프트 설계·프로바이더별 차이·운영 팁"
description: "LLM 프로덕션에서 비용과 레이턴시를 줄이는 Prompt Caching의 prefix matching 원리, 고정/변동 프롬프트 분리 설계, Anthropic·OpenAI·Bedrock·Gemini별 캐싱 방식 비교, 히트율 메트릭 운영까지 정리"
img: llm_api_title.jpg
date: 2026-08-05 00:12:00 +0900
last_modified_at: 2026-08-05 22:50:00 +0900
tags: [llm, prompt-caching, cost-optimization, anthropic, openai, bedrock, observability]
related: llm
categories: dev
---

LLM을 프로덕션에서 쓸 때 가장 먼저 부딪히는 비용 문제는 **같은 시스템 프롬프트를 매 요청마다 다시 보내고, 매번 전액을 낸다**는 점이다. Prompt Caching을 제대로 적용하면 무신사(29CM)의 실측 사례처럼 **전체 비용 64% 절감, 히트율 98%**도 가능하다 — 다만 "얼마나 절감되나"는 프롬프트 구조와 프로바이더에 따라 크게 달라진다. 이 글은 코드 구현보다 **원리·프롬프트 설계·프로바이더별 차이·운영 체크리스트** 위주로 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 사실 검증·사례 보완 후 발행했다.)

<!--more-->

> **TL;DR:** Prompt Caching = **요청 앞부분(prefix)이 바이트 단위로 일치할 때 KV 캐시를 재사용**하는 것. ① 고정 영역(시스템 프롬프트·규칙·예시·스키마)은 앞에, 변동 영역(사용자 입력·데이터)은 뒤에 두는 구조 분리가 전부의 시작이다. ② 캐시 마커·TTL·과금 방식은 프로바이더마다 다르다 — Anthropic은 명시적 breakpoint(기본 5분, 1h 옵션), OpenAI는 자동(1,024토큰 이상), Bedrock은 `cachePoint` 블록, Gemini는 암시적+명시적 병행. ③ "캐시 90% 할인"이 청구서 90% 절감이 아니다 — 절감 상한은 **캐시 가능한 prefix가 전체 비용에서 차지하는 비중**이 정한다.

## 왜 Prompt Caching인가

| 문제 | 기존 방식 | 캐싱 적용 후 |
|------|-----------|--------------|
| 동일 시스템 프롬프트(수천~수만 토큰) 반복 전송 | 매 호출 전액 과금 | 최초 쓰기(cache write) 후 읽기(cache read) 요금으로 처리 |
| 레이턴시 | 매번 전체 프리필(prefill) 연산 | 캐시 히트 시 프리필 생략 → TTFT 단축 |
| 비용 가시성 | 입력/출력 합계만 보임 | 읽기/쓰기 토큰 분리 집계 → 히트율 정량 확인 |

## 동작 원리 — Prefix Matching

1. **직렬화**: 요청이 `tools → system → messages` 순으로 토큰화된다
2. **prefix 비교**: 요청 **앞부분부터 연속된 토큰**이 기존 캐시와 일치하는지 확인
3. **히트**: 일치하면 해당 구간의 KV(키-밸류) 캐시를 재사용 (`cache_read`)
4. **미스**: 없으면 전체 프리필 후 캐시에 저장 (`cache_write`)

> **핵심 불변식**: prefix 어딘가의 **한 바이트만 달라져도 그 뒤 전체가 무효**가 된다. 시스템 프롬프트에 현재 시각을 찍거나, 툴 목록 순서가 요청마다 바뀌거나, JSON 직렬화가 비결정적이면 마커를 아무리 달아도 캐시는 죽는다.

## 프롬프트 설계 — 고정/변동 분리

| 영역 | 포함 내용 | 위치 | 캐시 |
|------|-----------|------|------|
| **고정 (stable)** | 도메인 규칙·정책, 출력 스키마, few-shot 예시, 안전 가이드 | 시스템 프롬프트 (앞) | 대상 |
| **변동 (volatile)** | 사용자 질문, 상품 데이터, RAG 검색 결과, 세션 컨텍스트 | 유저 메시지 (뒤) | 비대상 |

```text
[System]  규칙·스키마·예시 … (예: 15K 토큰, 요청 간 동일)
   ← 캐시 경계 (breakpoint) →
[User]    이번 요청의 데이터 (예: 2K 토큰, 매번 다름)
```

멀티턴 대화는 마지막 턴 끝에 breakpoint를 옮겨가며 찍으면 이전 대화 전체가 누적 캐시된다. 반대로 **매 요청 첫 부분부터 달라지는 프롬프트는 캐시하지 마라** — 읽기는 0회인데 쓰기 프리미엄만 낸다.

## 프로바이더별 차이 — 여기가 함정이다

"캐시 포인트 + TTL 1시간" 같은 설정이 범용인 것처럼 보이지만, 실제로는 프로바이더마다 방식·TTL·과금이 다르다.

| | Anthropic (Claude) | OpenAI | AWS Bedrock | Google Gemini |
|---|---|---|---|---|
| 방식 | 명시적 `cache_control` breakpoint (최대 4개) | **자동** (마커 없음) | 명시적 `cachePoint` 블록 (Converse API, 최대 4개) | 암시적(자동) + 명시적 캐시 병행 |
| TTL | 기본 **5분** (읽을 때마다 갱신), `ttl: "1h"` 옵션 | 유휴 5~10분 후 제거 (조정 불가) | 기본 5분, 모델에 따라 `CachePointBlock`에 1h TTL 지정 가능 | 명시적 캐시 기본 60분 (TTL 지정 가능) |
| 읽기 요금 | 기본 입력의 **0.1배** | 세대에 따라 50~90% 할인 | ~90% 할인 | 75~90% 할인 |
| 쓰기 요금 | 5분 TTL **1.25배**, 1h TTL **2배** | 없음 (자동) | 모델별 프리미엄 | 명시적 캐시는 **시간당 스토리지 요금** 별도 |
| 최소 크기 | 모델별 **512~4,096토큰** | 1,024토큰 | 모델별 (Claude 계열 ~1,024) | 모델별 2,048~4,096 |

운영 관점의 시사점:

- **Anthropic**: 최소 크기가 모델별로 다르고(최신 모델 512, 일부 구형 4,096) 그보다 짧은 prefix는 **에러 없이 조용히 캐시가 안 된다**. TTL은 읽을 때마다 갱신되므로, 트래픽이 5분 간격보다 촘촘하면 기본 5분 캐시가 계속 살아 있다 — 1h TTL(쓰기 2배)은 트래픽이 띄엄띄엄한 배치성 워크로드에서만 이득이다. 손익분기: 5분 TTL은 2번째 요청부터, 1h TTL은 3번째 요청부터.
- **OpenAI**: 마커가 없고 자동이라 코드 수정 없이 적용되지만, 반대로 **TTL을 제어할 수 없어** 배치 간격이 벌어지면 조용히 만료된다.
- **Bedrock**: `cachePoint`는 Bedrock Converse API 고유 용어다. Anthropic 직접 호출과 파라미터 체계가 다르니 문서를 섞어 읽지 말 것.
- **Gemini**: 명시적 캐시는 히트 할인 외에 **보관 시간에 대한 스토리지 요금**이 따로 붙는 독특한 구조다.

## 현실 계산 — "90% 할인 ≠ 청구서 90% 절감"

가장 흔한 기대 오류다. 캐시 읽기가 90% 할인이어도, **절감액의 상한은 캐시 가능한 prefix가 전체 비용에서 차지하는 비중**이다. [Anthropic 공식 과금 산식을 분석한 글](https://www.youngju.dev/blog/2026-07-17-llm-api-cost-arithmetic)의 계산이 좋은 예다 — Anthropic 문서의 예제 시나리오에서는 캐시가 완벽히 걸린 상태에서도 **총액 절감이 25.5%**에 그친다. 출력 토큰(입력보다 5배 비쌈)과 변동 입력은 캐시와 무관하게 전액이기 때문이다.

절감이 극대화되는 조건은 명확하다:

- 시스템 프롬프트(고정 prefix)가 **크고** (수천~수만 토큰)
- 변동 입력과 출력이 상대적으로 **작고**
- 같은 prefix로 요청이 **TTL 안에 반복**될 때

국내 프로덕션 사례로는 [무신사(29CM) 테크블로그의 "LLM 비용 64% 절감, 캐시 히트율 98% 달성기"](https://techblog.musinsa.com/llm-%EB%B9%84%EC%9A%A9-64-%EC%A0%88%EA%B0%90-%EC%BA%90%EC%8B%9C-%ED%9E%88%ED%8A%B8%EC%9C%A8-98-%EB%8B%AC%EC%84%B1%EA%B8%B0-d568135bd40e){:target="_blank"}가 정확히 이 조건에 해당한다. AWS Bedrock 기반 LLM(상품 속성 추출·고객 응대 등)에서 비용 증가를 분석한 뒤, 시스템 프롬프트 블록 뒤에 `CachePointBlock`(TTL 1시간)을 붙이고 LangChain4j의 `ChatModelListener`로 토큰 type(input/output/cache_read/cache_write)별 Prometheus 메트릭을 수집해 — **히트율 98%, 전체 비용 64% 절감**을 실측으로 검증했다. 고정 프롬프트가 크고 호출이 잦은 워크로드의 모범 사례다.

계산 예시로는 [멀티 프로바이더 캐싱 아키텍처 글](https://wikidocs.net/blog/@jaehong/12145/){:target="_blank"}의 시나리오(시스템 프롬프트 10K + 동적 100토큰, 월 100만 요청)에서 99% 히트 시 **88% 절감**이 나온다. 반대로 짧은 시스템 프롬프트 + 긴 출력 구조라면 캐싱 효과는 미미하다 — 적용 전에 자기 워크로드로 계산부터 해야 한다.

**셀프호스팅**도 같은 원리가 적용된다. vLLM의 [Automatic Prefix Caching](https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html)은 과금이 아니라 **GPU 프리필 연산 자체를 절약**한다 — 온프레미스에서 같은 시스템 프롬프트로 대량 요청을 처리한다면 기본으로 켜야 하는 기능이다.

## 메트릭 — 히트율과 비용 검증

| 메트릭 | 의미 | 확인 포인트 |
|--------|------|-------------|
| `input` | 캐시 밖 일반 입력 토큰 | 기준선 |
| `cache_write` (Anthropic: `cache_creation_input_tokens`) | 캐시 적재 토큰 | 안정 상태에서는 드물게만 발생해야 함 |
| `cache_read` (`cache_read_input_tokens`) | 히트로 읽은 토큰 | **히트율 = cache_read / (cache_read + input + cache_write)** |
| 레이턴시 (TTFT) | 첫 토큰까지 시간 | 히트 시 감소 확인 |

**반복 요청에서 `cache_read`가 계속 0이면 조용한 무효화 요인이 있다** — 시스템 프롬프트 안의 `datetime.now()`, 정렬 안 된 JSON 직렬화, 요청마다 달라지는 툴 목록이 3대 범인이다. 두 요청의 렌더링된 프롬프트를 바이트 비교하면 바로 찾는다.

## 운영 체크리스트

| 단계 | 확인 항목 |
|------|-----------|
| **적용 전** | 토큰 메트릭 수집 파이프라인 구축 → 비용 상위 API 1~2개 식별 → 해당 프롬프트의 고정/변동 분리 가능 여부와 **예상 절감률 계산** |
| **적용 중** | 고정 영역을 시스템 블록으로 이동 → breakpoint를 고정 영역 끝에 배치 → 배포 후 첫 요청에 `cache_write`, 이후 `cache_read`만 찍히는지 확인 |
| **적용 후** | 히트율 모니터링 (목표는 워크로드에 따라 다름) → 실측 절감 vs 사전 계산 대조 → 프롬프트·툴 변경 배포 시 캐시 전량 무효화(첫 요청 재적재) 인지 |

## 자주 하는 실수

| 실수 | 증상 | 해결 |
|------|------|------|
| 고정 규칙이 유저 메시지 중간에 위치 | 히트율 0~10% | 모든 고정 내용을 시스템 블록 앞쪽으로 |
| 시스템 프롬프트에 동적 값 삽입 (시각, 세션 ID) | 히트율 0% | 동적 값은 마지막 breakpoint 뒤로, 또는 제거 |
| prefix가 최소 크기 미만 | 마커 있어도 `cache_write` 0 (조용한 실패) | 프로바이더·모델별 최소 토큰 확인 |
| TTL보다 긴 호출 간격 | 매 호출 `cache_write` 재발생 | 호출 주기 확인 후 긴 TTL 옵션 또는 워밍 요청 검토 |
| 메트릭 미분리 | 히트율 계산 불가 | 토큰 유형별 태그로 분리 집계 |
| 모델/툴 변경을 가볍게 배포 | 배포마다 캐시 전량 무효화 | 캐시는 모델 단위로 분리됨을 릴리스 절차에 반영 |

## 마무리

Prompt Caching은 "옵션 하나 켜기"가 아니라 **프롬프트를 고정 prefix / 변동 suffix로 정리하는 설계 작업**이다. 그리고 도입 전에 반드시 자기 워크로드로 계산하라 — 프로바이더의 "90% 할인"은 캐시된 토큰에 대한 것이지 청구서에 대한 것이 아니다. 구조 분리가 잘 되어 있고 트래픽이 TTL 안에 반복되는 워크로드라면, 인프라 투자 없이 얻을 수 있는 가장 저렴한 최적화임은 분명하다.

## 참고

- [Anthropic Prompt Caching 문서](https://platform.claude.com/docs/en/build-with-claude/prompt-caching){:target="_blank"}
- [OpenAI Prompt Caching 문서](https://platform.openai.com/docs/guides/prompt-caching){:target="_blank"}
- [AWS Bedrock Prompt Caching 문서](https://docs.aws.amazon.com/bedrock/latest/userguide/prompt-caching.html){:target="_blank"}
- [Gemini Context Caching 문서](https://ai.google.dev/gemini-api/docs/caching){:target="_blank"}
- [LLM API 비용 산수 — 캐싱 90% 할인이 청구서에서 25%인 이유](https://www.youngju.dev/blog/2026-07-17-llm-api-cost-arithmetic){:target="_blank"}
- [무신사(29CM): LLM 비용 64% 절감, 캐시 히트율 98% 달성기](https://techblog.musinsa.com/llm-%EB%B9%84%EC%9A%A9-64-%EC%A0%88%EA%B0%90-%EC%BA%90%EC%8B%9C-%ED%9E%88%ED%8A%B8%EC%9C%A8-98-%EB%8B%AC%EC%84%B1%EA%B8%B0-d568135bd40e){:target="_blank"}
- [vLLM Automatic Prefix Caching](https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html){:target="_blank"}
- [OpenRouter 무료 모델 (관련글)]({{site.baseurl}}/tools/2026/07/15/openrouter_free.html)
- [RAG & AI 에이전트 주간 동향 (관련글)]({{site.baseurl}}/dev/2026/08/04/rag_agent_weekly.html)
