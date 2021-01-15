---
layout: post
title: "오라클 클라우드 서비스 수신정책 추가 설정하기."
description: "오라클 클라우드 서비스 수신정책 추가 설정하기."
img: cloud-title.webp
date: 2021-01-15 11:00:00 +0900
last_modified_at: 2021-01-15 11:00:00 +0900
tags: [oracle cloud, oracle Linux, firewalld, iptables] # add tag
related: opc
categories: tools
---

오라클 클라우드 [프리티어 인스턴스](https://www.oracle.com/kr/cloud/free/#always-free)를 만들고 특정 TCP 서비스를 추가하는 방법을 알아본다. 

프리티어 인스턴스는 2개를 만들 수 있고 이미지는 선택 가능한데 기본으로 선택되어 있는 오라클리눅스 7.9 환경이다. 

> 상시무료 : 무제한으로 이용하실 수 있는 서비스입니다. (무제한이란다)

<!--more-->

## 서비스 수신 규칙 추가 

기본으로 생성된 "가상 클라우드 네트워크(vpc)"는 초기에 ssh 터미널 접속을 위한 22번 포트만 수신규칙에 추가되어 있으며 사용할 서비스의 정보를 추가해야 한다. 

> 위치 : 네트워킹 >> 가상 클라우드 네트워크 >> vcn-20210115-1556 >> 보안 목록 세부정보 >> 수신 규칙  
> ssh 접속을 위한 22번 설정이 되어 있으니 참고한다. 

![opc_add rules]({{site.baseurl}}/assets/img/opc_add_rules.png)

## 리눅스 방화벽 정책 추가 

AWS와 마찬가지로 오라클 클라우드도 규칙을 추가한 후 리눅스 인스턴스의 방화벽정책을 추가해주어야 한다. 

오라클 리눅스에는 firewalld 서비스가 기본으로 동작하므로 firewall-cmd 명령어로 규칙을 추가한다. 

```bash
[opc@instance-20210115-1556 .ssh]$ sudo firewall-cmd --get-active-zones
public
  interfaces: ens3
[opc@instance-20210115-1556 .ssh]$ sudo firewall-cmd --zone=public --add-port=8803/tcp --permanent
success
[opc@instance-20210115-1556 .ssh]$ sudo firewall-cmd --reload
success
```

정책을 추가한 후 "iptables -nL" 명령어로 추가된 규칙을 확인하면 아래와 같이 퍼블릭 체인에 서비스 추가된 것을 볼 수 있다. 

```bash 
[opc@instance-20210115-1556 ~]$ sudo iptables -nL

Chain IN_public_allow (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 ctstate NEW,UNTRACKED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8803 ctstate NEW,UNTRACKED
```


## 참고 URL
  -  [Linux Firewall: (firewalld, firewall-cmd, firewall-config)](https://oracle-base.com/articles/linux/linux-firewall-firewalld){:target="_blank"}
-  [Linux firewalls: What you need to know about iptables and firewalld](https://opensource.com/article/18/9/linux-iptables-firewalld){:target="_blank"}
