---
layout: post
title: "SSH 리버스 터널링을 이용한 방화벽 서버 접속"
description: "Firewall 아래 있는 HOST를 외부에서 접속 할 수 없는 경우 SSH Reverse Tunneling을 이용하여 연결을 가능하게 하는 방법을 알아본다."
img: "M_ssh_tunneling.jpg"
date: 2020-09-02 23:00:00 +0900
last_modified_at: 2021-05-30 15:00:00 +0900
tags: [ssh, tunnel, reverse] # add tag
related: ssh
categories: tools
---

Firewall 아래 있는 HOST:A 를 외부에서 접속 할 수 없는 경우 *SSH Reverse Tunneling* 기능을 이용하여 연결을 가능하게 하는 방법을 알아본다.

다만, ID/PW 인증을 하는 경우에는 HOST:A에서 HOST:B로 연결을 먼저 해야 하고 공개키 인증을 설정해 놓은 경우에는 자동 연결도 가능하다. 

<!--more-->

아래와 같은 단순화 한 구성이 있다.  

HOST:A(SSHD:22) <--> Firewall <--> Internet <--> HOST:B(SSHD:53022)

![diagrams]({{site.baseurl}}/assets/img/ssh_reverse_01.jpg){:width="700px"}

1. HOST:A 는 HOST:B에 ssh 연결이 가능해야 한다. 
2. Firewall 에는 HOST:B로의 접속이 허용된다. 
3. HOST:A 에는 SSHD 서버는 로컬호스트에서 접속이 가능해야 한다. 
4. HOST:B 에서는 ssh 클라이언트를 쓸 수 있다. 

## Reverse Tunneling  

아래 HOST:A는 Windows 10 환경에 SSHD 서버가 설정되어 있고 HOST:B는 레드햇 리눅스 환경이다. HOST:A 에서 HOST:B로 ssh 연결하고 사용자는 HOST:B에서 연결된 SSH 터널을 이용해서 HOST:A 에 접속할 수 있다. 

### 1. HOST:A 에서 HOST:B로 SSH 연결

man ssh 에서 "-R [bind_address:]port:host:hostport"를 보면 아래와 같이 -R 옵션으로 리모트 서버에 소켓을 열 수 있고 서버에서는 설정된 포트를 이용해서 연결된 SSH 채널로 클라이언트 호스트로 접속할 수 있다. 

```bash
-R [bind_address:]port:host:hostport
-R [bind_address:]port:local_socket
-R remote_socket:host:hostport
-R remote_socket:local_socket
    Specifies that connections to the given TCP port or Unix socket on the remote (server) host are to be forwarded to the given host and port, or Unix socket, on the local side.  This works by allocating a socket to listen to either a TCP port or to a Unix socket on the remote side.  Whenever a connection is made to this port or Unix socket, the connection is forwarded over the secure channel, and a connection is made to either host port hostport, or local_socket, from the local machine.
```

아래와 같이 HOST:A 에서 HOST:B로 SSH 연결을 하면 HOST:B에 43022 포트가 열리고 43022 포트로 연결을 하면 HOST:A의 localhost의 22번 포트로 연결된다. 

```bash
HOST:A $ ssh -R 43022:localhost:22 id@HOST:B -p 53022
```

ssh 연결 후 명령 실행이 필요 없을 경우 **-f -N -T** 명령어를 사용하면 실행 후 바로 백그라운드로 동작한다. 

```bash
HOST:A $ ssh -f -N -T -R 43022:localhost:22 id@HOST:B -p 53022
```

### 1-1. HOST:B 에서 43022 LISTEN 포트 확인 

연결된 HOST:B에서 ss 명령어르 이용해 LISTEN 포트를 확인하면 아래와 같이 43022번 포트가 확인되어야 한다. 

```bash
HOST:B $ ss -t -anpo | grep :43022

LISTEN 0 128   127.0.0.1:43022   *:*     
LISTEN 0 128   ::1:43022         :::*     
```

### 2. HOST:B 에서 HOST:A로 SSH 연결 

HOST:B에서 ssh 명령어와 -p 옵션을 사용하여 로컬호스트 43022번으로 접속한다.  

```bash
HOST:B $ ssh HOST:A-ID@localhost -p 43022
softroom@localhost's password: input password 
``` 

sftp 명령어와 -oPort 옵션을 사용하거나 파일질라 등을 사용하여 파일전송도 가능하다. 

```bash
HOST:B $ sftp -oPort=43022 HOST:A-ID@localhost 
softroom@localhost's password: input password 
sftp>
``` 

### 2-1. HOST:A 에서 22번 포트 연결 상태 확인  

HOST:A 에서 PowerShell 명령어로 22번으로 연결된 Established 상태의 세션을 보면 **::1** 과 같이 loopback에서 연결된 걸로 표시된다.

> SSH 채널을 이용하지 않고 직접 연결이 되면 RemoteAddress에 실제 접속한 IP가 표시된다.  

```powershell
❯ Get-NetTCPConnection | Where-Object {$_.LocalPort -eq 22} 

LocalAddress   LocalPort RemoteAddress  RemotePort State       AppliedSetting OwningProcess
------------   --------- -------------  ---------- -----       -------------- -------------
::1            22        ::1            12699      Established Internet       5584
::             22        ::             0          Listen                     5584
0.0.0.0        22        0.0.0.0        0          Listen                     5584
```

## TL;DR 

내부서버로의 연결이 방화벽으로 막혀 있을 경우 내부서버에서 외부 SSH 터널을 연결하고 외부 서버에서 접속을 요청한 서버로 접속 할 수 있는 방법이다.  

보안상 이슈가 있을 수 있어 필요시에만 쓰는걸로 .. 


## 참고

- [SSH port forwarding - Example](https://www.ssh.com/academy/ssh/tunneling/example){:target="_blank"}
- [What Is Reverse SSH Tunneling?](https://bit.ly/2Dm4ONU){:target="_blank"}
- [What is Reverse SSH Port Forwarding](https://blog.devolutions.net/2017/3/what-is-reverse-ssh-port-forwarding){:target="_blank"}
- [What is IP Address '::1'](https://stackoverflow.com/questions/4611418/what-is-ip-address-1){:target="_blank"}
- [Port forwarding with SSH](https://rufflewind.com/2014-03-02/ssh-port-forwarding){:target="_blank"}

