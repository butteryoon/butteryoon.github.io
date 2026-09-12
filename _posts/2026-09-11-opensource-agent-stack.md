---
layout: post
comments: true
title: "오픈소스로 AI 에이전트 구축하기 — 설치부터 게이트웨이, 모델 라우팅까지의 전체 지도"
description: "Hermes Agent 설치 → 소스 구조 → Oracle Cloud 무료 인스턴스 → OmniRoute 게이트웨이 → fallback 모델 설정까지, 지금까지 쓴 오픈소스 에이전트 구축 포스팅들을 한 흐름으로 엮은 종합 지도."
img: agent_stack_title.webp
date: 2026-09-11 20:00:00 +0900
last_modified_at: 2026-09-12 11:00:00 +0900
tags: [hermes, ai-agent, opensource, omniroute, oracle-cloud, llm-routing, llm] # add tag
related: llm
categories: tools
---

지금까지 오픈소스 AI 에이전트를 실제로 구축하며 쓴 글들이 흩어져 있다. [Hermes 설치]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html), [소스 구조]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html), [OCI 무료 인스턴스]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html), [OmniRoute 게이트웨이]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html). 이 글에서는 그 글들을 **설치 → 구조 → 인프라 → 게이트웨이 → 모델 설정** 순서로 엮어, "오픈소스로 에이전트 하나를 끝까지 굴리는 법"의 전체 지도를 그려본다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행한다.)

<!--more-->

> **TL;DR:** 오픈소스 AI 에이전트 구축은 (1) Hermes Agent 설치 → (2) 소스 구조 파악 → (3) Oracle Cloud 무료 인스턴스 확보(LLM 게이트웨이·MCP 서버 실험용) → (4) OmniRoute 게이트웨이로 무료 LLM 통합 → (5) fallback 중심의 모델 설정, 이 다섯 단계면 끝난다. 전 과정을 오픈소스 + 무료 티어 + 자체 호스팅으로 돌릴 수 있어, 벤더에 묶이지 않고 **오픈웨이트 모델 에이전트의 품질을 실험하는** 개인용 스택이 나온다.

## 1. 시작: Hermes Agent 설치

오픈소스 에이전트의 첫 단계는 [Hermes Agent](https://github.com/NousResearch/hermes-agent){:target="_blank"} 설치다. Nous Research가 만든 이 에이전트는 **"어떤 LLM이든 붙이고, 어디서든 돌아가는"** 설계를 노린다. Windows 11에 설치할 때 챙길 건 이 정도다.

- 셸 설치 스크립트로 `uv`·Python·venv·launcher를 한 번에 구성
- `hermes setup`으로 프로바이더·모델 연결
- CLI(`hermes`), TUI, 데스크톱 앱, 텔레그램 등 어느 표면에서 불러도 같은 코어가 돈다

설치는 git-bash(또는 WSL)에서 한 줄이면 된다.

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash

hermes setup     # 프로바이더·모델 연결 마법사
hermes doctor    # 상태 점검
hermes           # 실행
```

자세한 설치는 [Hermes Agent 설치부터 설정까지 — Windows에서 시작하기]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html)에 있다. 처음엔 OpenRouter 모델을 붙여 쓰다가 나중에 OmniRoute 라우팅으로 갈아탔는데, 그 얘기는 4절에서 한다.

## 2. 구조를 알면 커스터마이징이 쉬워진다

설치만으론 부족하다. "이 명령어가 실제로 어떤 코드를 타는가"를 알아야 플러그인·크론·게이트웨이를 제대로 확장할 수 있다. [소스 트리 투어]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html)에서 정리한 구조는 이렇다.

- **cli.py**: 진입점. 명령어가 실제 타는 모듈의 출발
- **conversation_loop**: 에이전트 코어 런타임. 한 턴이 끝나는 주기
- **gateway**: 텔레그램·디스코드 등 멀티 플랫폼 표면의 루트
- **cron**: 예약 작업 스케줄러

한 줄로 줄이면 "Hermes = 모델 무관 코어 + 갈아끼우는 플랫폼/모델"이다. 이 구조가 눈에 들어오면 크론에서 특정 모델을 고정하거나 게이트웨이에서 원하는 채널로 결과를 보내는 일이 명령어 하나로 끝난다.

## 3. 인프라: Oracle Cloud 무료 티어

Oracle Cloud 무료 인스턴스를 만든 목적은 분명했다 — **오픈소스 LLM 게이트웨이를 올릴 자리**가 필요했다. MCP 서버를 띄워 테스트하고, 오픈소스 LLM 라우터를 실험하려면 24시간 켜져 있는 서버가 있어야 하는데, 로컬 PC로는 감당이 안 된다.

- **Arm A1 + AMD Micro** (CPU/메모리 계열) — 프리티어인데 GPU는 없다
- RAM 1GB 이하에서도 에이전트 게이트웨이와 라우터는 충분히 돈다
- 시작은 [OCI 무료 인스턴스 시작하기]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html), 스토리지는 [블록볼륨 추가]({{site.baseurl}}/tools/2026/09/09/oci-block-volume-attach.html)로 확장

무료 인스턴스 두 대(oc01·oc02)에 데이터 마운트까지 붙여두니, 내 서버 한 대 없이도 MCP 서버·LLM 라우터 실험장을 상시 가동할 바닥이 깔렸다. 실제로 이 인스턴스에는 지오코딩 MCP 서버와 다음 절의 OmniRoute가 돌고 있다.

## 4. 게이트웨이: OmniRoute로 모델 하나로 통합

OmniRoute를 개인용으로 구축한 이유는 단순하다 — **무료 LLM을 거의 무제한으로 쓸 수 있다**는 얘기를 듣고서다. LLM 프로바이더(OpenRouter, NVIDIA, 커스텀)를 여럿 쓰다 보면 엔드포인트가 흩어지는 문제도 있는데, 오픈소스 AI 게이트웨이 [OmniRoute]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)가 이 둘을 함께 푼다.

- OCI 무료 인스턴스(RAM 951MB)에 OmniRoute를 올리고 **Caddy로 HTTPS**를 붙였다
- 스왑으로 OOM을 잡고, OCI iptables 함정은 우회했다
- 500개 가까운 모델을 **하나의 OpenAI 호환 엔드포인트**로 내보낸다

설치의 뼈대는 Docker 한 줄 + Caddyfile 두 줄이다(스왑 설정·iptables·systemd 구성 등 전체 절차는 [OmniRoute 셀프호스팅 글]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html) 참고).

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo docker run -d --name omniroute -p 20128:20128 \
  -v omniroute-data:/data --restart no diegosouzapw/omniroute
```

```text
# Caddyfile — 이 두 줄로 HTTPS 발급·갱신·리다이렉트까지 끝난다
awsome.duckdns.org {
    reverse_proxy localhost:20128
}
```

덕분에 Hermes는 프로바이더별 API 키를 일일이 챙길 것 없이 OmniRoute 엔드포인트 하나만 보면 된다.

한 가지 기대와 달랐던 점 — OmniRoute의 `auto/best-reasoning`은 이름과 달리 **요청 목적(추론·코딩·일반)을 보고 모델을 골라주는 라우팅이 아니다**. 추론 계열 모델 풀에서 가용한 것을 잡아줄 뿐이다. 목적별 라우팅이 필요해서 **별도로 `auto/route` 프록시를 만들어 테스트**해봤는데, 이 실험은 따로 글로 정리할 만한 분량이라 여기서는 존재만 언급해 둔다.

## 5. Hermes 에이전트의 모델 설정

마지막 단계는 Hermes에 어떤 모델을 물릴지 정하는 일이다. 핵심 전제는 이것이다 — **무료 모델은 컨텍스트 길이 제한과 사용량 리밋에 자주 걸린다.** 그래서 주 모델 하나로 버티는 구성은 오래 못 가고, **fallback 모델 설정이 필수**다.

fallback으로는 **gemma-4**, **nemotron-3 super/ultra** 등 여러 모델을 에이전트에 물려 테스트해보려고, [build.nvidia.com](https://build.nvidia.com/models){:target="_blank"}이 제공하는 무료 엔드포인트를 등록했다. NVIDIA API 키만 발급받아 `Authorization: Bearer <API_KEY>` 헤더로 호출하면 된다.

```
config.yaml (Hermes):
  fallback chain:
    1차: custom/auto/best-reasoning   # OmniRoute의 추론 모델 풀
    2차: nvidia/google/gemma-4-31b-it # 리밋 걸리면 여기로
```

NVIDIA Free Endpoint를 쓰는 순서는 이렇다.

1. [build.nvidia.com](https://build.nvidia.com/models){:target="_blank"}에서 회원가입/로그인
2. **API Keys** 메뉴에서 API 키 발급 (무료, 일일 제한 있음)
3. 모델 페이지(gemma-4-31b-it, nemotron-3-ultra)에서 **API 엔드포인트 확인** — `https://integrate.api.nvidia.com/v1/chat/completions`
4. 요청 헤더에 `Authorization: Bearer <API_KEY>` 포함
5. OpenAI 호환 포맷으로 요청 가능 — curl로 바로 확인할 수 있다:

```bash
curl https://integrate.api.nvidia.com/v1/chat/completions \
  -H "Authorization: Bearer $NVIDIA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"google/gemma-4-31b-it","messages":[{"role":"user","content":"ping"}]}'
```

Hermes 설정 예시:
```yaml
providers:
  nvidia:
    base_url: "https://integrate.api.nvidia.com/v1"
    api_key: "${NVIDIA_API_KEY}"   # .env에 저장
```

그리고 **크론 잡은 목적에 따라 모델 프로바이더를 따로 지정할 수 있다.** 수집처럼 가벼운 작업은 gemma-4로, 초안 생성이나 심층 분석은 고성능 모델로 — 잡마다 명시적으로 지정해 두면 전역 설정과 무관하게 각자 제 모델로 돈다.

## 6. 이 조합으로 무엇을 할 수 있나

이 스택은 프로덕션 시스템이라기보다 **실험장**이다. 우선 이 조합으로 AI 에이전트의 기능을 하나씩 테스트해보고, 작은 작업들을 실제로 맡겨볼 수 있다.

- **오픈웨이트 모델 에이전트의 품질 측정**: 프론티어 모델이 아니라 gemma-4, nemotron-3 같은 다양한 오픈웨이트 모델로 에이전트를 구성했을 때 어느 수준의 품질이 나오는지가 이 스택의 핵심 관찰 대상이다. 실제로 이 블로그의 초안 수집·작성 크론이 그 실험이다.
- **남은 숙제**: 오픈웨이트 모델로 업무용 에이전트를 제대로 구성하려면 어떤 기술(도구 호출 안정화, 컨텍스트 관리, 검증 루프 등)이 더 필요한지는 계속 공부해야 할 부분이다.

## 7. 운영 관점의 한계

이 스택은 "돌아간다"와 "운영된다" 사이에 있다. 실험장으로 쓰면서 확인한, 프로덕션과의 거리를 솔직하게 적어둔다.

- **가용성은 무료 티어 세 개의 곱이다.** OCI 프리티어(회수 가능성) × OmniRoute가 중계하는 무료 모델 풀(리밋·품질 변동) × NVIDIA Free Endpoint(일일 제한). fallback을 2단 뒀지만 둘 다 무료 자원이라, 무료 풀 전체가 혼잡한 시간대에는 같이 막힐 수 있다. 정석은 최후단에 소량 과금 모델을 서킷브레이커로 두는 것 — "전부 무료"라는 이 스택의 정체성과 상충하는 트레이드오프다.
- **게이트웨이가 단일 장애점이다.** OmniRoute가 죽으면(RAM 951MB + 스왑 의존이라 OOM 재발 여지가 있다) 에이전트 전체가 모델을 잃는다. 헬스체크와 자동 재시작(systemd `Restart=always` 수준이라도)은 아직 안 붙였다.
- **관측 가능성이 없다.** 어떤 요청이 어떤 모델로 갔고 리밋에 몇 번 걸렸는지 기록하지 않으면, 6절의 "오픈웨이트 에이전트 품질 측정"은 인상 비평에 머문다. 라우팅 로그와 실패율 카운터가 다음 순위의 작업이다.
- **보안 베이스라인이 얇다.** OmniRoute에는 Caddy로 HTTPS를 붙였지만, 인스턴스에 함께 띄운 실험용 서비스까지 전부 인증·TLS 뒤에 있는 건 아니다. 무료 LLM 게이트웨이는 유출되면 남이 내 쿼터를 태우는 자산이므로, 공개 포트 최소화·API 키 인증·방화벽 기본 차단이 숙제로 남아 있다.
- **재현성 주의.** OmniRoute·Hermes 모두 빠르게 변하는 프로젝트다. 이 글은 2026년 9월 초 시점의 구성이며, 몇 달 뒤에는 명령과 설정 키가 달라져 있을 수 있다.

## 마무리

이 다섯 단계를 다 붙이면 **오픈소스만으로 굴러가는 에이전트 스택**이 완성된다.

| 단계 | 구성 요소 | 담당 글 |
|------|----------|---------|
| 1. 설치 | Hermes Agent | [Hermes 설치]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html) |
| 2. 구조 | 소스 트리 파악 | [소스 구조]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html) |
| 3. 인프라 | OCI 무료 인스턴스 | [OCI 시작]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html), [블록볼륨]({{site.baseurl}}/tools/2026/09/09/oci-block-volume-attach.html) |
| 4. 게이트웨이 | OmniRoute + Caddy | [OmniRoute 셀프호스팅]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html) |
| 5. 모델 | fallback 모델 설정 (NVIDIA Free Endpoint) | 이 글 |

벤더에 묶이지 않고, 무료 티어로 항상 켜져 있고, 엔드포인트 하나로 모든 모델을 쓴다. 이 조합이면 개인이든 사내든 오픈소스 에이전트를 본격적으로 굴려볼 준비는 끝난 셈이다. 다음 편에서는 크론 기반 자동 블로그 분석 파이프라인(수집 → 초안 생성)을 실제로 굴리며 부딪힌 문제와 고친 과정을 풀어보겠다.

## 참고
- [Hermes Agent 설치]({{site.baseurl}}/tools/2026/07/25/hermes_agent_setup.html)
- [Hermes 소스 트리]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html)
- [OCI 무료 인스턴스 시작]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)
- [OCI 블록볼륨 추가]({{site.baseurl}}/tools/2026/09/09/oci-block-volume-attach.html)
- [OmniRoute 셀프호스팅]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/PSpf_XgOM5w){:target="_blank"}
