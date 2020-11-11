---
layout: post
title: "Windows PowerShell 기본 명령어 익히기"
img: "powershell_title.jpg"
date: 2020-09-11 22:00:00 +0900
tags: [Windows, powershell, ps1, script] # add tag
related: powershell
categories: dev
---

Windows의 GUI 환경은 휼륭하다(최소한 Windows 10버전은) 하지만 가끔은 마우스로 원하는 기능을 찾기가 정말 힘들 때가 있다. 

특히 Windows에서 시험환경이나 반복되는 작업을 할 경우에는 아무래도 명령어라인 콘솔이 있으면 더 편리하게 작업을 할 수 있다. 

윈도우즈10 환경에서 간단한 명령어로 조금 편하게 쓸 수 있는 기능들을 정리해 본다. 

[PowerShell 설명서를 사용하는 방법](https://docs.microsoft.com/ko-kr/powershell/scripting/how-to-use-docs?view=powershell-5.1) 페이지를 읽고 시작하자. 

## PowerShell 버전 

먼저 도움말을 찾으려면 버전을 정확히 알고 있어야 한다.  
Windows 10에는 기본으로 5.1 버전이 설치되어 있다. 

```powershell
PS C:\Users\softr> Write-Output $PSVersionTable 
Name                           Value
----                           -----
PSVersion                      5.1.18362.752
```

## 파워쉘 프로파일 설정 

Linux 최초 구동시 ".bashrc"와 같이 PowerShell 기본 설정은 아래의 명령어로 확인 할 수 있고 자신만의 명령어를 정의하려면 "CurrentUserCurrentHost" 파일에 추가한다. 

```powershell
PS C:\Users\softr> $PROFILE | select *
AllUsersAllHosts       : C:\Windows\System32\WindowsPowerShell\v1.0\profile.ps1
AllUsersCurrentHost    : C:\Windows\System32\WindowsPowerShell\v1.0\Microsoft.PowerShell_profile.ps1
CurrentUserAllHosts    : C:\Users\softr\Documents\WindowsPowerShell\profile.ps1
CurrentUserCurrentHost : C:\Users\softr\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
Length                 : 75
```

## 기본 환경 설정 및 수정

Visual Studio Code를 사용한다면 터미널에서 code $PROFILE 하면 로그인 사용자의 기본 설정파일이 열린다.  

아래와 같이 인코딩 설정과 기본 Function을 정의해서 사용한다. 

{% gist butteryoon/5b0b7b848b4e14b452e9bca19f80d3b5 %} 

## 도움말 

PowerShell의 명령어(cmdlet)는 익숙하지가 않아 무조건 찾아본다. 

> "Get-Help -Name 검색어"  
> -Online 옵션을 사용하면 도움말 사이트로 연결된다. 

```powershell
❯ Get-Help -Name Uninstall

Name                              Category  Module                    Synopsis
----                              --------  ------                    --------
Uninstall-Package                 Cmdlet    PackageManagement         Uninstall-Package...
```

## 주로 쓰는 명령어 정리 

Linux 명령어 쉘에 익숙하다면 "Get-Alias" 명령어를 찾아보면 파워쉘의 기본 명령어를 알아볼 수 있다. 

```powershell
❯ Get-Alias curl

CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Alias           curl -> Invoke-WebRequest
```

### Uninstall-Package : 설치된 프로그램 삭제  

Get-Package -Name Ahn* 명령어로 프로그램 이름을 찾은 후 삭제한다.

```powershell
❯ Uninstall-Package -Name ipMonitor10
경고: Reboot is required to complete the uninstallation
operation.

Name                           Version          Source
----                           -------          ------
ipMonitor10                    10.9.1
```

### Get-Content : 파일 정보 확인 및 출력

파일의 내용이나 정보 확인. (cmd 에서는 type이었던거 같은데)

> cat, type 명령어도 모두 "Get-Content"와 매핑되어 있다. 

```powershell
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

cat 명령어도 Alias 설정 되어 있다. 

> ls, grep 등등 명령어로 get-alias 명령어로 찾아보면 파워쉘의 기본 명령어를 찾을 수 있다. 

```powershell
PS > get-alias cat

CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Alias           cat -> Get-Content
```

### Get-Process : 프로세스 정보 조회

로컬포트를 사용하는 프로세스 Id를 검색 하고 "Stop-Process -Name" 명령어로 Kill 할 수 있다. 

```powershell
❯ Get-Process -Id (Get-NetTCPConnection -LocalPort 58803).OwningProcess

Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
-------  ------    -----      -----     ------     --  -- -----------
    247      22     9216      20540       1.45  11060   1 python
```

### 실행중인 프로세스 파일버전 표시. 

```powershell
❯ Get-Process liveview -FileVersionInfo

ProductVersion   FileVersion      FileName
--------------   -----------      --------
1.0.0.1          1.0.0.1          C:\IIOT-LIVEVIEW\LIVEVIEW.exe
```

### Resolv-DnsName : 도메인의 IP 찾기.    

nslookup 명령어를 사용해도 된다.  
가끔 도메인이 필요할 때가 있어 duckdns.org 서비스를 이용하는데 내 랩탑의 IP가 제대로 업데이트 되었는지 확인 할 때 사용한다. 

```powershell
❯ Resolve-DnsName softroom.duckdns.org

Name                                           Type   TTL   Section    IPAddress
----                                           ----   ---   -------    ---------
softroom.duckdns.org                           A      60    Answer     106.xxx.xxx.xxx
```

### 디렉토리나 이름 및 파일 찾기

디렉토리에서 css 이름을 가진 디렉토리 찾기  

> Get-ChildItem -Directory
> Get-ChildItem -File

```powershell
❯ Get-ChildItem -Directory | Where-Object { $_.Name -eq 'css' }

    디렉터리: D:\Dropbox\Public\butteryoon.github.io

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
da----     2020-09-05   오후 7:32                css
```

지정한 하위 디렉토리 전체에서 특정이름을 가진 파일을 찾기

```powershell
❯ Get-ChildItem -Path C:\ -Include svccare* -Recurse 

디렉터리: C:\Program Files (x86)\infitec
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----      2017-12-13   오후 3:31         219520 svccare.exe

디렉터리: C:\Windows\Prefetch
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----      2020-09-09   오후 2:11          10298 SVCCARE.EXE-642142C3.pf
```

### 인터넷에서 파일 다운로드 

wget 명령어가 "Invoke-WebRequest"로 Alias 설정되어 있다. 

> wget.exe를 받아서 쓰는게 속 편하긴 하다. 

```powershell
wget http://ax.itunes.apple.com/detection/itmsCheck.js -o itemsCheck.js

PS D:\Dropbox\tools\CustomURL> Get-Alias wget

CommandType     Name                                               Version    Source
-----------     ----                                               -------    ------
Alias           wget -> Invoke-WebRequest
```

### Widows Uptime

Linux의 uptime 명령어와 같은 기능이며 파워쉘 함수로 등록해서 사용한다. 

```powershell
Function uptime {Get-WmiObject -Class Win32_OperatingSystem | Select-Object @{n='LastBoot';e={$_.ConvertToDateTime($_.LastBootUpTime)}}}

LastBoot
--------
2020-10-14 오후 4:09:40
```

## 참고 URL
- [1장 - PowerShell 시작](https://docs.microsoft.com/ko-kr/powershell/scripting/learn/ps101/01-getting-started?view=powershell-5.1)
- [PowerShell이란?](https://docs.microsoft.com/ko-kr/powershell/scripting/overview?view=powershell-7)
- [Windows Terminal Preview 릴리스](https://www.lesstif.com/pages/viewpage.action?pageId=71401723)
- [execution of scripts is disabled on this system.](https://www.hahwul.com/2017/08/powershell-execution-of-scripts-is.html)
- [Use Windows PowerShell to search for files](https://devblogs.microsoft.com/scripting/use-windows-powershell-to-search-for-files/)