---
layout: post
title: "Windows PowerShell"
img: "M_PowerSehll.jpg"
date: 2020-07-16 13:29:00 +0900
tags: [Windows, PowerShell, comlet] # add tag
categories: dev
---

## Windows PowerShell Command

Windows의 GUI 환경은 좋지만 가끔은 마우스로 원하는 기능을 찾기가 힘들 때가 있다. 
윈도우즈10 환경에서 간단한 명령어로 조금 편하게 쓸 수 있는 기능들을 정리해 본다. 

[What is PowerShell](https://docs.microsoft.com/ko-kr/powershell/scripting/overview?view=powershell-7) 한번 읽어두자. 

## 현재 사용하고 있는 터미널 레이아웃.  

> PowerShell, cmd, wsl2 터미널 이렇게 세개의 창을 배치. 

![Terminal Layout]({{site.bashurl}}/assets/img/Terminal_Layout.png) 

## 기본 환경 변수

```
⚡ softroom@YOON-IP700  C:\>
❯ $PROFILE | select *

AllUsersAllHosts       : C:\Windows\System32\WindowsPowerShell\v1.0\profile.ps1
AllUsersCurrentHost    : C:\Windows\System32\WindowsPowerShell\v1.0\Microsoft.PowerShell_profile.ps1
CurrentUserAllHosts    : D:\App\Dropbox\WindowsPowerShell\profile.ps1
CurrentUserCurrentHost : D:\App\Dropbox\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
Length                 : 65
```
 
## 도움말 

> "Get-Help -Name 검색어" 
> PowerShell의 명령어는 익숙하지가 않아 무조건 찾아본다. 

``` 
❯ Get-Help -Name Uninstall

Name                              Category  Module                    Synopsis
----                              --------  ------                    --------
Uninstall-Package                 Cmdlet    PackageManagement         Uninstall-Package...
```

## alias 

> Get-Alias  


## 주로 쓰는 명령어 정리 

> Get-Location : 현재 디렉토리 위치이며 보통 스크립트에서 현재 디렉토리 확인시 사용한다. 

```
❯ Get-Location :

Path
----
D:\App\Dropbox\Public\butteryoon.github.io\_posts
```

> Uninstall-Package : 설치된 프로그램 삭제 
> Get-Package -Name Ahn* 명령어로 프로그램 이름을 찾은 후 삭제한다.

```
❯ Uninstall-Package -Name ipMonitor10
경고: Reboot is required to complete the uninstallation
operation.

Name                           Version          Source
----                           -------          ------
ipMonitor10                    10.9.1
```

> Stop-Process -Name "프로그램 이름"
> 파일의 내용이나 정보 확인. (cmd 에서는 type이었던거 같은데)

```
❯ Get-Content .\.wakatime.cfg
[settings]
debug = false
hidefilenames = false
ignore =
    COMMIT_EDITMSG$
    PULLREQ_EDITMSG$
    MERGE_MSG$
    TAG_EDITMSG$
    api_key=0e64xxxx-xxxx-xxxx-xxxx-xxxx2865xxxx
```

> Get-Process : 특정 포트를 사용하는 프로세스 찾기. 

```
❯ Get-Process -Id (Get-NetTCPConnection -LocalPort 58803).OwningProcess

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    247      22     9216      20540       1.45  11060   1 python
```

> 실행중인 프로세스 파일버전 표시. 

```
❯ Get-Process liveview -FileVersionInfo

ProductVersion   FileVersion      FileName
--------------   -----------      --------
1.0.0.1          1.0.0.1          C:\IIOT-LIVEVIEW\LIVEVIEW.exe
```

> Resolv-DnsName : 도메인의 IP 찾기.   
> nslookup 명령어를 사용해도 된다. 
> 가끔 도메인이 필요할 때가 있어 duckdns.org 서비스를 이용하는데 내 랩탑의 IP가 제대로 업데이트 되었는지 확인 할 때 사용한다. 

```
❯ Resolve-DnsName softroom.duckdns.org

Name                                           Type   TTL   Section    IPAddress
----                                           ----   ---   -------    ---------
softroom.duckdns.org                           A      60    Answer     106.xxx.xxx.xxx
```

## 참고 URL
-  [](https://www.lesstif.com/pages/viewpage.action?pageId=71401723)
-  [execution of scripts is disabled on this system.](https://www.hahwul.com/2017/08/powershell-execution-of-scripts-is.html)