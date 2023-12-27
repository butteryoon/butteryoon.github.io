---
layout: post
comments: true
title: "프로세스 로드밸런싱"
description: "iptables로 서버 내 N개의 서비스 트래픽 분배하기"
img: command-title.webp
date: 2023-12-23 00:50:00 +0900
last_modified_at: 2023-12-27 21:50:00 +0900
tags: [command-line, iptables , masquerade, nat, port forward, 라우드로빈, 로드밸런싱] # add tag
related: command-line
categories: tools
---

한 대의 서버에 같은 서비스를 다른 포트로 여러개를 실행하고 요청을 분배하는 방법에 대해 알아보자. 

운영중인 서비스 중지 없이 서비스 노드를 추가하고 **iptables** 를 이용해 로드밸런싱 규칙을 추가한다. 

<!--more-->

## lo 인터페이스

서비스 포트는 8081 하나로 구성되는 환경이 외부에 노출된 API 서버 인터페이스로 서비스 요청이 되면 로컬 서버 포트로 전달하여 서비스하는 환경에서 적용된다.  

일반적으로 로컬에서 자기 자신의 IP로 접근하는 경우 라우팅 테이블을 거치지 않고 로컬 루프백(lo) 인터페이스로 바로 전달되기 때문에 아래와 같이 로컬 포워딩 설정을 한다. 

```bash
❯ cat /etc/sysctl.conf

# Enable IP Forwarding
net.ipv4.ip_forward=1
net.ipv4.conf.all.route_localnet=1

❯ sudo sysctl -p
net.ipv4.ip_forward = 1
net.ipv4.conf.all.route_localnet = 1
```

아래 사이트에 iptables 패킷 플로우를 참고한다. 
> [iptables packet flow](https://rakhesh.com/linux-bsd/iptables-packet-flow-and-various-others-bits-and-bobs/){:target="_blank"}  


## 라운드로빈 

TCP 연결을 새로 맺을 때 (--state NEW) iptables의 통계를 이용하여 (-m statistic) nth 알고리즘 (--mode nth)을 이용해서 라운드로빈을 구현한다. 

> --every 3, --every 2, --every 1 규칙을 적용했을 때 세 개의 룰에 순차적으로 적용된다.   


```bash
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode nth --every 3 --packet 0 -j DNAT --to 127.0.0.1:8081
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode nth --every 2 --packet 0 -j DNAT --to 127.0.0.1:8082
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode nth --every 1 --packet 0 -j DNAT --to 127.0.0.1:8083
```

## 확률기반 랜덤 밸런싱

각 세션을 33% 씩 분배하는 규칙을 적용해본다. 

처음 33%의 세션은 첫번째 규칙에 적용되고 패스된 66%의 세션 중 50%를 두번째 규칙에서 적용하고 나머지 세션은 마지막 규칙에 적용되어 33%씩 분배된다. 

```bash
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode random --probability .33 -j DNAT --to 127.0.0.1:8081
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -m statistic --mode random --probability .50 -j DNAT --to 127.0.0.1:8082
sudo iptables -t nat -A OUTPUT -p tcp --dport 8081 -m state --state NEW -j DNAT --to 127.0.0.1:8083
```

## 테스트 

iptables 로드밸런싱 규칙을 적용한 후 8081 포트로 요청을 해보면 아래와 같이 순차적으로 OUTPUT 규칙에 적용된 패킷 수를 확인할 수 있다. 

```bash
Every 1.0s: sudo iptables -t nat -nL -v                                LAPTOP-JSYOON: Wed Dec 27 22:40:13 2023
...
Chain OUTPUT (policy ACCEPT 4 packets, 240 bytes)
 pkts bytes target     prot opt in     out     source               destination
   14   840 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8081 state NE
W statistic mode nth every 3 to:127.0.0.1:8081
   14   840 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8081 state NE
W statistic mode nth every 2 to:127.0.0.1:8082
   13   780 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:8081 state NE
W statistic mode nth every 1 to:127.0.0.1:8083
...
```

## TL;DR 

VM으로 구성된 서버에서 성능을 높히고 싶을 때 CPU나 메모리 리소스만 추가한 후 필요한 모듈을 추가로 구동하고 로드밸런싱 규칙을 추가하여 기존 서비스에 영향을 주지 않고 스케일업을 구현할 수 있다. 


## 참고 URL

- [iptables 를 이용한 로드밸런싱 구현하기](https://winixsite.wordpress.com/2015/03/31/iptables-%EB%A5%BC-%EC%9D%B4%EC%9A%A9%ED%95%9C-%EB%A1%9C%EB%93%9C%EB%B0%B8%EB%9F%B0%EC%8B%B1-%EA%B5%AC%ED%98%84%ED%95%98%EA%B8%B0/){:target="_blank"} 
- [Turning IPTables into a TCP load balancer for fun and profit](https://scalingo.com/blog/iptables){:target="_blank"}
- [redirect local request with NAT](https://copyprogramming.com/howto/iptables-redirect-local-request-with-nat){:target="_blank"}
- [IP-tables based Load Balancing](https://loadbalancerweb.wordpress.com/2016/07/03/ip-tables-based-load-balnacing/){:target="_blank"}