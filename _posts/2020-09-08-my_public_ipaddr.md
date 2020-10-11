---
layout: post
title: "외부 공인 IP 확인하여 슬랙에 공유하기"
fig-caption: "외부 공인 IP 확인하여 슬랙에 공유하기"
img: "M_photo-cabling.jpg"
date: 2020-09-08 21:00:00 +0900
tags: [duckdns, ipify, slack, webhook] # add tag
categories: tools
---

공인 IP가 변경되는 환경에서 (주로 노트북에 WSL을 구동하여 시험하는 환경)에서 외부 공인IP가 변경되면 슬랙 채널로 메세지를 보낸다. 

외부 IP는 [ipify.org](https://www.ipify.org) 서비스 사용

{% gist butteryoon/0047eea87040bf3584a7c54a7a3797c5 %} 



## 참고 URL 

- [Duckdns Linux cron](https://www.duckdns.org/install.jsp)
- [A Simple Public IP Address API](https://www.ipify.org)
- [Using the Slack Web API](https://api.slack.com/web)