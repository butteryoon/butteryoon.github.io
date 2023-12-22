---
layout: post
comments: true
title: "프로세스 로드밸런싱"
description: "iptables로 서버 내 N개의 서비스 트래픽 분배하기"
img: command-title.webp
date: 2023-12-23 00:50:00 +0900
last_modified_at: 2023-12-23 00:50:00 +0900
tags: [command-line, iptables , masquerade, nat, port forward, 라우드로빈, 로드밸런싱] # add tag
related: command-line
categories: tools
---

한 대의 서버에 같은 서비스를 다른 포트로 여러개를 실행하고 요청을 분배하는 방법에 대해 알아보자. 

운영중인 서비스 중지 없이 서비스 노드를 추가하고 **iptables** 를 이용해 로드밸런싱 규칙을 추가한다. 

<!--more-->

## 라운드로빈 

TCP 연결을 새로 맺을 때 (--state NEW) iptables의 통계를 이용하여 (-m statistic) nth 알고리즘 (--mode nth)을 이용해서 라운드로빈을 구현한다. 

```bash
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode nth --every 3 --packet 0 -j DNAT --to 127.0.0.1:8081
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode nth --every 2 --packet 0 -j DNAT --to 127.0.0.1:8082
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode nth --every 1 --packet 0 -j DNAT --to 127.0.0.1:8083
```

## 확률기반 랜덤 밸런싱

각 세션을 33% 씩 분배하는 규칙을 적용해본다. 

```bash
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode random --probability .33 -j DNAT --to 127.0.0.1:8081
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode random --probability .50 -j DNAT --to 127.0.0.1:8082
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -j DNAT --to 127.0.0.1:8083
```


## TL;DR 

VM으로 구성된 서버에서 성능을 높히고 싶을 때 CPU나 메모리 리소스만 추가해서 간단하게 스케일업을 구현할 수 있다. 


## 참고 URL

- [iptables 를 이용한 로드밸런싱 구현하기](https://winixsite.wordpress.com/2015/03/31/iptables-%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%A1%9C%EB%93%9C%EB%B0%B8%EB%9F%B0%EC%8B%B1-%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0/){:target="_blank"} 
- [Turning IPTables into a TCP load balancer for fun and profit](https://scalingo.com/blog/iptables){:target="_blank"}
- [redirect local request with NAT](https://copyprogramming.com/howto/iptables-redirect-local-request-with-nat){:target="_blank"}