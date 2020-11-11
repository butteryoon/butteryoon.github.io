---
layout: post
title: "netsh를 이용하여 WSL2 서비스 포트포워딩 설정하기"
description: "Windows 10에서 netsh 포트프록시 명령을 사용하여 포트포워딩을 설정하고 파워쉘 함수로 등록한다."
img: ms_loves_linux.png
date: 2020-10-28 11:00:00 +0900
last_modified_at: 2020-10-28 11:00:00 +0900
tags: [netsh, wsl, windows] # add tag
related: wsl
categories: tools
---

Windows 10에서 netsh 포트프록시 명령을 사용하여 WSL2 서비스 포트포워딩 설정하고 파워쉘 함수로 등록해 본다. 
<!--more-->

## WSL2 서비스 접속하기 

WSL2에는 고유한 IP 주소가 있는 가상화된 이더넷 어댑터 "vEthernet (WSL)" 가 있고 배포 리눅스에서는 "vEthernet (WSL)" 인터페이스를 게이트웨이로 설정하여 인터넷을 사용할 수 있다.  

WSL2의 서비스는 윈도우즈 호스트에서는 "vEthernet (WSL)" IP나 localhost로 서비스를 테스트 할 수 있지만 외부 접속(LAN 환경에서 시험)을 접속할 때는 포트포워딩 설정으로 WSL 내부 서비스에 접근하도록 설정해야 한다. 

아래 예제에서는 WSL의 Jekyll 서비스 포트를 포트포워딩 설정을 해보기로 한다. 

## netsh portproxy 규칙 추가

Windows 10의 네트워크쉘(netsh) 명령어를 이용하여 WSL2 호스트에 접근 가능한 포트프록시 연결을 추가한다. 

- [LAN(Local Area Network)에서 WSL 2 배포에 액세스](https://docs.microsoft.com/ko-kr/windows/wsl/compare-versions?irgwc=1&OCID=AID2000142_aff_7593_1243925&tduid=%28ir__dxkjzbsd00kfqwo9kk0sohzxzm2xshpsg6fckkpf00%29%287593%29%281243925%29%28je6NUbpObpQ-1bcWZjxnlsXtmgYWlnwQSw%29%28%29&irclickid=_dxkjzbsd00kfqwo9kk0sohzxzm2xshpsg6fckkpf00#accessing-a-wsl-2-distribution-from-your-local-area-network-lan){:target="_blank"}

```powershell
PS > netsh interface portproxy add v4tov4 listenport=4000 listenaddress=0.0.0.0 connectport=4000 connectaddress=172.20.147.206
```

netsh 포트프록시 설정 목록을 확인한다. 

```powershell
PS > netsh interface portproxy show all

Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
0.0.0.0         4000        172.20.147.206  4000
```

## PowerShell 함수로 만들기 

WSL2 서비스는 항상 운영되는게 아니기 때문에 필요할 때 명령 프롬프트에서 해당 포워딩 규칙을 추가/삭제할 수 있는 함수를 만들기로 한다. 

> localhost로 테스트는 가능하지만 "duckdns.org"와 같은 도메인 테스트가 필요한 경우에 사용한다. 

명령어 이름은 "Set-ForwardRules"로 정하고 파라미터 정의는 "-On"로 설정을 추가하고 파라미터가 없으면 해당 규칙을 삭제한다. 

{% gist butteryoon/792e38aabd2a9cfe817bd9ec4cb45377 %} 


## 참고 URL
- [Netsh 인터페이스 포트 프록시 명령](https://docs.microsoft.com/ko-kr/windows-server/networking/technologies/netsh/netsh-interface-portproxy){:target="_blank"}
- [Connecting to WSL2 server via local network](https://stackoverflow.com/questions/61002681/connecting-to-wsl2-server-via-local-network){:target="_blank"}
- 
