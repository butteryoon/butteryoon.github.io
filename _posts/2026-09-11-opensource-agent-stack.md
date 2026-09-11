---
layout: post
comments: true
title: "오픈소스로 AI 에이전트 구축하기 — 설치부터 게이트웨이, 모델 라우팅까지의 전체 지도"
description: "Hermes Agent 설치 → 소스 구조 → Oracle Cloud 무료 인스턴스 → OmniRoute 게이트웨이 → best-reasoning 모델 설정까지, 지금까지 쓴 오픈소스 에이전트 구축 포스팅들을 한 흐름으로 엮은 종합 지도."
img: command-title.webp
date: 2026-09-11 20:00:00 +0900
last_modified_at: 2026-09-11 20:00:00 +0900
tags: [hermes, ai-agent, opensource, omniroute, oracle-cloud, llm-routing, llm] # add tag
related: llm
categories: tools
---

지금까지 오픈소스 AI 에이전트를 실제로 구축하며 쓴 글들이 흩어져 있다. [Hermes 설치]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html), [소스 구조]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html), [OCI 무료 인스턴스]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html), [OmniRoute 게이트웨이]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html). 이 글에서는 그 글들을 **설치 → 구조 → 인프라 → 게이트웨이 → 모델 설정** 순서로 엮어, "오픈소스로 에이전트 하나를 끝까지 굴리는 법"의 전체 지도를 그려본다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행한다.)

<!--more-->

> **TL;DR:** 오픈소스 AI 에이전트 구축은 (1) Hermes Agent 설치 → (2) 소스 구조 파악 → (3) Oracle Cloud 무료 인스턴스 확보 → (4) OmniRoute 게이트웨이로 모델 통합 → (5) best-reasoning 라우팅 설정, 이 다섯 단계면 끝난다. 전 과정을 오픈소스 + 무료 티어 + 자체 호스팅으로 돌릴 수 있어, 벤더에 묶이지 않는 개인용 에이전트 스택이 나온다.

## 1. 시작: Hermes Agent 설치

오픈소스 에이전트의 첫 단계는 [Hermes Agent](https://github.com/NousResearch/hermes-agent){:target="_blank"} 설치다. Nous Research가 만든 이 에이전트는 **"어떤 LLM이든 붙이고, 어디서든 돌아가는"** 설계를 노린다. Windows 11에 설치할 때 챙길 건 이 정도다.

- 셸 설치 스크립트로 `uv`·Python·venv·launcher를 한 번에 구성
- `hermes setup`으로 프로바이더·모델 연결
- CLI(`hermes`), TUI, 데스크톱 앱, 텔레그램 등 어느 표면에서 불러도 같은 코어가 돈다

자세한 설치는 [Hermes Agent 설치부터 설정까지 — Windows에서 시작하기]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html)에 있다. 처음엔 OpenRouter 모델을 붙여 쓰다가 나중에 OmniRoute 라우팅으로 갈아탔는데, 그 얘기는 4절에서 한다.

## 2. 구조를 알면 커스터마이징이 쉬워진다

설치만으론 부족하다. "이 명령어가 실제로 어떤 코드를 타는가"를 알아야 플러그인·크론·게이트웨이를 제대로 확장할 수 있다. [소스 트리 투어]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html)에서 정리한 구조는 이렇다.

- **cli.py**: 진입점. 명령어가 실제 타는 모듈의 출발
- **conversation_loop**: 에이전트 코어 런타임. 한 턴이 끝나는 주기
- **gateway**: 텔레그램·디스코드 등 멀티 플랫폼 표면의 루트
- **cron**: 예약 작업 스케줄러

한 줄로 줄이면 "Hermes = 모델 무관 코어 + 갈아끼우는 플랫폼/모델"이다. 이 구조가 눈에 들어오면 크론에서 특정 모델을 고정하거나 게이트웨이에서 원하는 채널로 결과를 보내는 일이 명령어 하나로 끝난다.

## 3. 인프라: Oracle Cloud 무료 티어

에이전트를 로컬에만 두면 금방 한계에 부딪힌다. 예약 작업, 게이트웨이, 웹훅을 항상 켜두려면 **24시간 돌아가는 서버**가 있어야 한다. 그 자리를 Oracle Cloud 무료 인스턴스로 메웠다.

- **Arm A1 + AMD Micro** (CPU/메모리 계열) — 프리티어인데 GPU는 없다
- RAM 1GB 이하에서도 에이전트 게이트웨이와 라우터는 충분히 돈다
- 시작은 [OCI 무료 인스턴스 시작하기]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html), 스토리지는 [블록볼륨 추가]({{site.baseurl}}/tools/2026/09/09/oci-block-volume-attach.html)로 확장

무료 인스턴스 두 대(oc01·oc02)에 데이터 마운트까지 붙여두니, 내 서버 한 대 없이도 오픈소스 에이전트 스택을 상시 가동할 바닥이 깔렸다.

## 4. 게이트웨이: OmniRoute로 모델 하나로 통합

LLM 프로바이더(OpenRouter, NVIDIA, 커스텀)를 여럿 쓰다 보면 **엔드포인트가 흩어진다**. 이 문제를 푸는 게 오픈소스 AI 게이트웨이 [OmniRoute]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)다.

- OCI 무료 인스턴스(RAM 951MB)에 OmniRoute를 올리고 **Caddy로 HTTPS**를 붙였다
- 스왑으로 OOM을 잡고, OCI iptables 함정은 우회했다
- 500개 가까운 모델을 **하나의 OpenAI 호환 엔드포인트**로 내보낸다

덕분에 Hermes는 프로바이더별 API 키를 일일이 챙길 것 없이 OmniRoute 엔드포인트 하나만 보면 된다.

## 5. 모델 설정: best-reasoning 라우팅

마지막 단계는 어떤 모델을 쓸지 정하고 OmniRoute의 라우팅에 맡기는 일이다. 실제로는 **여기저기 흩어진 모델 변경 작업을 정리하고 OmniRoute의 `auto/best-reasoning` 모델 하나로 몰아주는** 쪽으로 정리됐다.

이 글에서 쓰는 **gemma-4-31b-it**와 **nemotron-3-ultra-550b-a55b**는 NVIDIA [Free Endpoint](https://build.nvidia.com/models){:target="_blank"}로 돈 들이지 않고 쓸 수 있다. NVIDIA API 키만 발급받아 `Authorization: Bearer <API_KEY>` 헤더로 호출하면 된다.

```
config.yaml (Hermes):
  fallback chain:
    1차: custom/auto/best-reasoning   # OmniRoute가 질문에 따라 최적 추론 모델로 자동 분배
    2차: nvidia/google/gemma-4-31b-it # 폴백
```

NVIDIA Free Endpoint를 쓰는 순서는 이렇다.

1. [build.nvidia.com](https://build.nvidia.com/models){:target="_blank"}에서 회원가입/로그인
2. **API Keys** 메뉴에서 API 키 발급 (무료, 일일 제한 있음)
3. 모델 페이지(gemma-4-31b-it, nemotron-3-ultra)에서 **API 엔드포인트 확인** — `https://integrate.api.nvidia.com/v1/chat/completions`
4. 요청 헤더에 `Authorization: Bearer <API_KEY>` 포함
5. OpenAI 호환 포맷으로 요청 가능

Hermes 설정 예시:
```yaml
providers:
  nvidia:
    base_url: "https://integrate.api.nvidia.com/v1"
    api_key: "${NVIDIA_API_KEY}"   # .env에 저장
```

여기서 짚어둘 건 두 가지다.

**하나, 엔드포인트 하나에서 모델이 알아서 골라진다.** `best-reasoning` 태그로 들어온 요청은 질문 유형(추론·코딩·일반)에 맞춰 게이트웨이가 모델을 나눠준다. 쓰는 쪽에선 모델명 하나만 적으면 되고, 성능을 손보고 싶으면 게이트웨이만 건드리면 된다.

**둘, 크론 잡은 모델을 고정해 둔다.** Hermes는 전역 설정을 바꾸면 핀이 안 걸린 크론 잡을 `[drift_skip:silent]`로 조용히 건너뛴다. 그래서 크론 잡마다 실행할 모델을 **명시적으로 핀**해 뒀다 — 수집은 가벼운 gemma-4, 초안 생성이나 심층 분석은 고성능 모델로. 이렇게 해둬야 전역 설정이 바뀌어도 예약 작업이 제 모델로 계속 돈다.

## 마무리

이 다섯 단계를 다 붙이면 **오픈소스만으로 굴러가는 에이전트 스택**이 완성된다.

| 단계 | 구성 요소 | 담당 글 |
|------|----------|---------|
| 1. 설치 | Hermes Agent | [Hermes 설치]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html) |
| 2. 구조 | 소스 트리 파악 | [소스 구조]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html) |
| 3. 인프라 | OCI 무료 인스턴스 | [OCI 시작]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html), [블록볼륨]({{site.baseurl}}/tools/2026/09/09/oci-block-volume-attach.html) |
| 4. 게이트웨이 | OmniRoute + Caddy | [OmniRoute 셀프호스팅]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html) |
| 5. 모델 | best-reasoning 라우팅 | 이 글 |

벤더에 묶이지 않고, 무료 티어로 항상 켜져 있고, 엔드포인트 하나로 모든 모델을 쓴다. 이 조합이면 개인이든 사내든 오픈소스 에이전트를 본격적으로 굴려볼 준비는 끝난 셈이다. 다음 편에서는 크론 기반 자동 블로그 분석 파이프라인(수집 → 초안 생성)을 실제로 굴리며 부딪힌 문제와 고친 과정을 풀어보겠다.

## 참고
- [Hermes Agent 설치]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html)
- [Hermes 소스 트리]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html)
- [OCI 무료 인스턴스 시작]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)
- [OCI 블록볼륨 추가]({{site.baseurl}}/tools/2026/09/09/oci-block-volume-attach.html)
- [OmniRoute 셀프호스팅]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)