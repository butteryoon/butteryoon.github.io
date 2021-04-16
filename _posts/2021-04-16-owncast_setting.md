---
layout: post
title: "owncast 라이브스트리밍 서버"
description: "Owncast 라이브스트리밍 서버 설치 및 테스트"
img: m_owncast_title.webp
date: 2021-04-16 18:00:00 +0900
last_modified_at: 2021-04-16 20:00:00 +0900
tags: [streaming, owncast, rtmp] # add tag
related: streaming
categories: dev
---

[Owncast](https://owncast.online)는 셀프호스팅으로 라이브스트리밍과 웹 채팅 서비스를 구축할 수 있는 오픈소스 이며 OBS와 같은 툴로 라이브 방송을 할 수 있다. 

오라클 클라우드 인트턴스(Ubuntu 20.04 Free tier)에 Owncast를 설치하고 간단하게 동작을 테스트 해 본다. 
<!--more-->

## Install 

인스톨 스크립트를 제공하여 curl로 간단하게 설치 가능한데 아래의 필수 유틸리티는 꼭 있어야 한다. 

```bash
  requireTool "curl"
  requireTool "unzip"
  requireTool "tar"
  requireTool "which"
```

ffmpeg은 install.sh 스크립트에서 체크하여 다운로드 한다. 

```bash
Owncast Installer v0.0.6

Created directory  [✓]
Downloaded Owncast v0.0.6 for linux  [✓]

Success! Run owncast by changing to the owncast directory and run ./owncast.
The default port is 8080 and the default streaming key is abc123.
Visit https://owncast.online/docs/configuration/ to learn how to configure your new Owncast server.
```

## Starting 

설치가 완료되면 owncast 디렉토리로 이동 후 owncast 바이너리를 실행한다. 

RTMP 포트는 1935 웹서버 포트는 8080으로 초기 설정되어 있다. 

```bash
ubuntu@instance-20210116-0003:~/owncast$ ./owncast
INFO[2021-04-16T22:12:40+09:00] Owncast v0.0.6-linux-64bit (36645437ef17f622c8926fc32a2bf2de27a6e8d7)
INFO[2021-04-16T22:12:43+09:00] RTMP is accepting inbound streams on port 1935.
INFO[2021-04-16T22:12:43+09:00] Web server is listening on port 8080.
INFO[2021-04-16T22:12:43+09:00] The web admin interface is available at /admin.
```

## Settings

http://hostname:8080/admin 페이지에서 기본 설정을 확인할 수 있고 기본 패스워드는 "abc123" 이다. 

- Configuration > Server Setup 메뉴에서 "Stream Key" 를 설정한다.   
- "Stream Key"는 admin 페이지 접속과 RTMP 스트림키로 사용된다.  

스트림키는 바꾸고 나머지 포트는 기본설정으로 둔다. 

![owscast config]({{site.baseurl}}/assets/img/m_owncast_server_config.webp)

## Start Streaming 

OBS를 이용해서 방송설정을 하고 PC의 화면을 송출해본다. 

방송 설정은 아래와 같다. 

- 서비스 : 사용자지정 
- 서버 : rtmp://yourserver:1935/live/
- 스트림 키 : admin 페이지에서 변경한 "Stream Key" 

스트림이 Owncast 서버로 정상 스트리밍 되면 admin 페이지에서 아래처럼 스트림의 상태를 볼 수 있다. 

![owscast Status]({{site.baseurl}}/assets/img/m_owncast_status.webp)

## Stream View 

http://hostname:8080 으로 접속하면 아래처럼 영상플레이어에서 영상이 재생되고 오른쪽에서 채팅을 할 수 있다. 

![owscast Stream View]({{site.baseurl}}/assets/img/m_owncast_rtmp_stream.webp)

## TL;DR

오라클 클라이언트 인스턴스 프리티어가 메모리가 1GB 뿐이라 FullHD 해상도로 스트리밍을 하면 메모리 Full 상태가 된다. 720p로 스트리밍하면 간당간당하게 버티는 듯 하다. 

개인용도의 실시간 스트리밍 서버로 사용을 위해 간단하게 구축하기 적합한 걸로 보인다. 설치 및 설정도 간단하게 적용된다. 

기본으로 제공되는 커스터마이징 기능과 Web Hook 기능도 시간되면 확인 해보자.

## 참고 URL

- [Owncast-오픈소스 라이브 스트리밍 서버](https://news.hada.io/topic?id=3450){:target="_blank"}
