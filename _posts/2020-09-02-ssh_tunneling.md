---
layout: post
title: "SSH 터널링을 이용한 Reverse 접속"
img: "M_ssh_tunneling.jpg"
date: 2020-09-02 23:00:00 +0900
tags: [ssh, tunnel, remote] # add tag
categories: tools
---

아래와 같은 단순화 한 구성이 있다.  

Firewall 아래 있는 HOST:A에는 외부에서 접속 할 수 없는 경우 임시로 연결을 가능하게 하는 방법을 알아본다. 

다만, ID/PW 인증을 하는 경우에는 HOST:A에서 HOST:B로 연결을 먼저 해야 하며 공개키 인증을 설정해 놓은 경우에는 자동 연결도 가능하다. 

```bash
HOST:A(SSHD:22) <--> Firewall <--> Internet <--> HOST:B(SSHD:53022)
```

1. HOST:A 에서는 HOST:B에 ssh 연결이 가능해야 한다. 
2. 물론 Firewall 에는 HOST:B로의 접속이 허용된다. 
3. HOST:A 에는 SSHD 서버가 기동되어 있다. 

## Reverse Tunneling  

### 1. HOST:A 에서 HOST:B로 SSH 연결

> $ man ssh의 "-R [bind_address:]port:host:hostport" 설명을 읽어두자. 

```bash
HOST:A $ ssh -R 43022:localhost:22 id@HOST:B -p 53022
```

### 1-1. HOST:B 에서 43022 LISTEN 포트 확인 

```bash
HOST:B $ ss -t -anpo | grep :43022
LISTEN 0 128   127.0.0.1:43022   *:*     
LISTEN 0 128   ::1:43022         :::*     
```

### 2. HOST:B 에서 HOST:A로 SSH 연결 

```bash
HOST:B $ ssh HOST:A-ID@localhost -p 43022
softroom@localhost's password: input password 
``` 

### 2-1. HOST:A 에서 22번 포트 연결 상태 확인  

> 22번으로 엱결된 Connection을 보면 아래와 같이 RemoteAddress 


```powershell
❯ Get-NetTCPConnection | Where-Object {$_.LocalPort -eq 22} 

LocalAddress   LocalPort RemoteAddress  RemotePort State       AppliedSetting OwningProcess
------------   --------- -------------  ---------- -----       -------------- -------------
::1            22        ::1            12699      Established Internet       5584
::             22        ::             0          Listen                     5584
0.0.0.0        22        0.0.0.0        0          Listen                     5584
```

### 참고 URL
-  [What Is Reverse SSH Tunneling?](https://bit.ly/2Dm4ONU)
-  [What is Reverse SSH Port Forwarding](https://blog.devolutions.net/2017/3/what-is-reverse-ssh-port-forwarding)