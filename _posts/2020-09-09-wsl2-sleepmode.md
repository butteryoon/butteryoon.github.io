---
layout: post
title: "Sleep 모드 이후 WSL 네트워크 안됨"
img: wsl.png
date: 2020-09-09 20:00:00 +0900
tags: [WSL, WSL2, wslconfig] # add tag
categories: dev
---

Windows 10 Sleep mode (보통은 노트북을 쓰다가 덮게를 덮으면 최대절전모드로 바뀐다) 이후 WSL2를 사용하려고 하면 네트워킹이 안되는 현상이 있다. 

microsoft/WSL github에 아래와 같은 issue가 Open 상태에 있다. 

- [[WSL2] Network connections are not restored after returning from Windows sleep mode #4992](https://github.com/microsoft/WSL/issues/4992)  
- [Doc Idea: when you need to restart LxssManager service #634](https://github.com/microsoft/WSL/issues/634)

혹시나 해서 Network 재시작을 해봤지만 안된다. 
> WSL을 중지하고 vEthernet을 재시작한다. 
> 

```
> PowerShell
wsl --shutdown Ubuntu-18.04
Restart-NetAdapter -Name vEthernet* 
``` 

## WSL 서비스 관리자 재시작 

기타 WSL2가 응답이 없고 구동이 안될 경우가 있는데 이 때 WSL 서비스관리자를 재시작하면 해결 될 수 있다. 

**그러나!! Sleep Mode 문제는 해결이 안된다.**

> 관리자: cmd 명령어  

```
sc query LxssManager
sc stop LxssManager
sc start LxssManager 
```

> PowerShell 명령어 

```
PS❯ Get-Service -Name LxssManager

Status   Name               DisplayName
------   ----               -----------
Running  LxssManager        LxssManager

PS❯ Restart-Service -Name LxssManager -Confirm

확인
이 작업을 수행하시겠습니까?
대상 "LxssManager (LxssManager)"에서 "Restart-Service" 작업을 수행합니다.
[Y] 예(Y)  [A] 모두 예(A)  [N] 아니요(N)  [L] 모두 아니요(L)  [S] 일시 중단(S)  [?] 도움말 (기본값은 "Y"):
```


## Page Link 
- [WSL2 정식 버전 사용하기](https://www.lesstif.com/software-architect/wsl-2-windows-subsystem-for-linux-2-89555812.html)  
- [WSL 2(Windows Subsystem For Linux 2) Preview 버전 사용하기](https://www.lesstif.com/software-architect/wsl-2-windows-subsystem-for-linux-2-preview-71401661.html)   
- [WSL2 시작하기]({{site.baseurl}} {% link _posts/2020-03-03-wsl2.md %}) 
