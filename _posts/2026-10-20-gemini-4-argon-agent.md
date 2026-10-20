---
layout: post
comments: true
title: "Gemini 4 Argon — 에이전트 운영자가 볼 가격·접근 경로·실측 수치"
description: "구글 Gemini 4 Argon을 에이전트 운영 관점에서 정리했다. 도입가 $2/$10(11월 30일 이후 $4/$20), 입출력 합산 1M 토큰과 continuation protocol, ADK v2.12.0에 들어간 세션 오버헤드 개선, vLLM Gemma 4 E2B 2.85배 가속, IPI 공격 성공률 0.7%까지."
img: gemini-4-argon_title.webp
date: 2026-10-20 20:40:00 +0900
last_modified_at: 2026-10-20 20:40:00 +0900
tags: [deepmind, gemini, gemini-4-argon, antigravity, adk, vllm, prompt-injection, llm-agent, llm, agent-research]
related: agent-research
categories: dev
source_url: https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-argon/
source_date: 2026-10-13
---

구글이 2026년 10월 13일 [Gemini 4 Argon](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-argon/){:target="_blank"}을 공개했다. 발표문에는 벤치마크 순위 외에도 구글 내부 팀과 얼리 액세스 파트너가 실제로 Argon을 돌려 얻은 수치가 꽤 들어 있다. 이 글은 그중 에이전트를 직접 만들고 운영하는 사람이 당장 따져 볼 만한 것, 즉 가격과 접근 경로, 토큰 한도, 공개 저장소에 반영된 최적화, 안전 장치의 범위만 추려 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Argon은 Gemini API 유료 티어(AI Studio·Android Studio)와 Gemini 플랫폼, Antigravity(AI Ultra 구독자)에서 쓸 수 있다. 도입가는 입력 100만 토큰당 $2, 출력 $10이고 캐시 입력은 95% 할인된다. 2026년 11월 30일 이후에는 $4/$20이 된다. 한도는 입출력 합산 1M 토큰이며 긴 출력은 새 continuation protocol 문서를 따라야 한다. 구글은 Antigravity에서 에이전트 코드 제출이 35% 넘게 늘고 롤백은 3분의 1 가까이 줄었다고 밝혔다. ADK 세션 오버헤드 개선(ADK v2.12.0 포함)과 vLLM의 Gemma 4 E2B 2.85배 가속은 공개 저장소에서 직접 확인할 수 있다.

## 어디서, 얼마에 쓰나

| 대상 | 접근 경로 |
|------|-----------|
| 개발자 | Gemini API 유료 티어(AI Studio, Android Studio). Google AI Ultra 구독자는 Antigravity와 AI Studio에서도 사용 |
| 기업 | Gemini for business, Gemini API, Google Antigravity |
| 일반 사용자 | Gemini 앱, 검색 AI Mode (Google AI Ultra·Pro 구독자) |
| 연구자 | Antigravity 최고 사용량 티어 6개월 무료 — 과학자 1만 명 대상, [신청서](https://forms.gle/WvmBbPvRbpKG1BPn9){:target="_blank"} 접수 중(자격·용량·선발 기준 적용) |
| 보안 방어자 | [Fairwind Program](https://deepmind.google/fairwind-program/){:target="_blank"} 가입 시 사이버 가드레일을 뺀 Argon 제공(순차 온보딩) |

[Google Antigravity](https://antigravity.google/){:target="_blank"}는 구글 내부 도구가 아니라 공개 서비스다. 다만 Argon은 AI Ultra 구독자에게만 열려 있다.

[요금](https://aistudio.google.com/docs/pricing){:target="_blank"}은 도입가 기준 입력 100만 토큰당 $2, 출력 $10이다. 캐시된 입력 토큰은 95% 할인된다. 도입 기간은 2026년 11월 30일까지이고 그 뒤로는 $4/$20이 적용되므로 월 예산은 인상 후 단가로 잡아 두는 편이 안전하다. 긴 시스템 프롬프트나 스킬 문서를 매 턴 다시 넣는 에이전트라면 캐시 할인 폭이 체감 비용을 크게 좌우한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The model launches today at an introductory price of $2 per million input tokens and $10 per million output tokens (with cached input tokens at 95% off)." / "After the introductory period expires on November 30, 2026, the price of $4 per 1M input tokens and $20 per 1M output tokens will apply."</blockquote></details>

## 1M 토큰은 입력·출력 합산이다

Gemini API에서 Argon의 한도는 입력과 출력을 합쳐 100만 토큰이다. 구글은 이 한도 덕분에 Argon이 복잡한 과학·코딩·에이전트 문제를 "더 오래, 더 깊게" 풀 수 있다고 설명한다. 긴 출력을 받으려면 새로 나온 continuation protocol을 써야 하고 설정 방법은 개발자용 [long decode 문서](https://aistudio.google.com/docs/long-decode){:target="_blank"}와 기업용 [Argon 가이드](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/guides/gemini-4-argon#long-decodes){:target="_blank"}에 있다. 기존 클라이언트 코드를 그대로 붙이기 전에 이 문서부터 확인해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"This is enabled by an industry-leading 1M unified input-output token limit via the Gemini API. Read through our documentation for developers and enterprises to learn about the new continuation protocol and ensure the appropriate setup for your needs."</blockquote></details>

## 에이전트 작업에서 나온 수치

구글 전체 엔지니어링 수치부터 보면, 최근 몇 주 동안 Antigravity에서 Argon을 쓴 팀들의 에이전트 코드 제출은 35% 넘게 늘었고 코드 롤백은 3분의 1 가까이 줄었다. Argon을 쓴 팀 전반의 수치이고 특정 프로젝트 하나에 한정된 결과가 아니다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Argon in Antigravity has helped teams increase agentic code submissions by more than 35% while reducing code rollbacks by nearly a third."</blockquote></details>

개별 사례는 네 가지다.

- **Chrome 리팩터링**: Chromium 코드베이스는 4,000만 줄이다. 그 안에 있는 약 600만 줄짜리 레거시 브라우저 기능 폴더에서 핵심 웹페이지 처리 코드를 분리하는 작업에 Argon이 투입됐다. 깨끗한 코드 변경 여러 개를 한 번에 흘려보내는 방식으로 수백 개 파일에 걸친 아키텍처 리팩터링을 끝냈고 몇 달 걸릴 일이 몇 주로 줄었다고 한다([Chromium 이슈](https://issues.chromium.org/issues/540709814){:target="_blank"}).
- **ADK 성능 개선**: [Agent Development Kit](https://github.com/google/adk-python){:target="_blank"} 팀은 Argon에게 `.agents/skills` 룰북과 "기존 앱을 깨뜨리지 않고 프레임워크 오버헤드를 없애라"는 한 문단짜리 목표만 주고 밤새 자율로 돌렸다. 결과는 [커밋](https://github.com/google/adk-python/commit/1754b3eeb6cdb2c16e3b30619a12066623df132e){:target="_blank"}으로 반영됐다. 커밋 메시지를 보면 `InMemorySessionService`가 읽을 때마다 세션 전체를 깊은 복사하던 부분을 줄였고 OpenTelemetry·인증·LLM flow 모듈은 처음 쓰일 때 import하도록 미뤘다. 50턴 뒤 첫 모델 요청까지 걸리는 시간이 64ms에서 36ms로, 새 프로세스의 ADK import가 1.22초에서 0.97초로 줄었다. 이 커밋은 ADK v2.12.0(2026-10-13 릴리스)에 들어 있으니 `InMemorySessionService`로 긴 세션을 돌리는 사람은 버전만 올려도 효과를 본다. 과정은 구글 개발자 블로그의 [엔지니어링 글](https://developers.googleblog.com/optimizing-a-200000-line-python-codebase-overnight-with-gemini-4-argon){:target="_blank"}에 따로 정리돼 있다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"Working autonomously overnight, Argon surfaced optimizations, now live in google/adk-python, that cut in-memory session prep time by 40% in long sessions and sped up cold imports by 21%."</blockquote></details>
- **vLLM 서빙 가속**: A100 한 장에서 vLLM으로 Gemma 4 E2B를 서빙할 때 엔드투엔드 처리량이 초당 약 37,900토큰에서 108,000토큰 이상으로 2.85배 올랐다. 토큰 일치율은 100%였고 품질 저하는 없었다. 몇 달 걸리던 작업을 며칠 만에 끝냈다고 한다. 어텐션 튜닝부터 speculative decoding까지 손댄 내용은 vLLM [업스트림 PR 목록](https://github.com/vllm-project/vllm/pulls?q=is%3Apr+author%3Alucianommartins+%28created%3A%3E2026-10-05+created%3A%3C2026-10-10%29){:target="_blank"}에서 볼 수 있다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"Using Argon, we achieved a 2.85x end-to-end inference speedup on an A100 via vLLM (accelerating throughput from ~37,900 to over 108,000 tokens per second on Gemma 4 E2B) with 100% token parity and zero quality drop in just a few days, a process that would usually take months."</blockquote></details>
- **xTB 최적화**: 오픈소스 양자화학 패키지 xTB의 분자 구조 예측이 최대 2.3배, 컨포머 탐색이 1.8배 빨라졌고 변경 사항은 공식 저장소에 기여됐다.

## Antigravity의 멀티 에이전트 루프

구글 내부에서는 Antigravity의 `/goal` 같은 슬래시 커맨드와 커스텀 스킬을 묶어 멀티 에이전트 루프를 돌린다. 이 방식으로 3D 시각화와 게임, 인터랙티브 애니메이션을 브라우저에서 바로 렌더링되는 바닐라 코드로 프로토타이핑하고 있다. 공개된 데모 몇 가지를 보면 이렇다.

- 책상 사진 몇 장으로 미니 레이싱 게임 만들기: 물체 모델링과 충돌 검사는 Blender MCP, 텍스처는 Nano Banana, 엔진 소리는 Gemini Omni Flash가 맡았고 Three.js로 묶었다.
- Nano Banana 이미지 한 장에서 출발해 Argon이 Blender MCP를 자율 루프로 조작하며 고딕 건축 3D 장면을 완성했다.
- 외부 라이브러리 없이 순수 Python으로 레이 트레이싱 엔진과 비디오 도구, 오디오 믹서를 직접 만들어 영상을 제작했다([소스 코드](https://github.com/google-gemini/gemini-4-demo-vanilla-3D){:target="_blank"}).

로보틱스 쪽에서는 ERQA(Embodied Reasoning) 벤치마크에서 87%를 기록했다. 사람이 콘홀(cornhole)을 하는 영상을 보고 공간 좌표와 전신 관절 움직임을 계산해 시뮬레이션 코드를 스스로 짰고 이 결과를 Apptronik Apollo 휴머노이드에 옮기자 실제 환경에서 연속 7번 성공했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Using slash commands like /goal and prompting for multi-agent loops with custom skills in Antigravity, Googlers are using Argon to prototype complex 3D visualizations, high-fidelity games, and interactive animations in vanilla code that can be rendered directly in the browser."</blockquote></details>

과학 분야 사례도 여럿 실렸다. AlphaQubit 디코더 추론 처리량을 두 배 가까이 끌어올렸고 Grad 추측의 반례를 더 넓은 부류로 찾아냈다([arXiv:2610.07304](https://arxiv.org/abs/2610.07304){:target="_blank"}). Gamow Labs 비공개 평가에서는 RareBench 57%, LabBench 52.2%를 기록했다. 다만 에이전트 운영자가 자기 환경에 바로 가져올 내용은 아니어서 여기서는 소개만 한다.

## 안전 장치: 모델에 들어 있는 것과 구글 내부에만 있는 것

Argon을 프로덕션 에이전트에 붙이려는 사람이라면 아래 세 가지를 구분해서 봐야 한다.

- **모델 자체의 학습**: Argon은 시스템 경계를 지키고 부정적인 결과도 솔직하게 보고하며 되돌릴 수 없는 단계 앞에서는 사람 확인을 기다리도록 학습됐다. 권한 오류나 실패하는 테스트를 만났을 때 허가 없는 우회로를 택하는 비율, 즉 쇼트컷 시도율은 코딩 벤치마크에서 2%로 역대 Gemini 모델 중 가장 낮았다.
- **오용 거절**: 공격용 익스플로잇 코드 작성이나 위험한 생물 물질 지원 같은 요청을 들어주지 않는 능력도 좋아졌다. 스트레스 테스트용 적대적 데이터셋에서 위반 건수가 이전 모델 대비 2배 가까이 줄었고 외부 바이오 레드팀이 뚫은 바이오 하위 과제는 1%가 안 됐다.
- **misalignment 모니터**: 에이전트의 단계별 추론과 행동을 읽고 의도하지 않은 우회를 실시간으로 막는 자동 모니터다. 구글은 이것을 내부 배포·역량 평가·강화학습 훈련에 썼다고 밝혔다. API 사용자에게 함께 제공된다는 언급은 없으므로 자기 에이전트에서 같은 역할을 원한다면 직접 구축해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"On coding benchmarks where models have no incentive to hold back, shortcut attempts dropped to just 2% on Gemini 4 Argon, our best performance compared to previous Gemini models." / "We used this monitor across internal deployments, capability evaluations, and reinforcement learning training runs"</blockquote></details>

프롬프트 인젝션 방어도 수치로 공개됐다. Argon은 Gray Swan 간접 프롬프트 인젝션(IPI) 벤치마크에서 1위를 했고 적응형 공격 15회 기준 공격 성공률이 0.7%였다. 웹페이지나 메일, 코드를 읽는 에이전트에는 의미 있는 숫자다. 다만 벤치마크상 0.7%이지 0%는 아니다. 도구 권한 제한이나 외부 입력 격리 같은 자체 가드레일은 계속 유지해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Through adversarial training and automated red teaming, Argon leads the Gray Swan Indirect Prompt Injection (IPI) benchmark, cutting attack success across 15 adaptive attempts to 0.7%."</blockquote></details>

자세한 평가 내용은 [Gemini 4 Argon 모델 카드](https://storage.googleapis.com/deepmind-media/Model-Cards/Gemini-4-Argon-Model-Card.pdf){:target="_blank"}에 있다.

## 정리: 지금 확인할 것

1. **비용**: 도입가 $2/$10은 11월 30일까지만 적용된다. 장기 운영 계획은 $4/$20 기준으로 세우고 반복해서 들어가는 프롬프트는 캐시를 적용한다.
2. **긴 출력**: 1M은 입출력 합산 한도다. 긴 생성이 필요하면 continuation protocol 문서를 보고 클라이언트를 맞춘다.
3. **ADK 사용자**: v2.12.0 이상으로 올리면 긴 세션의 `InMemorySessionService` 오버헤드와 import 시간이 줄어든다.
4. **vLLM으로 Gemma 4를 서빙 중이라면**: 위 PR 목록에서 어떤 최적화가 언제 업스트림에 들어갔는지 확인한다.
5. **가드레일**: 쇼트컷 2%, IPI 0.7%는 모델 수준의 개선이다. 실시간 misalignment 모니터는 구글 내부 장치이므로 자체 모니터링과 권한 통제는 그대로 둔다.

## 참고

- [Gemini 4 Argon: Built to solve your hardest problems](https://blog.google/innovation-and-ai/models-and-research/gemini-models/introducing-gemini-4-argon/){:target="_blank"} — Google 블로그, 2026-10-13
- [Gemini 4 Argon 모델 카드 (PDF)](https://storage.googleapis.com/deepmind-media/Model-Cards/Gemini-4-Argon-Model-Card.pdf){:target="_blank"}
- [google/adk-python 커밋 1754b3e](https://github.com/google/adk-python/commit/1754b3eeb6cdb2c16e3b30619a12066623df132e){:target="_blank"}
- [Optimizing a 200,000-line Python codebase overnight with Gemini 4 Argon](https://developers.googleblog.com/optimizing-a-200000-line-python-codebase-overnight-with-gemini-4-argon){:target="_blank"} — Google Developers Blog
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/58HvxSq7rrU){:target="_blank"}
