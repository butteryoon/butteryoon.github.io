---
layout: post
comments: true
title: "'Make it a LOT more fun' 한 줄이 통했다 — 바이브 코딩 게임 4종의 프롬프트 기록"
description: "Simon Willison이 코드 한 줄 안 보고 만든 게임 4종. 단일 프롬프트에서 SPEC 주도 개발까지 네 가지 작업 방식과, 실제로 쓴 프롬프트·모델 선택 기준을 원문과 대조해 정리했다."
img: simonw_vibe_game_title.webp
date: 2026-11-01 20:30:00 +0900
last_modified_at: 2026-11-01 20:30:00 +0900
tags: [simonw, vibe-coding, coding-agent, claude-opus, threejs, prompt-pattern, llm-agent]
related: llm-agent
categories: dev
source_url: https://simonwillison.net/2026/Oct/28/vibe-game-development/
source_date: 2026-10-28
---

Simon Willison이 10월 28일에 쓴 글은 게임 이야기처럼 보이지만, 실제로 읽히는 건 코딩 에이전트를 어디까지 맡길 수 있는가에 대한 기록이다. 그는 코드를 한 줄도 보지 않고 게임 네 개를 만들었고, 프로젝트마다 작업 방식을 바꿨다. 단일 프롬프트, 반복 프롬프트, 기능 확장, SPEC 주도 개발 — 지금 에이전트를 쓰는 사람이 매일 고르는 네 가지 선택지 그대로다.

(이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Willison은 2025년 11월을 코딩 에이전트의 변곡점으로, 2026년 10월을 '바이브 게임 개발'의 변곡점으로 본다. 모델이 3D 모델링·음악에 더해 **게임 디자인(재미)** 까지 해내기 시작했다는 것이다. 실무로 가져갈 건 셋이다 — ① `Build a mobile-friendly three.js game where...` 로 시작해 코딩 에이전트 없이 채팅에서 검증하고, ② 작동은 하는데 심심할 때 `Make it a LOT more fun` 을 던지고, ③ 규모가 커지면 브레인스토밍 → SPEC → 구현으로 쪼갠다. 네 프로젝트 중 그가 4시간 넘게 쓴 건 하나도 없다.

## 변곡점을 어디에 찍었나

Willison은 변곡점을 두 번 찍는다. 2025년 11월은 Opus 4.5와 GPT-5.1이 코딩 에이전트를 '매일 쓰는 도구'로 만든 시점이고, 2026년 10월은 모델이 게임을 **재미있게** 만들기 시작한 시점이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"I've written before about how November 2025 was an inflection point for coding agents—the point where Opus 4.5 and GPT-5.1 crossed a tipping point that meant their respective coding agents became reliable daily drivers for software engineers. I think we've reached a new inflection point in October 2026, and this time it's for vibe coding computer games."</blockquote></details>

구분이 중요하다. '게임처럼 보이는 것'은 모델이 꽤 오래전부터 만들었다고 그는 적는다. 넘지 못한 선은 **플레이할 만한가**였다.

그가 Opus 5.5에서 달라졌다고 꼽은 영역은 셋이다 — 3D 모델링, 음악 생성, 그리고 게임 디자인. 세 번째에만 느낌표를 붙여 놨다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Claude Opus 5.5 in particular exhibits meaningful improvements in several areas needed to build a compelling game: 3D modeling, Music generation (see Scrimshaw Jukebox), Game design (!)"</blockquote></details>

여기서 말하는 바이브 코딩은 2025년 3월에 그가 붙인 원래 정의 그대로다 — 코드를 아예 보지 않는 것(`no looking at the code at all`).

## 프롬프트 네 개

원문에서 그대로 가져올 수 있는 건 결국 프롬프트다. 네 개가 반복해서 등장한다.

**① 시작 템플릿.** 글 말미에서 그가 권하는 출발점이다.

```
Build a mobile-friendly three.js game where...
```

`chatgpt.com` 과 `claude.ai` 채팅에서 바로 돌아간다. 코딩 에이전트를 띄울 필요가 없다는 점이 핵심이고, 개념 검증이 끝난 다음에 Codex나 Claude Code로 옮기라고 명시한다.

**② 재미 주입.** Battle Drones 초기 버전은 돌아가긴 했지만 재미가 없었다. 그때 던진 한 줄이 이 글 전체의 제목값을 한다.

```
Make it a LOT more fun
```

**③ 상투성 회피.** Emberwick에서 쓴 프롬프트인데, 모델에게 '뻔하지 않게'를 명시적으로 요구한 사례다.

```
Now turn this into an actual game - quadruple the size of the playable area,
introduce fun game mechanics, add a plot that isn't too cliche, some twists,
a magic system, make it really good
```

결과에 대한 평가는 솔직하다 — 조금 뻔하긴 한데 끝까지 따라갈 만큼은 새로웠다고 적었다.

**④ 디테일 지시.** Quarter Sun Cove에 넣은 한 줄로, 이스터에그가 쏟아진 원인이다.

```
surprise and delight the user frequently
```

장난치는 너구리, 계체 대회를 운영하는 코끼리물범, 호박을 바다로 굴려 가는 항구물범, 떠돌이 염소, 가끔 들어맞는 호박 재배 미신까지 나왔다.

## 네 프로젝트가 그리는 비용 곡선

네 프로젝트는 투입량 순으로 늘어놓을 수 있고, 그 배열 자체가 작업 방식 선택표가 된다.

| 프로젝트 | 작업 방식 | 반복 | 도구 |
|---|---|---|---|
| Tree Appreciator | 단일 프롬프트 | **0회** | Claude 채팅 |
| Emberwick | 단계적 확장 + 플레이테스트 | 여러 번 | Claude 채팅 |
| Battle Drones | 포팅 + 기능 확장 | 여러 번 | Claude Code |
| Quarter Sun Cove | 브레인스토밍 → SPEC → 구현 | — | Opus 5.5 Ultracode |

**Tree Appreciator** 는 차를 타고 가다 즉흥으로 만든 것이다. 프롬프트 한 번에 모바일에서 돌아가는 틀린그림찾기 게임이 나왔고, 완성한 나무를 '허바리움'에 모으는 수집 요소까지 붙어 있었다. 후속 프롬프트가 한 번도 없었다.

**Emberwick** 은 반대다. 탑다운 RPG 엔진 프로토타입에서 출발해 지형·지하 터널·동굴을 얹고, 마지막에 ③번 프롬프트로 게임화했다. 반복이 필요했던 이유가 실무적으로 흥미롭다 — 코드 버그가 아니라 **게임플레이 버그**였다. 다른 스토리 요소 때문에 NPC가 엉뚱한 곳으로 옮겨 가 퀘스트를 끝낼 수 없게 되는 식이다. 최종본은 15분짜리 스토리에 레벨 에디터까지 들어 있다.

**Battle Drones** 는 7월에 Claude Fable 5로 만든 2D 버전이 출발점이었다. Opus 5.5에게 `Make it a LOT more fun` 과 `Port this game to three.js` 를 차례로 시키자 다른 게임이 됐다. 지금은 2~4인 멀티플레이(바이브 코딩으로 만든 WebSocket 릴레이 서버), 메달, 리플레이, 맵 7종, 랜덤 맵 생성기, 룰 변형 뮤테이터까지 붙어 있다.

**Quarter Sun Cove**(호박 농장)는 규모가 커지자 방식을 바꾼 사례다. 브레인스토밍 세션으로 SPEC 파일을 먼저 뽑고, 그다음 Opus 5.5 Ultracode가 수백만 토큰을 태우며 이틀 가까이 구현했다. 아직 Battle Drones만큼 재미있진 않다고 스스로 평가하면서도 계속 붙잡는 이유는, 예전 같으면 아예 만들어지지 않았을 종류의 게임이라서다.

전체를 관통하는 수치가 하나 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Not a single project represents more than four hours of my work (plus many more hours of Claude Code churning away by itself)."</blockquote></details>

사람이 쓴 시간은 프로젝트당 4시간 미만이고, 에이전트가 혼자 돌아간 시간은 그보다 훨씬 길다. 이 비율이 지금 코딩 에이전트의 작업 분배를 그대로 보여준다.

## 모델 선택 — Opus 5.5 vs GPT-6 Astra

같은 프롬프트를 양쪽에 돌려 본 비교가 들어 있다. 결론은 GPT-6 Astra도 게임을 만들긴 하는데 재미 감각이 아직 못 따라온다는 것이고, 연말까지는 따라잡으리라 봤다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"I haven't done an exhaustive study, but my current impression is that GPT-6 Astra isn't quite there yet. It can absolutely build a game, but its sense of fun doesn't match that of Opus 5.5. I expect OpenAI will get there by the end of the year."</blockquote></details>

Emberwick의 초기 프롬프트 묶음을 GPT-6 Pro에도 똑같이 넣고 완성본을 공개해 뒀다. 보기엔 훌륭한데 플레이 감이 떨어진다는 게 그의 평가다. 벤치마크 점수 한 줄보다 이런 동일 프롬프트 대조가 모델 고를 때 더 쓸모 있다.

## 에이전트 대전 모드는 평가 하네스로 읽힌다

Battle Drones에는 LLM끼리 붙이는 모드가 있다. 게임이 5초 진행하고 멈춰서, 각 모델이 바뀐 상태를 해석하고 다음 명령 묶음을 내놓을 시간을 준다. Claude Haiku 5.5 대 GPT-6 Luna 대전 리플레이가 공개돼 있다.

구조만 떼어 놓고 보면 익숙하다 — 상태 스냅샷을 주고, 제한 시간 안에 행동을 받고, 결과를 JSON 리플레이로 재현한다. 에이전트 평가 하네스를 짜 본 사람이면 바로 가져다 쓸 수 있는 틀이다. 자원 관리와 전술 판단이 한 지표로 묶여 나오는 환경을 통째로 얻는 셈이다.

모델이 만든 3D 모델만 따로 모아 보는 페이지도 있어서, 이미지 생성 모델 없이 어디까지 나오는지 눈으로 확인할 수 있다.

## 반발도 같이 봐야 한다

Willison은 반대편에도 한 절을 할애했다. AI 생성 아트·음악·글·코드가 각각 반발을 사 왔는데 바이브 코딩 게임은 그 넷을 한꺼번에 합친다는 것이다. 근거로는 Steam Next Fest가 AI 슬롭으로 채워진다는 불만을 다룬 IGN 기사를 든다.

본인 입장은 분명하다. 자기 게임을 전통적인 게임 잼이나 대회에 내지는 않겠다고 썼다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"I certainly won't be entering any of my vibe coded games in traditional game jams or competitions—I don't feel like I've earned that right."</blockquote></details>

Emberwick 엔딩에서 감정이 움직인 자신에 대해 조금 민망하다고 적은 대목도 같은 맥락이다. 그래픽·플롯·캐릭터·음악·대사가 전부 모델에서 나왔고, 자기 창작 기여는 '판타지 마을이고 동굴에 트롤이 자고 있다' 정도였다는 것이다.

## 오늘 해 볼 것

읽고 끝낼 글은 아니다. 당장 손댈 수 있는 게 셋 있다.

1. **채팅에서 개념 검증부터.** `Build a mobile-friendly three.js game where...` 로 시작하면 코딩 에이전트를 띄우지 않고도 돌아가는 결과물이 나온다. 에이전트 전환은 개념 검증 이후다. 이 순서는 게임이 아닌 프로토타입에도 그대로 적용된다.
2. **품질 지시를 한 줄로 분리하기.** `Make it a LOT more fun` 이 통한 이유는 무엇을 고치라고 한 게 아니라 **어느 방향으로 더 밀라고** 준 데 있다. 기능 요구와 품질 요구를 한 프롬프트에 섞지 말고 떼어 보는 편이 낫다.
3. **규모가 커지면 SPEC을 먼저.** 브레인스토밍으로 SPEC을 뽑고 구현을 별도 실행에 맡기는 패턴은, 긴 작업에서 맥락이 흐려지는 문제를 피하는 데 지금으로선 가장 검증된 방법이다.

덧붙이면, 그는 Battle Drones를 Apache 2.0으로 공개했고 **바로 다음 날** Gabriel Turpin이 Panache라는 포크를 내놨다. 스팀펑크 1900년대 프랑스로 세계관을 바꾸고 모델·음악·인트로 시퀀스를 새로 넣은 버전이다(저장소 생성일 2026-10-21로 확인). 코드·에셋·디자인이 전부 모델에서 나온 프로젝트가 공개 하루 만에 포크되는 속도는, 그 자체로 이 방식의 생산성을 보여준다.

## 참고 자료

| 구분 | 링크 |
|---|---|
| 원문 | [The October inflection point for vibe game development](https://simonwillison.net/2026/Oct/28/vibe-game-development/){:target="_blank"} |
| Tree Appreciator | [플레이](https://tools.simonwillison.net/tree-appreciator){:target="_blank"} |
| Emberwick | [최종 버전 플레이](https://simonw.github.io/emberwick/v11/emberwick.html){:target="_blank"} |
| Battle Drones | [플레이](https://simonw.github.io/battledrones/v11/){:target="_blank"} · [3D 모델 브라우저](https://simonw.github.io/battledrones/models.html){:target="_blank"} · [저장소(Apache-2.0)](https://github.com/simonw/battledrones){:target="_blank"} |
| 에이전트 대전 리플레이 | [Haiku 5.5 vs GPT-6 Luna](https://simonw.github.io/battledrones/v11/?replay=battles/showcase/2026-10-14-frostbite-42-claude-haiku-5.5-vs-gpt-6-luna.json){:target="_blank"} |
| Battle Drones 개발기 | [10월 20일 글](https://simonwillison.net/2026/Oct/20/battledrones/){:target="_blank"} |
| Panache (포크) | [저장소](https://github.com/menufactory43/panache/){:target="_blank"} |
| Quarter Sun Cove | [플레이](https://simonw.github.io/pumpkin-farmer/){:target="_blank"} |
| 바이브 코딩 원래 정의 | [2025년 3월 글](https://simonwillison.net/2025/Mar/19/vibe-coding/){:target="_blank"} |
| 이전 변곡점(2025-11) | [태그 모음](https://simonwillison.net/tags/november-2025-inflection/){:target="_blank"} |
| 음악 생성 사례 | [Scrimshaw Jukebox](https://simonwillison.net/2026/Oct/6/scrimshaw-jukebox/){:target="_blank"} |
| three.js | [공식 사이트](https://threejs.org/){:target="_blank"} |

원문이 링크한 Quarter Sun Cove의 `SPEC.md`(`github.com/simonw/pumpkin-farmer`)는 2026년 11월 1일 기준 404다. 저장소가 공개 상태가 아니어서 SPEC 원본은 확인할 수 없었고, 게임 자체는 GitHub Pages에서 정상 동작한다.
