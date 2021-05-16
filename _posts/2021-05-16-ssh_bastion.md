---
layout: post
title: "SSH 멀티플렉싱 기능을 이용하여 내부 서버에 SSH 접속하기"
description: "SSH 멀티플렉싱 기능으로 Bastion 호스트의 터널링을 통해 내부 서버에 안전하게 접속하는 방법을 알아본다."
img: "M_ssh_tunneling.jpg"
date: 2021-05-16 01:00:00 +0900
last_modified_at: 2021-05-16 01:00:00 +0900
tags: [ssh, bastion, multiplexing, proxycommand] # add tag
related: ssh
categories: tools
---

AWS 서비스에는 Amazon EC2 인스턴스에 대한 인바운드 SSH(Secure Shell) 액세스를 허용하기 위해 퍼블릭 서브넷의 Linux 배스천 호스트를 구성하여 프라이빗 인스턴스에 연결을 제공하는 기능을 구성할 수 있다. 

보통 회사에서 개발서버를 두고 외부에서 터미널을 접속해야 할 때 IP 공유기에 모든 서버의 설정을 하지 않고 1개의 대표 서버의 연결을 허용하고 해당 서버를 통해 내부 서버에 접속하는 방법을 쓴다.

<!--more-->

물론 VPN 서버가 구성된 경우도 있겠지만 여기서는 하나의 서버가 외부 접속이 허용되어 있는 환경에서 내부 네트워크 서버에 SSH 접속을 하기 위한 클라이언트 환경에 대해 설명한다. 

*"IP Gateway"에서 8022 포트는 "Bastion Host"의 22번 포트로 포워딩 설정이 되어 있는 환경으로 가정한다.*

![ssh bastion]({{site.baseurl}}/assets/img/ssh_bastion_diagram.jpg)

## ssh 로컬 포워딩을 이용한 접속 

SSH 프로토콜은 기본으로 터널링 기능을 제공하여 연결된 터널을 이용하여 다른 서버에 접속할 수 있는 기능을 제공하며 아래의 "-L" 파라미터로 로컬포트와 접속할 서버를 설정할 수 있다. 

```bash
$ ssh -f -N -L 50100:192.168.0.100:22 user@test.domain.com -p 8022
```

이후 "ssh user@localhost -p 50100" 명령어로 로컬 50100 포트로 접속하면 터널을 통해 192.168.0.100번 서버로 접속된다. 

*"putty"와 같은 툴을 이용하면 "SSH Local Forwarding"을 설정할 수 있다.*

## ssh 점프 호스트를 이용한 접속

아래와 같이 "점프 호스트" 를 이용하여 내부 서버에 접속할 수 있다. 

```bash
ssh -J user@test.domain.com:8022 user@192.168.0.100
```

## ssh config 설정 

ssh config 파일은 **"~/.ssh/config"** ssh로 접속시 각 연결의 특정한 정보를 설정할 수 있는 파일이며 아래와 같이 퍼블릭키를 이용한 인증에서 각 호스트별로 다른 인증서를 사용할 때 설정을 할 수 있다. 

- [퍼블릭키를 이용한 인증에서 호스트별 SSH 키 설정]({{ site.baseurl }}{% link _posts/2021-01-12-ssh_publickey.md %}){:target="_blank"}

```config
#
# GitHub
#
Host github.com
        HostName github.com
        PreferredAuthentications publickey
        IdentityFile ~/.ssh/id001_rsa
#
# bitbucket
#
Host bitbucket.org
        HostName bitbucket.org
        PreferredAuthentications publickey
        IdentityFile ~/.ssh/id002_rsa
```

## ssh ProxyCommand and Multiplexing

ssh 멀티플렉싱은 SSH 접속을 할 때 기존에 생성된 SSH 터널을 이용하여 하나의 커넥션으로 이용할 수 있어 최초 핸드쉐이킹 이후에는 생성된 터널을 이용할 수 있다. 

*아래의 설정은 [Using SSH Multiplexing](https://blog.scottlowe.org/2015/12/11/using-ssh-multiplexing/){:target="_blank"} 페이지를 참고했다.*

```config
## IWS SSH Gateway 

Host bastion
  Hostname test.domain.com
  User user
  ForwardAgent yes
  ControlPath ~/.ssh/socket-%r@%h:%p
  ControlMaster auto
  ControlPersist 10m

Host 192.168.*
  ProxyCommand ssh bastion -p 8022 -W %h:%p
```

위와 같이 "Host 192.168.*"설정을 추가하면 접속 대상 네트워크에 있는 것처럼 "192.168.0.100"으로 직접 커넥션 명령이 가능해진다.

처음 "192.168.*" 대역으로 접속시에는 ProxyCommand 에 설정된 bastion 호스트로 접속한 후 소켓 파일이 생성되고 두번째 이후 연결에서는 연결된 터널을 이용하여 해당 서버로 바로 접속된다. 

```zsh
$ ssh user@192.168.0.100
user@test.domain.com's password:
user@192.168.0.100's password:

$ ls -ltr
...
srw-------  1 butteryoon  staff     0  5 16 23:24 cm-user@test.domain.com:50322

$ file cm-user@test.domain.com:50322
cm-user@test.domain.com:50322: socket
```

## TL;DR

SSH ProxyCommand 와 멀티플렉싱을 이용하여 외부에서도 안전하게 내부 네트워크에 있는 것과 동일하게 접속할 수 있는 환경을 구성할 수 있다. 

물론 로컬 포워드 설정으로 가능하지만 원격 네트워크 환경에서 내부 네트워크와 동일한 IP로 접속을 할 수 있는 환경이 가능해지고 putty와 같은 별도의 툴을 쓰지 않아도 되어 설정이 간단해지는 장점이 있다. 


## 참고

- [AWS 기반 Linux 배스천 호스트](https://aws.amazon.com/ko/quickstart/architecture/linux-bastion/){:target="_blank"}
- [Setting Up an SSH Bastion Host](https://goteleport.com/blog/ssh-bastion-host/){:target="_blank"}
- [Using ssh bastion host](https://blog.scottlowe.org/2015/11/21/using-ssh-bastion-host/){:target="_blank"}
- [SSH Can Do That? Productivity Tips for Working with Remote Servers](http://blogs.perl.org/users/smylers/2011/08/ssh-productivity-tips.html){:target="_blank"}