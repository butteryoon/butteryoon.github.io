---
layout: post
comments: true
title: "TailScale을 이용한 개인 네트워크 구성하기" 
description: "tailscale 서비스를 이용해서 프라이빗 네트워크를 구성해보자."
img: "M_ssh_tunneling.jpg"
date: 2023-11-14 01:00:00 +0900
last_modified_at: 2023-11-14 01:00:00 +0900
tags: [ssh, tailscale, vpn, CloudFlare, remote desktop] # add tag
related: ssh
categories: tools
---

PC에 테스트 환경을 만들고 외부에서 접근해야 할 때 일반적으로 VPN을 구성하고 외부에 제공될 서비스의 IP/PORT를 등록해서 일정 기간동안 사용한다.

사실 내부의 서버에 SSH 접속가능한 서버 한대만 있으면 [SSH 멀티플렉싱 기능을 이용하여 내부 서버에 접속하기]({{ site.baseurl }}{% link _posts/2021-05-16-ssh_bastion.md %}) 같은 방법으로 내부 서버를 편하게 이용할 수 있다. 

하지만 위와 같은 환경이 안되거나 설정이 어려운 경우, 또는 급하게 다른 사용자에게 로컬PC에서 개발중인 특정 서비스를 보여주거나 외부에서 회사 내부의 PC를 연결해야 할 때 [tailScale](https://tailscale.com/) 서비스를 이용하면 별도의 설정이나 기다림 없이 편하게 사용할 수 있다. 

<!--more-->

## tailscale 서비스

[tailScale](https://tailscale.com/) 서비스는 PC나 서버에 설치해서 로그인하면 해당 디바이스는 100.x.x.x의 IP를 할당 받고 동일계정으로 로그인된 디바이스간 보안 네트워크를 만들어주는 서비스로 **Wireguard** 프로토콜을 사용해서 만든 클라우드 서비스다. 


### Free Plan

최근에 [프리 요금제](https://tailscale.com/pricing/)는 3명의 사용자가 100개의 디바이스를 등록하여 쓸 수 있도록 확장되어 사용자를 초대하여 자신의 tailscale 네트워크에 접속 권한을 줄 수 있다. (일단 써보자)

> **어드민콘솔**의 Users 탭에서 **Invites users** 버튼으로 사용자 추가.
> 팀 내에서 개발 중인 내용을 상호 검토하려고 할 때 유용하게 쓸 수 있을 듯. 

## install

[TailScale quickstart](https://tailscale.com/kb/1017/install/)를 참고한다. 

간단하게 [Download Tailscale](https://tailscale.com/download) 페이지에서 플랫폼에 맞는 패키지 버전을 설치하고 google 계정으로 로그인하면 **어드민콘솔**에서 아래와 같이 등록된 디바이스 목록을 볼 수 있다. 

| MACHINE | ADDRESS | VERSION | LAST SEEN |
| :---: | :---: | :---: | :---: |
| laptop-home | 100.x.x.11 | 1.42.0 | Oct 30 |
| laptop-work | 100.x.x.12 | 1.42.0 | Connected |

LAST SEEN 에 "Connected" 라고 보이는 디바이스는 해당 IP 또는 MACHINE에 표시되는 이름으로 접근할 수 있다. 

> tailscale 설치 버전이 다르면 접속이 되지 않을 수도 있어 최신 버전으로 유지해주는게 좋다.  

접근하고 싶은 디바이스에 **tailscale**을 설치 후 동일계정으로 로그인 해 두면 해당 **어드민콘솔**에 해당 목록이 추가되고 해당 디바이스에는 자유롭게 접근이 가능하다. 

## 윈도우즈PC 접근

회사에 놓고 쓰는 데스크탑PC에 **tailscale**을 깔아두고 [원격 데스크탑](https://support.microsoft.com/ko-kr/windows/%EC%9B%90%EA%B2%A9-%EB%8D%B0%EC%8A%A4%ED%81%AC%ED%86%B1%EC%9D%84-%EC%82%AC%EC%9A%A9%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95-5fe128d5-8fb1-7a23-3b8a-41e636865e8c)을 설정해 두면 [MS RemodeDesktop](https://apps.microsoft.com/detail/9WZDNCRFJ3PS?hl=en-gb&gl=GB) 과 같은 원격데스크탑 앱으로 접근하여 쓸 수 있다. 

생각보다 느리지 않아 원할하게 사용 가능하지만 PC가 절전이나 슬립모드로 빠지면 접속이 안될 때가 있으니 **전원설정**을 확인해야 한다. 

## hosts

**Windows11** 의 hosts 파일을 보면 아래와 같이 **tailscale**의 IP와 호스트이름이 저장되어 있어 접속 할 때 hostname 으로도 접속이 가능하다. ( IP는 아무래도 안 외어진다. )

```
100.x.x.x7 butterphone.tail2xxxx.ts.net. butterphone
100.x.x.x1 desktop-home.tail2xxxx.ts.net. desktop-home
100.x.x.xx1 laptop-home.tail2xxxx.ts.net. laptop-home
100.x.x.x2 laptop-work.tail2xxxx.ts.net. laptop-work
100.x.x.x8 macmini.tail2xxxx.ts.net. macmini

# TailscaleHostsSectionEnd
```

## TL;DR

자신의 PC에 개발중인 프로토타입 서비스 등을 임시로 보여줘야 할 때 tailscale 서비스를 이용해보자. ( 관리자에게 아쉬운 부탁을 안 해도 된다. )

[tunnelto.dev](https://tunnelto.dev), [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) 서비스도 다음 기회에 알아봐야 겠다. 

VSCode용 플러그인을 깔고 **tailscale SSH**를 활성화 하면 연결된 디바이스에 바로 접속해서 작업이 가능하다고 한다.(아직 안해봄)

## 참고

- [tailscale.com](tailscale.com){:target="_blank"}
- [What is TailScale](https://ddii.dev/kubernetes/what-is-tailscale/){:target="_blank"}
- [tailscale이 무료 요금제를 유지하는 방법](https://news.hada.io/topic?id=6171){:target="_blank"}
- [테일스케일은 어떻게 작동하는가](https://tailscale.tistory.com/5)