---
layout: post
title: "오라클 클라우드 프리티어 시작하기."
description: "오라클 클라우드 프리티어 인스턴스를 만들고 기본 서비스 설정한다."
img: cloud-title.webp
date: 2021-01-18 19:00:00 +0900
last_modified_at: 2021-01-18 19:00:00 +0900
tags: [oci, oracle cloud, oracle Linux, firewalld, iptables] # add tag
related: oci
categories: tools
---

오라클 클라우드 [상시 무료 클라우드](https://www.oracle.com/kr/cloud/free/#always-free) 서비스를 이용하여 무료로 쓸 수 있는 서버를 만들고 [duckdns](https://www.duckdns.org) 서비스를 이용해 도메인으로 접속해본다. 

> 상시무료 : 무제한으로 이용하실 수 있는 서비스입니다. ( 제한된 범위 내에서 무제한 입니다. )

프리티어 소개 페이지에는 AWS 클라우드 프리티어과 비교하여 오라클 클라우드의 이점이 잘 나와 있다. 

> 개인용 웹서버나 개발서버로 쓰기에는 부족함이 없어 보인다.

<!--more-->

## 인스턴스 생성. 

프리티어 인스턴스는 2개를 만들 수 있고 이미지는 선택 가능한데 기본으로 오라클리눅스 7.9가 제공된다.

두개의 인스턴스를 만들 수 있으니 1개는 기본 오라클 리눅스르 설정하고 다른 한개는 우분투 20.04 LTS를 설치하기로 한다. 

오라클 클라우드 인스턴스에 접속할 수 있는 공인IP가 기본으로 제공되는데 인스턴스를 재부팅해도 유지된다. 


## node.js 설치 

간단한 API서버를 위해 우분투 인스턴스에 node.js를 설치하기로 하고 패키지 관리자는 snap을 써보기로 한다.

```bash
ubuntu@instance-20210116-0003:~$ sudo snap install node --classic --channel=14
Download snap "core" (10583) from channel "stable" ( ... skip )
... (skip)
node (14/stable) 14.15.4 from OpenJS Foundation (iojs✓) installed
```

## 서비스 "수신 규칙" 추가 

기본으로 생성된 "가상 클라우드 네트워크(VCN)"는 초기에 ssh 터미널 접속을 위한 22번 포트만 수신규칙에 추가되어 있으며 사용할 서비스의 정보를 추가해야 한다. 

인스턴스를 생성할 때 VCN을 같게 설정하면 동일한 네트워크에서 인스턴스간 통신을 허용할 수 있고 동일한 수신정책을 적용할 수 있다. 

> 위치 : 네트워킹 >> 가상 클라우드 네트워크 >> vcn-20210115-1556 >> 보안 목록 세부정보 >> 수신 규칙  
> ssh 접속을 위한 22번 설정이 되어 있으니 참고한다. 

![opc_add rules]({{site.baseurl}}/assets/img/opc_add_rules.png)

## 오라클 리눅스 방화벽 정책 추가 

AWS와 마찬가지로 오라클 클라우드도 "수신규칙"을 추가한 후 리눅스 인스턴스의 방화벽정책을 추가해주어야 한다. 

오라클 리눅스 인스턴스에는 firewalld 서비스가 기본으로 동작하므로 firewall-cmd 명령어로 규칙을 추가한다. 

```bash
[opc@instance-20210115-1556 .ssh]$ sudo firewall-cmd --get-active-zones
public
  interfaces: ens3
[opc@instance-20210115-1556 .ssh]$ sudo firewall-cmd --zone=public --add-port=3000/tcp --permanent
success
[opc@instance-20210115-1556 .ssh]$ sudo firewall-cmd --reload
success
```

정책을 추가한 후 "iptables --list" 명령어로 추가된 규칙을 확인하면 아래와 같이 퍼블릭 체인에 서비스 추가된 것을 볼 수 있다. 

```bash 
[opc@instance-20210115-1556 ~]$ sudo iptables --list

Chain IN_public_allow (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:22 ctstate NEW,UNTRACKED
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:3000 ctstate NEW,UNTRACKED
```

## 우분투 리눅스 방화벽 정책 추가 

우분투 리눅스의 경우에는 iptables 명령어로 직접 추가하거나 rules 저장파일을 편집하여 포트를 추가한다. 

아래와 같이 "--line-numbers" 옵션으로 정책의 번호를 확인하고 "-I INPUT 6" 옵션으로 6번째 정책으로 추가한다.  

```bash
ubuntu@instance-20210116-0003:~$ sudo iptables --list INPUT --line-numbers
Chain INPUT (policy ACCEPT)
num  target     prot opt source               destination
1    ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
2    ACCEPT     icmp --  anywhere             anywhere
3    ACCEPT     all  --  anywhere             anywhere
4    ACCEPT     udp  --  anywhere             anywhere             udp spt:ntp
5    ACCEPT     tcp  --  anywhere             anywhere             state NEW tcp dpt:ssh
6    REJECT     all  --  anywhere             anywhere             reject-with icmp-host-prohibited

ubuntu@instance-20210116-0003:~$ sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 3000 -j ACCEPT
```

아래와 같이 "rules.v4" 파일을 보면 ssh연결을 위한 22번 포트 허용 정책이 있고 아래와 같이 서비스 포트를 추가한 후 iptables를 재시작 하여 변경된 규칙을 적용할 수도 있다. 

```bash
$ sudo vi /etc/iptables/rules.v4

... (skip)
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
-A INPUT -p tcp -m state --state NEW -m tcp --dport 3000 -j ACCEPT
... (skip)

$ sudo service iptables restart
```

## duckdns.org 서비스 

무료 도메인이 필요한 경우에는 "duckdns.org"서비스에서 5개 까지 도메인을 만들 수 있고 [Install](https://www.duckdns.org/install.jsp)메뉴에 보면 각 OS에서 사용할 수 있는 툴 및 스크립트를 볼 수 있으며 오라클 인스턴스에서는 cron을 이용할 수 있다. 

아래의 내용을 쉘 스크립트로 만들고 약 5분 주기로 설정한다. 

```bash
echo url="https://www.duckdns.org/update?domains=exampledomain&token=a7c4d0ad-ba1d-d217904a50f2&ip=" | curl -k -o ~/duckdns/duck.log -K -
```

duckdns.org 페이지에 로그인해 보면 등록한 도메인 현황을 볼 수 있다. 


위 설정으로 간단한 웹 서비스를 구동하고 인터넷에서 자신의 도메인으로 서비스 접속이 가능하며 [Letsencrypt](https://letsencrypt.org) 서비스로 인증서를 받아 https 서비스도 가능하다. 



## 참고 URL
  -  [Linux Firewall: (firewalld, firewall-cmd, firewall-config)](https://oracle-base.com/articles/linux/linux-firewall-firewalld){:target="_blank"}
-  [Linux firewalls: What you need to know about iptables and firewalld](https://opensource.com/article/18/9/linux-iptables-firewalld){:target="_blank"}