---
layout: post
title: "외부 공인 IP 확인하여 슬랙에 공유하기"
description: "외부 공인 IP 확인 및 슬랙에 공유하는 방법을 기록한다."
img: "M_photo-cabling.jpg"
date: 2020-09-08 21:00:00 +0900
last_modified_at: 2021-04-07 15:00:00 +0900
tags: [duckdns, ipify, slack, webhook, terminal] # add tags 
related: terminal
categories: tools
---

공인 IP가 변경되는 환경에서 (주로 노트북에 WSL을 구동하여 시험하는 환경)에서 외부 공인IP가 변경되면 슬랙 채널로 메세지를 보낸다. 

공인 IP Address는 [ipify.org](https://www.ipify.org) 서비스를 사용한다. 

- [ipify: 300억 요청 처리, 그 너머로](https://edykim.com/ko/post/ipify-to-30-billion-and-beyond/) 


## ipify API로 공인 IP 확인 

```bash
❯ curl 'https://api.ipify.org?format=text'
106.240.xxx.xxx%
```

## 공인 IP 변경되면 Slack에 공유 

공인 IP가 바뀌었을 경우 슬랙채널에 공유하기 위해 기존 IP를 저장하고 현재 IP와 다르면 슬랙에 공유한다. 

{% gist butteryoon/0047eea87040bf3584a7c54a7a3797c5 %} 

## 공인 IP를 확인하는 다른 방법 

### - "nslookup" 툴을 이용하는 방법 

```powershell
❯ nslookup myip.opendns.com. resolver1.opendns.com
Server:  resolver1.opendns.com
Address:  208.67.222.222

Non-authoritative answer:
Name:    myip.opendns.com
Address:  106.240.xxx.xxx
```

### - "dig" 툴을 이용하는 방법 

```bash
❯ dig @resolver4.opendns.com myip.opendns.com +short
106.240.xxx.xxx
```

## [ifconfig.co](https://ifconfig.co) 서비스 

ipfy와 비슷하게 IP Address를 text로 응답하며 오픈소스로 공개되어 있다. 

```powershell
❯ curl.exe ifconfig.co
106.240.xxx.xxx
``` 

json 포맷으로 요청하면 IP와 국가, 지역, 위치정보 등을 확인할 수 있다. 

```powershell
❯ curl ifconfig.co/json
{
  "ip": "106.240.xxx.xxx",
  "ip_decimal": 17941.....,
  "country": "South Korea",
  "country_iso": "KR",
  "country_eu": false,
  "region_name": "Gyeonggi-do",
  "region_code": "41",
  "zip_code": "xxx",
  "city": "xxx",
  "latitude": 37.3907,
  "longitude": 126.9167,
  "time_zone": "Asia/Seoul",
  "asn": "AS3786",
  "asn_org": "LG DACOM Corporation",
  "user_agent": {
    "product": "curl",
    "version": "7.55.1",
    "raw_value": "curl/7.55.1"
  }
}
```

## 참고 URL 

- [Duckdns Linux cron](https://www.duckdns.org/install.jsp)
- [A Simple Public IP Address API](https://www.ipify.org)
- [Using the Slack Web API](https://api.slack.com/web)
- [개발자 친화적인 "What is my ip"](https://news.hada.io/topic?id=3986)
