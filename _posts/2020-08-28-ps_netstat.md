---
layout: post
title: "Windwos에서 TCP 연결 정보 확인하기"
img: "M_photo-cabling.jpg"
date: 2020-08-28 15:29:00 +0900
tags: [Windows, PowerShell, comlet, netstat] # add tag
categories: tools
---

Windows10 환경에서 네트워크 관련 연결 정보를 확인하는 방법에 대해 정리해 본다. 

[PowerShell 명령어 입문 : 콘솔에서 효과적으로 Windows 제어하기](https://bit.ly/3jqLZZ9) 한번 읽어두자. 

## Remote Port 접속 정보 확인

> "Get-NetTCPConnection 
> Linux에서는 "ss -t -anpo | grep :15490" 과 같은 명령어를 사용한다.   
> ※ Format-List 옵션으로 결과를 리스트 형식으로 표시할 수 있다.  

``` 
❯ Get-NetTCPConnection | Where-Object {$_.RemotePort -eq 15490} | Format-List

LocalAddress   : 172.18.0.8
LocalPort      : 5847
RemoteAddress  : 192.168.20.11
RemotePort     : 15490
State          : Established
AppliedSetting : Internet
OwningProcess  : 21616
CreationTime   : 2020-08-28 오후 12:52:04
OffloadState   : InHost
```

## 특정 프로세스의 TCP 연결정보 표시 

### 1. Process의 PID 확인 

```
❯ Get-Process -Name liveview | select-object -ExpandProperty Id
21616
``` 

### 2. Get-NetTCPConnection의 OwingProcess 필드로 조회 
 
``` 
❯ Get-NetTCPConnection | Where-Object {$_.OwningProcess -eq 21616 }

LocalAddress                        LocalPort RemoteAddress                       RemotePort State       AppliedSetting OwningProcess
------------                        --------- -------------                       ---------- -----       -------------- -------------
0.0.0.0                             5847      0.0.0.0                             0          Bound                      21616
172.18.0.8                          5847      192.168.20.11                       15490      Established Internet       21616
```

### 3. 프로세스 이름으로 연결된 TCP 연결 정보 확인

```
❯ Get-NetTCPConnection | Where-Object {$_.OwningProcess -eq (Get-Process -Name liveview | select-object -ExpandProperty Id) }

LocalAddress                        LocalPort RemoteAddress                       RemotePort State       AppliedSetting OwningProcess
------------                        --------- -------------                       ---------- -----       -------------- -------------
0.0.0.0                             5847      0.0.0.0                             0          Bound                      21616
172.18.0.8                          5847      192.168.20.11                       15490      Established Internet       21616
```

 
## 참고 URL 

-  [파워쉘(Powershell)을 이용한 포트 정보 조회](https://coozplz.me/2017/09/08/powershell-포트-정보-조회/)