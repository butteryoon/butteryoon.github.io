---
layout: post
comments: true
title: "오라클 클라우드 무료 티어에 OmniRoute 셀프호스팅 — RAM 1GB로 AI 게이트웨이 돌리기"
description: "Oracle Cloud 무료 인스턴스(RAM 951MB)에 오픈소스 AI 게이트웨이 OmniRoute를 올리고 Caddy로 HTTPS를 붙인 구축기. 스왑으로 OOM 잡기, OCI iptables 함정, 그리고 질문 유형별 자동 분배를 위해 파이썬 표준 라이브러리로 직접 만든 분류 프록시까지."
img: omniroute-oci-title.webp
date: 2026-09-06 20:40:00 +0900
last_modified_at: 2026-09-06 20:40:00 +0900
tags: [omniroute, oracle-cloud, free-tier, ai-gateway, caddy, self-hosting, llm-routing] # add tag
related: llm
categories: tools
---

[오라클 클라우드 프리티어 시작하기]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)를 쓴 지 5년이 넘었는데, 그 무료 인스턴스가 여전히 현역이다. 정확히는 기존 인스턴스가 종료(Terminated)되는 바람에 새로 만들면서 이참에 **오픈소스 AI 게이트웨이 OmniRoute**를 올려 개인용 LLM 라우터를 구축했다. RAM 951MB짜리 무료 인스턴스에서 500개 가까운 모델을 하나의 OpenAI 호환 엔드포인트로 쓰기까지의 기록이다.

<!--more-->

> **TL;DR:** Ubuntu 26.04 무료 인스턴스(2 vCPU, RAM 951MB)에 ① 스왑 4GB로 OOM 방지 → ② OmniRoute를 Docker + systemd로 상주 → ③ Caddy 두 줄로 HTTPS 리버스 프록시(Let's Encrypt 자동) → ④ OmniRoute가 안 해주는 "질문 유형별 자동 분배"는 **파이썬 표준 라이브러리만으로 분류 프록시를 직접 제작**해 해결했다. 최종 사용은 Base URL 하나에 model `auto/route` — 코딩 질문은 코딩 모델로, 추론 문제는 추론 모델로 알아서 간다.

## 0. 배경 — 인스턴스가 죽어 있었다

기존 인스턴스(awsome.duckdns.org)가 어느 날 Terminated 상태가 되어 있었다. 정황상 RAM 1GB 미만 환경에서 스왑 없이 돌리다 **OOM으로 죽었던 것**으로 추정한다. 새 인스턴스(Ubuntu 26.04 LTS, 2 vCPU, RAM 951MB, 디스크 45GB)를 만들었고 duckdns가 새 공인 IP를 자동 갱신해줘서 기존 SSH 설정은 그대로 살았다.

교훈부터 반영했다 — **스왑 4GB**:

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile && sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
```

`swappiness=10`으로 평소엔 RAM을 쓰고 스왑은 OOM 방어선으로만 둔다.

## 1. OmniRoute 설치 — Docker + systemd

[OmniRoute](https://github.com/diegosouzapw/OmniRoute){:target="_blank"}는 290개 이상 프로바이더·500개 이상 모델을 하나의 OpenAI 호환 API로 묶어주는 오픈소스 게이트웨이다. [OpenRouter]({{site.baseurl}}/tools/2026/07/15/openrouter_free.html) 같은 서비스의 셀프호스팅 판이라고 보면 된다.

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo docker run -d --name omniroute -p 20128:20128 \
  -v omniroute-data:/data --restart no diegosouzapw/omniroute
```

재시작 관리는 Docker와 systemd가 서로 싸우지 않도록 **docker restart 정책은 no로 두고 systemd가 단독 관리**한다:

```ini
# /etc/systemd/system/omniroute.service
[Service]
ExecStart=/usr/bin/docker start -a omniroute
ExecStop=/usr/bin/docker stop omniroute
Restart=always
```

RAM 951MB 환경에서도 스왑 덕에 안정적으로 돈다.

## 2. HTTPS — Caddy 두 줄, 그리고 OCI의 함정

OCI Ubuntu 이미지에는 함정이 있다. 클라우드 콘솔의 Security List만 열면 되는 게 아니라, **호스트 iptables가 22번 외 전부 REJECT**로 깔려 있다:

```bash
sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 443 -j ACCEPT
sudo apt install iptables-persistent   # 재부팅 영속화
```

콘솔 Security List에도 80/443 Ingress를 추가한 뒤, Caddy로 리버스 프록시를 세운다. Caddyfile은 정말 두 줄이다:

```text
awsome.duckdns.org {
    reverse_proxy localhost:20128
}
```

Let's Encrypt 인증서 발급·갱신은 Caddy가 알아서 한다. HTTPS가 붙은 뒤에는 20128 직접 노출 포트를 Security List에서 제거해 **HTTPS 경유만 허용**했다.

## 3. API 사용 — auto 라우팅의 실체

이제 OpenAI 호환 클라이언트에서 Base URL만 바꾸면 된다:

```text
Base URL: https://awsome.duckdns.org/v1
```

사용 가능한 모델은 494개. 그중 눈여겨볼 것은 `auto/*` 자동 라우팅 별칭 38개다 — `auto/best-coding`, `auto/best-reasoning`, `auto/best-fast`, `auto/cheap` 같은 식이다. 주의할 점 하나: 이 auto 라우팅은 **요청 내용을 분석해서 고르는 게 아니라**, 지연시간×비용×성공률×컨텍스트 적합성 점수로 해당 카테고리 안에서 모델을 고르는 방식이다. 즉 "코딩 질문이니 코딩 모델로"는 스스로 못 한다.

## 4. 분류 프록시 직접 만들기 — 이 글의 하이라이트

그래서 "질문 유형별 자동 분배"는 직접 만들었다. 파이썬 **표준 라이브러리만으로** 작은 프록시(`/opt/route-proxy/route_proxy.py`, 20129 포트, systemd 서비스)를 세우고 Caddy에 경로 하나를 추가했다:

```text
handle_path /router/* {
    reverse_proxy localhost:20129
}
```

동작은 단순하다:

1. 요청의 model이 `auto/route`면 → `auto/best-fast`에게 질문을 **coding / reasoning / vision / chat 중 1단어로 분류**시킨다 (~1초)
2. 분류 결과에 맞는 `auto/best-*`로 원 요청을 포워딩
3. 다른 모델명이나 스트리밍 요청은 그대로 통과, 분류 실패 시 chat 폴백

```text
최종 사용: Base URL https://awsome.duckdns.org/router/v1, model "auto/route"
```

### 트러블슈팅 — 추론 모델에게 분류를 시키면 생기는 일

분류가 계속 빈 응답으로 실패했는데, 원인이 재밌었다. 분류를 맡긴 `auto/best-fast`가 **추론(reasoning) 모델**이어서 분류 프롬프트가 길면 **사고 토큰이 max_tokens를 다 소진**해 정작 `content`가 null로 오는 것이었다. 두 가지로 해결했다:

- 분류 프롬프트를 한 문장으로 압축
- 그래도 content가 비면 `reasoning_content`의 꼬리에서 카테고리 단어를 **역방향 스캔**하는 폴백 추가

테스트 결과: 데코레이터 질문 → coding, 평균속력 계산 → reasoning, 인사 → chat으로 정확히 분류됐다. 추론 모델을 유틸리티 작업(분류·추출)에 쓸 때는 max_tokens와 사고 토큰의 관계를 반드시 계산에 넣어야 한다는 교훈.

## 5. 자잘한 팁

- **Windows에서 한글 POST 테스트**: `curl -d "한글..."`은 CP949 인코딩이 깨진다 — UTF-8로 저장한 파일을 `--data-binary @file`로 보내거나 파이썬으로 테스트하는 편이 낫다.
- API 키는 어디에도 하드코딩하지 말 것 — OmniRoute 관리 화면에서 프로바이더별로 등록하고 게이트웨이 자체 키로만 노출한다.

## 마무리

무료 인스턴스 한 대로 "494개 모델을 향한 단일 엔드포인트 + 질문 유형 자동 분배"가 완성됐다. 구성 요소는 전부 흔한 것들(Docker, systemd, Caddy, 파이썬 표준 라이브러리)이고 특별한 건 조합뿐이다. 5년 전 [프리티어 시작 글]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)의 후속편으로, 무료 티어의 쓸모가 정적 서버에서 개인 AI 인프라로 넘어왔다는 게 개인적인 감상이다.

## 참고

- [OmniRoute (GitHub)](https://github.com/diegosouzapw/OmniRoute){:target="_blank"}
- [Caddy 문서 — reverse_proxy](https://caddyserver.com/docs/caddyfile/directives/reverse_proxy){:target="_blank"}
- [오라클 클라우드 프리티어 시작하기 (관련글)]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)
- [OpenRouter 무료 모델 (관련글)]({{site.baseurl}}/tools/2026/07/15/openrouter_free.html)
