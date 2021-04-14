---
layout: post
title: "Windows10 Clean"
description: "Windows 10 PC 깔끔하게 관리하기"
img: "vscode-title.jpg"
date: 2020-09-08 20:00:00 +0900
last_modified_at: 2024-04-14 18:00:00 +0900
tags: [Windows10, Visual Studio Code, vscode, Font, D2Coding] # add tag
related: Windows10
categories: tools
---

Windows 10 PC를 사용하면서 필요한 것들을 기억나는대로 정리해본다. 

<!--more-->

## svccare.exe 삭제. 

언재부터인가 PC가 이상한 느낌이 들어 찾아보면 돌고 있는 프로세스. Google에 검색해도 별 정보가 없어서 그냥 지우기로 한다.  

아래 디렉토리는 모두 지운다. 

```
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\Ahns.lnk 

❯ Get-ChildItem -Path C:\ -Include svccare* -Recurse

디렉터리: C:\Program Files (x86)\infitec
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----      2017-12-13   오후 3:31         219520 svccare.exe

디렉터리: C:\Windows\Prefetch
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----      2020-09-09   오후 2:11          10298 SVCCARE.EXE-642142C3.pf

디렉터리: C:\Windows.old\WINDOWS\Prefetch
Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----      2020-08-13   오후 6:22           8929 SVCCARE.EXE-642142C3.pf
``` 

## Windows 10 sshd

Windows 10 PC에 SSH 서버 설정. 데스크탑으로 쓸 때는 거의 필요 없는데 WSL을 설치하면서 간단한 스크립트 테스트용으로 쓴다. 

[Windows의 OpenSSH](https://docs.microsoft.com/ko-kr/windows-server/administration/openssh/openssh_overview)
[Configure SSH Server With Windows 10 Native Way](https://medium.com/rkttu/set-up-your-ssh-server-in-windows-10-native-way-1aab9021c3a6)
[Windows 10 네이티브 방식으로 SSH 서버 설정하기](https://medium.com/beyond-the-windows-korean-edition/windows-10-네이티브-방식으로-ssh-서버-설정하기-64988d87349)

```powershell
PS D:\Dropbox\tools\covid> ssh localhost
softr@localhost's password:
```

ssh 접속시 기본 쉘은 PowerShell로 변경 

```powershell
PS C:\IIOT-LIVEVIEW> New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe" -PropertyType String -Force


DefaultShell : C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
PSPath       : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SOFTWARE\OpenSSH
PSParentPath : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\SOFTWARE
PSChildName  : OpenSSH
PSDrive      : HKLM
PSProvider   : Microsoft.PowerShell.Core\Registry
```

## [chocolatey](https://chocolatey.org)

윈도우즈 패키지 관리자로 터미널에서 패키지 설치/삭제/업데이트 관리를 할 수 있다. 

> WinRAR 패키지 찾아보기 

```powershell
PS C:\Users\softr> choco search winRAR
Chocolatey v0.10.15
winrar 5.91 [Approved] Downloads cached for licensed users
1 packages found.

Did you know Pro / Business automatically syncs with Programs and
 Features? Learn more about Package Synchronizer at
 https://chocolatey.org/compare
```

검색결과 버전이 다른 패키지가 있을 경우 "--version" 파라미터로 특정 버전을 설치할 수 있다. 

```powershell
PS C:\Users\softr> choco install winrar --version 5.91
Chocolatey v0.10.15
Installing the following packages:
winrar
By installing you accept licenses for the packages.
Progress: Downloading winrar 5.91... 100%
winrar v5.91 [Approved]
winrar package files install completed. Performing other installation steps.
The package winrar wants to run 'chocolateyInstall.ps1'.
Note: If you don't run this script, the installation will fail.
Note: To confirm automatically next time, use '-y' or consider:
choco feature enable -n allowGlobalConfirmation
Do you want to run the script?([Y]es/[A]ll - yes to all/[N]o/[P]rint): A

Downloading winrar 64 bit
  from 'https://www.rarlab.com/rar/winrar-x64-591kr.exe'
Progress: 100% - Completed download of C:\Users\softr\AppData\Local\Temp\chocolatey\winrar\5.91\winrar-x64-591kr.exe (3.18 MB).
Download of winrar-x64-591kr.exe (3.18 MB) completed.
Error - hashes do not match. Actual value was 'B79A53D84C41F50AAD0CB38D091CB26726F512502529503D8C94B47B3D070FA5'.
ERROR: Checksum for 'C:\Users\softr\AppData\Local\Temp\chocolatey\winrar\5.91\winrar-x64-591kr.exe' did not meet 'd9751d0ebdd3931aaf1f1eef13d0b1c56ed15845bd3307e0f4b9540e8c999f1d' for checksum type 'sha256'. Consider passing the actual checksums through with --checksum --checksum64 once you validate the checksums are appropriate. A less secure option is to pass --ignore-checksums if necessary.
The install of winrar was NOT successful.
Error while running 'C:\ProgramData\chocolatey\lib\winrar\tools\chocolateyInstall.ps1'.
 See log for details.

Chocolatey installed 0/1 packages. 1 packages failed.
 See the log for details (C:\ProgramData\chocolatey\logs\chocolatey.log).

Failures
 - winrar (exited -1) - Error while running 'C:\ProgramData\chocolatey\lib\winrar\tools\chocolateyInstall.ps1'.
 See log for details.
PS C:\Users\softr> choco install winrar --checksum64 B79A53D84C41F50AAD0CB38D091CB26726F512502529503D8C94B47B3D070FA5
```

## putty 설정 파일 저장 

putty 세션설정 정보 백업, 설정이 레지스트리에 등록되어 있다. 

```
regedit /E "putty-settings.reg" HKEY_CURRENT_USER\Software\SimonTatham
```


## 참고 URL  

- [Install Chocolatey](https://chocolatey.org/install)
