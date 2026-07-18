---
layout: post
comments: true
title: "방화벽 뒤 PC 접속을 위한 터널링 도구들"
description: "ngrok, Cloudflare Tunnel, Tailscale 등 방화벽 뒤의 PC나 로컬 서비스를 외부에서 접속할 수 있게 해주는 터널링 도구들을 비교해본다."
img: cloud-title.webp
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-15 00:00:00 +0900
tags: [ngrok, tailscale, CloudFlare, tunnel, remote desktop] # add tag
related: tunnel
categories: tools
redirect_from:
  - /tools/2026/07/14/tunneling_tools.html
---

2023년에 작성해두었던 초안을 보완해서 발행한다. 방화벽으로 접속이 제한된 네트워크 안의 PC나 로컬에서 개발 중인 서비스를 외부에서 접속해야 할 때 사용할 수 있는 터널링 도구들을 정리해본다.

<!--more-->

## 터널링 도구가 필요한 경우

회사나 집의 PC는 대부분 공유기나 방화벽 뒤에 있어서 외부에서 직접 접속할 수 없다. 포트포워딩이나 VPN을 구성하면 되지만 네트워크 관리 권한이 없거나 설정이 번거로운 경우가 많다. 이럴 때 터널링 도구를 쓰면 내부에서 외부의 중계 서버로 아웃바운드 연결을 만들어두고, 그 연결을 통해 거꾸로 내부 서비스에 접속할 수 있다.

## ngrok

[ngrok](https://ngrok.com/)은 이 분야에서 가장 유명한 도구다. 로컬에서 `ngrok http 8080` 한 줄이면 `https://xxxx.ngrok-free.app` 같은 공개 URL이 만들어져서 웹훅 테스트나 데모 공유에 특히 편하다.

- 무료 플랜은 랜덤 도메인이 할당되고, 고정 도메인이나 트래픽 확장은 유료다.
- HTTP뿐 아니라 TCP 터널도 지원해서 SSH나 RDP 포트도 열 수 있다(무료는 제한적).
- 설치와 사용이 가장 간단한 대신, 무료로 계속 쓰기에는 제약이 많은 편이다.

## Cloudflare Tunnel (cloudflared)

[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)은 `cloudflared` 데몬을 내부 PC에 설치해서 Cloudflare 네트워크로 아웃바운드 터널을 만드는 방식이다.

- 자신의 도메인을 Cloudflare에 연결해두면 `myapp.example.com` 같은 주소로 내부 서비스를 무료로 상시 노출할 수 있다.
- 도메인 없이 임시로 써보려면 `cloudflared tunnel --url http://localhost:8080` 한 줄로 `trycloudflare.com` 임시 URL도 만들 수 있다(Quick Tunnel).
- Cloudflare Access와 조합하면 접속자 인증까지 걸 수 있어서 개인 서비스를 안전하게 운영하기 좋다.
- 대신 제대로 쓰려면 도메인과 Cloudflare 설정이 필요해서 초기 진입장벽은 ngrok보다 높다.

## Tailscale

[Tailscale](https://tailscale.com/)은 위 두 가지와 성격이 조금 다르다. 특정 포트를 공개 URL로 노출하는 게 아니라, 내 디바이스들끼리 WireGuard 기반의 프라이빗 네트워크(메시 VPN)를 만들어준다. 각 디바이스에 100.x.x.x 대역의 IP가 할당되고, 어디에 있든 서로 직접 접속할 수 있다.

외부의 불특정 다수에게 서비스를 보여주는 용도보다는 "내 PC들끼리 어디서나 SSH/RDP로 붙는" 용도에 가장 잘 맞는다. 자세한 사용법은 예전에 정리한 [TailScale을 이용한 개인 네트워크 구성하기]({{site.baseurl}}/tools/2023/11/14/tailscale.html) 글을 참고한다.

## tunnelto.dev는?

초안을 쓸 당시 관심을 가졌던 [tunnelto.dev](https://tunnelto.dev/)는 Rust로 만들어진 오픈소스 터널링 도구인데, 2026년 현재도 서비스는 살아있다. 무료 플랜이 있고 고정 서브도메인은 월 $4부터다. 다만 ngrok이나 Cloudflare Tunnel에 비해 커뮤니티나 기능 발전이 활발한 편은 아니라서, 지금 새로 시작한다면 아래 세 가지 중에서 고르는 것을 추천한다.

## 비교 정리

| 도구 | 무료 여부 | 방식 | 주 용도 |
| :--- | :--- | :--- | :--- |
| ngrok | 무료(랜덤 도메인, 제한 있음) | 공개 URL 터널 | 웹훅 테스트, 임시 데모 |
| Cloudflare Tunnel | 무료(자기 도메인 필요, Quick Tunnel은 도메인 불필요) | 공개 URL 터널 | 개인 서비스 상시 노출 |
| Tailscale | 무료(3 사용자 / 100 디바이스) | 메시 VPN | 내 디바이스 간 SSH/RDP 원격 접속 |

정리하면, 잠깐 보여줄 거면 **ngrok**, 도메인 걸고 계속 운영할 거면 **Cloudflare Tunnel**, 내 장비들 원격 접속용이면 **Tailscale**이다.

## 참고

- [ngrok](https://ngrok.com/){:target="_blank"}
- [Cloudflare Tunnel 문서](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/){:target="_blank"}
- [Tailscale](https://tailscale.com/){:target="_blank"}
- [tunnelto.dev](https://tunnelto.dev/){:target="_blank"}
- [tunnelto GitHub](https://github.com/agrinman/tunnelto){:target="_blank"}
