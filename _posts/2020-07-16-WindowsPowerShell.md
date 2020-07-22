---
layout: post
title: "Windows PowerShell"
img: PowerSehll.png
date: 2020-07-16 13:29:00 +0900
tags: [Windows PowerShell] # add tag
categories: dev
---

## Windows PowerShell Command

Windows의 GUI 환경은 좋지만 가끔은 마우스로 원하는 기능을 찾기가 힘들 때가 있다. 
윈도우즈10 환경에서 간단한 명령어로 조금 편하게 쓸 수 있는 기능들을 정리해 본다. 

[What is PowerShell](https://docs.microsoft.com/ko-kr/powershell/scripting/overview?view=powershell-7) 한번 읽어두자. 

## 현재 사용하고 있는 터미널 레이아웃.  

> PowerShell, cmd, wsl2 터미널 이렇게 세개의 창을 배치. 

![Terminal Layout]({{site.bashurl}}/assets/img/Terminal_Layout.png)
 
## 도움말 

> "Get-Help -Name 검색어" 
> PowerShell의 명령어는 익숙하지가 않아 무조건 찾아본다. 

``` 
❯ Get-Help -Name Uninstall

Name                              Category  Module                    Synopsis
----                              --------  ------                    --------
Uninstall-Package                 Cmdlet    PackageManagement         Uninstall-Package...
Uninstall-Module                  Function  PowerShellGet             ...
Uninstall-Script                  Function  PowerShellGet             ...
Start-OSUninstall                 Cmdlet    Dism                      Start-OSUninstall...
Uninstall-Dtc                     Function  MsDtc                     ...
Uninstall-ProvisioningPackage     Cmdlet    Provisioning              Uninstall-ProvisioningPackag...
Uninstall-TrustedProvisioningC... Cmdlet    Provisioning              Uninstall-TrustedProvisionin...
```

## 주로 쓰는 명령어 정리 

> Get-Location : 현재 위치 표시

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

```
❯ Resolve-DnsName softroom.duckdns.org

Name                                           Type   TTL   Section    IPAddress
----                                           ----   ---   -------    ---------
softroom.duckdns.org                           A      60    Answer     106.xxx.xxx.xxx

Name      : duckdns.org
QueryType : NS
TTL       : 572
Section   : Authority
NameHost  : ns1.duckdns.org


Name      : duckdns.org
QueryType : NS
TTL       : 572
Section   : Authority
NameHost  : ns3.duckdns.org


Name      : duckdns.org
QueryType : NS
TTL       : 572
Section   : Authority
NameHost  : ns2.duckdns.org

ns1.duckdns.org                                A      43093 Additional 54.187.92.222
ns2.duckdns.org                                A      43093 Additional 54.191.117.119
ns3.duckdns.org                                A      43093 Additional 52.26.169.94
```

## 참고 URL
-  [Windows Terminal Preview 릴리즈](https://www.lesstif.com/pages/viewpage.action?pageId=71401723)
-  [execution of scripts is disabled on this system.](https://www.hahwul.com/2017/08/powershell-execution-of-scripts-is.html)