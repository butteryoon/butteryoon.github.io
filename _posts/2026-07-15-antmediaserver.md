---
layout: post
comments: true
title: "Ant Media Server - 오픈소스 저지연 스트리밍 서버"
description: "WebRTC, SRT, RTMP, HLS를 지원하는 미디어 서버인 Ant Media Server를 소개한다."
img: AntMediaServer-Cover.png
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-15 00:00:00 +0900
tags: [AntMediaServer, WebRTC, HLS, SRT, streaming] # add tag
related: streaming
categories: dev
redirect_from:
  - /dev/2026/07/14/antmediaserver.html
---

2020년에 링크만 모아두고 묵혀둔 초안을 보완해서 발행한다. 당시에는 HLS 스트리밍용으로 잠깐 살펴봤던 Ant Media Server가 지금은 어떤 모습인지 다시 정리해본다.

<!--more-->

## Ant Media Server란

[Ant Media Server](https://antmedia.io)는 초저지연 스트리밍에 초점을 맞춘 미디어 서버다. WebRTC 기반으로 약 0.5초 수준의 지연을 목표로 하며, 입력(ingest)으로 WebRTC, RTMP, SRT, RTSP, WHIP 등을 받고 출력으로 WebRTC, HLS, CMAF 등을 지원한다. 어댑티브 비트레이트, 트랜스코딩, 녹화 같은 미디어 서버의 기본기도 갖추고 있다.

즉, OBS에서 RTMP나 SRT로 송출한 스트림을 받아 브라우저에서는 WebRTC나 HLS로 재생하는 구성을 서버 하나로 처리할 수 있다.

## Community vs Enterprise

두 가지 에디션으로 제공된다.

- **Community Edition** : 오픈소스(GitHub 공개)이며 RTMP/SRT 입력, HLS 재생, 녹화 등 일반적인 스트리밍 용도와 개발/테스트에 적합하다.
- **Enterprise Edition** : WebRTC 초저지연 재생, 어댑티브 비트레이트, 클러스터링(스케일 아웃), GPU 가속, 기술지원 등이 추가된 유료 버전이다.

WebRTC로 대규모 시청자에게 서비스하려면 Enterprise가 필요하고, HLS 위주의 단순 구성이라면 Community로 충분하다.

## 설치

가장 간단한 방법은 Docker다. Community Edition은 GitHub 릴리즈에서 zip을 받아 이미지를 빌드하거나, 리눅스에서는 설치 스크립트 한 줄로 설치할 수 있다.

```
# 설치 스크립트 방식 (Ubuntu 등)
wget https://raw.githubusercontent.com/ant-media/Scripts/master/install_ant-media-server.sh
sudo bash install_ant-media-server.sh
```

설치 후 `http://서버주소:5080` 으로 접속하면 웹 관리 콘솔에서 앱 생성, 스트림 관리, 재생 테스트까지 할 수 있다. AWS, Azure, GCP 등 클라우드 마켓플레이스의 1-Click 이미지나 Kubernetes 배포도 공식 지원한다.

## 참고
- [antmedia.io](https://antmedia.io){:target="_blank"}
- [Repository: ant-media/Ant-Media-Server](https://github.com/ant-media/Ant-Media-Server){:target="_blank"}
- [Ant Media Documentation](https://docs.antmedia.io/){:target="_blank"}
- [Docker 설치 가이드](https://docs.antmedia.io/guides/installing-on-linux/ams-docker-installation/){:target="_blank"}
