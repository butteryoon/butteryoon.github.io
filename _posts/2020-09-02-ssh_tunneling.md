---
layout: post
title: "SSH 리버스 터널링을 이용한 방화벽 서버 접속"
img: "M_ssh_tunneling.jpg"
date: 2020-09-02 23:00:00 +0900
tags: [ssh, tunnel, remote] # add tag
related: ssh
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

man ssh의 "-R [bind_address:]port:host:hostport" 설명을 읽어두자. 

```bash
HOST:A $ ssh -R 43022:localhost:22 id@HOST:B -p 53022
```

ssh 연결 후 명령 실행이 필요 없을 경우 "-f -n -T" 명령어를 사용하면 실행 후 바로 백그라운드로 동작한다. 

```bash
HOST:A $ ssh -f -N -T -R 43022:localhost:22 id@HOST:B -p 53022
```

### 1-1. HOST:B 에서 43022 LISTEN 포트 확인 

연결된 호스트:B에서 ss 명령어르 이용해 LISTEN 포트를 확인한다. 

```bash
HOST:B $ ss -t -anpo | grep :43022
LISTEN 0 128   127.0.0.1:43022   *:*     
LISTEN 0 128   ::1:43022         :::*     
```

### 2. HOST:B 에서 HOST:A로 SSH 연결 

ssh 명령어와 -p 옵션을 사용하여 로컬호스트 43022번으로 접속한다.  

```bash
HOST:B $ ssh HOST:A-ID@localhost -p 43022
softroom@localhost's password: input password 
``` 

sftp 명령어와 -oPort 옵션을 사용하여 로컬호스트 43022번으로 접속한다. 

```bash
HOST:B $ sftp -oPort=43022 HOST:A-ID@localhost 
softroom@localhost's password: input password 
sftp>
``` 

### 2-1. HOST:A 에서 22번 포트 연결 상태 확인  

> 22번으로 연결된 Established 상태의 세션을 보면 ::1 (loopback) 주소로 표시된다.  
> 다른 Address에서 연결이 되면 RemoteAddress에 실제 접속한 IP가 표시된다.  

```powershell
❯ Get-NetTCPConnection | Where-Object {$_.LocalPort -eq 22} 

LocalAddress   LocalPort RemoteAddress  RemotePort State       AppliedSetting OwningProcess
------------   --------- -------------  ---------- -----       -------------- -------------
::1            22        ::1            12699      Established Internet       5584
::             22        ::             0          Listen                     5584
0.0.0.0        22        0.0.0.0        0          Listen                     5584
```

방화벽으로 내부 연결이 막혀 있을 경우 접속한 터널링 연결을 이용해서 접속을 요청한 클라이언트에 접속 할 수 있는 방법이다. 

### 참고 URL
- [What Is Reverse SSH Tunneling?](https://bit.ly/2Dm4ONU)
- [What is Reverse SSH Port Forwarding](https://blog.devolutions.net/2017/3/what-is-reverse-ssh-port-forwarding)
- [What is IP Address '::1'](https://stackoverflow.com/questions/4611418/what-is-ip-address-1)
- [Port forwarding with SSH](https://rufflewind.com/2014-03-02/ssh-port-forwarding)