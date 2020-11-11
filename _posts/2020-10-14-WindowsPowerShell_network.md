---
layout: post
title: "Windows PowerShell 네트워크 상태 확인하기"
img: "powershell_title.jpg"
date: 2020-10-14 14:00:00 +0900
tags: [Windows, powershell, ps1, network] # add tag
related: powershell
categories: dev
---

Windows 10 환경에서 네트워크 테스트에 필요한 명령어를 정리한다.   
PowerShell 환경에서 아래의 "Test-NetConnection" cmdlet을 사용하여 인터넷 및 서비스 연결 확인이 가능하다. 


## 네트워크 연결 확인

"ping" 명령어와 같이 동작한다.  
파라미터 없이 "Test-NetConnection" 명령어만 사용하면 "internetbeacon.msedge.net"에 PING 하고 연결 결과를 알려준다. 

```powershell
PS C:\Users\softr> Test-NetConnection 192.168.0.120 -InformationLevel Detailed

ComputerName           : 192.168.0.120
RemoteAddress          : 192.168.0.120
NameResolutionResults  : 192.168.0.120
InterfaceAlias         : 이더넷
SourceAddress          : 192.168.0.19
NetRoute (NextHop)     : 0.0.0.0
PingSucceeded          : True
PingReplyDetails (RTT) : 0 ms
```

## 대상 Host의 특정 서비스 포트 확인. 

리눅스에에서 "nc -v -z localhost 4000" 명령어와 같이 동작하며 TCP 접속 후 종료 상태를 확인한다. 

> 간편한 방법으로 telnet 클라이언트를 이용해서 접속이 되는지 확인할 수도 있다. 

```powershell
PS D:\Dropbox\Public\butteryoon.github.io> Test-NetConnection -ComputerName 111.111.111.111 -Port 4000

ComputerName     : 111.111.111.111
RemoteAddress    : 111.111.111.111
RemotePort       : 4000
InterfaceAlias   : 이더넷
SourceAddress    : 192.168.0.19
TcpTestSucceeded : True
``` 

### Resolv-DnsName : 도메인의 IP 찾기.    

nslookup 명령어를 사용해도 된다.  
가끔 도메인이 필요할 때가 있어 duckdns.org 서비스를 이용하는데 PC IP가 제대로 업데이트 되었는지 확인 할 때 사용한다. 

```powershell
❯ Resolve-DnsName softroom.duckdns.org

Name                                           Type   TTL   Section    IPAddress
----                                           ----   ---   -------    ---------
softroom.duckdns.org                           A      60    Answer     106.xxx.xxx.xxx
```

## 참고 URL
- [1장 - PowerShell 시작](https://docs.microsoft.com/ko-kr/powershell/scripting/learn/ps101/01-getting-started?view=powershell-5.1)
- [PowerShell이란?](https://docs.microsoft.com/ko-kr/powershell/scripting/overview?view=powershell-7)
- [Windows Terminal Preview 릴리스](https://www.lesstif.com/pages/viewpage.action?pageId=71401723)
- [execution of scripts is disabled on this system.](https://www.hahwul.com/2017/08/powershell-execution-of-scripts-is.html)
- [Use Windows PowerShell to search for files](https://devblogs.microsoft.com/scripting/use-windows-powershell-to-search-for-files/)