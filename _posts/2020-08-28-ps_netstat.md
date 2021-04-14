---
layout: post
title: "Windwos에서 TCP 연결 정보 확인하기"
description: "Windwos PowerSheel에서 TCP 연결 정보 및 관련 프로세스를 확인하는 방법을 알아본다."
img: "M_photo-cabling.jpg"
date: 2020-08-28 15:29:00 +0900
last_modified_at: 2020-10-28 19:00:00 +0900
tags: [Windows10, powershell, comlet, netstat] # add tag
related: Windows10
categories: tools
---

Windows10 환경에서 특정 프로세스가 사용하는 네트워크 연결정보를 확인하고 해당 포트를 사용하는 프로그램을 확인하는 방법을 알아본다. 

그리고 PowerShell 환경에서 유용하게 사용할 수 있도록 함수로 만들어 본다. 

[PowerShell 명령어 입문 : 콘솔에서 효과적으로 Windows 제어하기](https://bit.ly/3jqLZZ9) 한번 읽어두자. 

## Remote Port 접속 정보 확인

프로세스가 접속하는 포트를 알고 있는 경우 RemotePort 정보로 조회하여 세부 정보를 가져올 수 있다.

정보를 보면 Local, Remote 주소와 포트 정보 그리고 프로세스의 PID를 알 수 있다. 

> "Get-NetTCPConnection 
> Linux에서는 "ss -t -anpo | grep :15490" 과 같은 명령어를 사용한다.   
> ※ Format-List 옵션으로 결과를 리스트 형식으로 표시할 수 있다.  

```powershell 
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

프로그램이 사용하는 포트 정보를 모른다면 "Get-Process" 명령어를 이용하여 프로그램 이름으로 프로세스의 PID(OwnongProcess)를 얻어 올 수 있다. 

## 1. Process의 PID 확인 

```powershell
❯ Get-Process -Name liveview | select-object -ExpandProperty Id
21616
``` 

## 2. Get-NetTCPConnection의 OwingProcess 필드로 조회 

프로세스의 PID를 얻어온 후 "Get-NetTCPConnection" 명령어를 이용해서 네트워크 연결 상태를 얻어 올 수 있다. 

```powershell
❯ Get-NetTCPConnection | Where-Object {$_.OwningProcess -eq 21616 }

LocalAddress                        LocalPort RemoteAddress                       RemotePort State       AppliedSetting OwningProcess
------------                        --------- -------------                       ---------- -----       -------------- -------------
0.0.0.0                             5847      0.0.0.0                             0          Bound                      21616
172.18.0.8                          5847      192.168.20.11                       15490      Established Internet       21616
```

## 3. 프로세스 이름으로 연결된 TCP 연결 정보 확인

위의 두 가지 명려어를 파이프로 연결해서 프로세스 이름으로 네트워크 연결 정보를 확인 할 수 있다. 

```powershell
❯ Get-NetTCPConnection | Where-Object {$_.OwningProcess -eq (Get-Process -Name liveview | select-object -ExpandProperty Id) }

LocalAddress                        LocalPort RemoteAddress                       RemotePort State       AppliedSetting OwningProcess
------------                        --------- -------------                       ---------- -----       -------------- -------------
0.0.0.0                             5847      0.0.0.0                             0          Bound                      21616
172.18.0.8                          5847      192.168.20.11                       15490      Established Internet       21616
```

## 4. 파워쉘 함수로 등록

PowerShell 프로파일에 아래와 같이 Function으로 등록해 놓으면 "Get-Status 프로세스이름" 명령으로 사용할 수 있다. 

{% gist butteryoon/bfc127f9491a11f59248c4a9afcf99b0 %}
 
그럼 DropBox는 어떤 상태인지 조회해 본다. 

```powershell
PS > Get-Status DropBox

LocalAddress         LocalPort RemoteAddress     RemotePort State       AppliedSetting OwningProcess
------------         --------- -------------     ---------- -----       -------------- -------------
::                   17500     ::                0          Listen                     11372
0.0.0.0              9074      0.0.0.0           0          Bound                      11372
0.0.0.0              4257      0.0.0.0           0          Bound                      11372
0.0.0.0              4225      0.0.0.0           0          Bound                      11372
0.0.0.0              4222      0.0.0.0           0          Bound                      11372
0.0.0.0              4196      0.0.0.0           0          Bound                      11372
127.0.0.1            17600     0.0.0.0           0          Listen                     11372
0.0.0.0              17500     0.0.0.0           0          Listen                     11372
127.0.0.1            9074      127.0.0.1         9073       Established Internet       11372
127.0.0.1            9073      127.0.0.1         9074       Established Internet       11372
111.111.111.111      4257      162.125.35.136    443        Established Internet       11372
127.0.0.1            4225      127.0.0.1         4224       Established Internet       11372
127.0.0.1            4224      127.0.0.1         4225       Established Internet       11372
111.111.111.111      4222      162.125.35.134    443        Established Internet       11372
127.0.0.1            843       0.0.0.0           0          Listen                     11372
```

## 참고 URL 

- [파워쉘(Powershell)을 이용한 포트 정보 조회](https://coozplz.me/2017/09/08/powershell-포트-정보-조회/)