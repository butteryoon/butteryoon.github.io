---
layout: post
title: "Windows Terminal"
img: "M_WindowsTerminalPreview.png"
date: 2020-09-01 21:00:00 +0900
tags: [Windows, Terminal, wsl2] # add tag
categories: tools
---

Windows의 터미널이라니 !!  

CMD 창으로 할 수 있었던 일은 ipconfig 로 IP확인하기, 업무용을 VPN 연결시 우회용 라우팅 테이블 설정, 그리고 ... 그리고 ... 없었다. 

Windows 터미널은 사실 뭔가 새로운 걸 쓰기를 좋아하지 않는 이들에게는 별다른 감흥이 없을 수 있다. 

나도 사실 WSL을 써보려고 하기 전에는 Windows에서 터미널은 거의 필요가 없는 도구였지만 지금은 Windows Terminal은 거의 필수 구동 프로그램이 되었다. 

[Windows 터미널이란?](https://docs.microsoft.com/ko-kr/windows/terminal/) 한번 읽어두자. 

## Install  

설치는 [MS Store](https://aka.ms/terminal)에서 할 수도 있고 [GitHub](https://github.com/microsoft/terminal) 페이지를 보면 [Chocolatey](https://chocolatey.org)에서도 할 수 있다. 

```powershell
PS > choco list microsoft-windows-terminal -lo
Chocolatey v0.10.15
microsoft-windows-terminal 1.2.2381.0
1 packages installed.
```

설치 후에는 Windows 10의 검색창에서 wt 명령어로 실행 가능하다.  

## Windows Terminal(PowerShell) 한글설정(UTF-8)   

Windows Terminal 에서 PowerShell을 쓸 경우 git 결과의 한글이 깨지게 되어 기본설정을 UTF-8로 변경한다. 

아래 "$PROFILE" 환경변수에서 사용자 프로파일을 확인하고 아래 LC_ALL 값을 UTF-8로 설정한다.  

> 프로파일 경로 확인  

```powershell
PS D:\Dropbox\Public\butteryoon.github.io> $PROFILE | select *

AllUsersAllHosts       : C:\Windows\System32\WindowsPowerShell\v1.0\profile.ps1
AllUsersCurrentHost    : C:\Windows\System32\WindowsPowerShell\v1.0\Microsoft.PowerShell_profile.ps1
CurrentUserAllHosts    : C:\Users\softr\Documents\WindowsPowerShell\profile.ps1
CurrentUserCurrentHost : C:\Users\softr\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1
``` 

> 파워쉘을 실행할 경우 profile.ps1 파일 내용

```powershell
$env:LC_ALL='C.UTF-8'
$PSDefaultParameterValues['Out-File:Encoding'] = 'utf8
```

## Windows Terminal 창 분할 

PowerShell, cmd, wsl2 터미널 이렇게 세개의 창을 배치하고 WSL 창에서 간단한 스크립트를 사용할 대 편리하다.   

※ 창분할 기능 [Panes in Windows Terminal](https://docs.microsoft.com/ko-kr/windows/terminal/panes) 참고. 

![Terminal Layout]({{site.bashurl}}/assets/img/Terminal_Layout.png) 

## Setting 

![Terminal Setting]({{site.bashurl}}/assets/img/WindowsTerminal_setting.png)   

Windows Terminal의 설정은 json 파일로 되어 있어 편집이 가능하다. (json 확장자를 Visual Studio Code로 연결해 놓으면 편하게 설정 가능하다.)  

> 설정파일 : settings.json 

```
C:\Users\사용자
\AppData\Local\Packages
\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState
\settings.json 
```

## 명령 팔레트 

[명령 팔레트](https://docs.microsoft.com/ko-kr/windows/terminal/command-palette)를 정의하여 Windows 터미널 내에서 실행할 수있는 명령을 확인할 수 있다. 

## 서드파티 툴 

터미널이 쓸 만해 지니 [PowerShell]({{site.baseurl}} {% link _posts/2020-07-16-WindowsPowerShell.md %})에도 관심이 가서 쪼금 건드려 보는 중이다. 

git 프로프트를 편하게 표시해주는 [자습서: Windows 터미널에서 Powerline 설정](https://docs.microsoft.com/ko-kr/windows/terminal/tutorials/powerline-setup) 을 설치해보니 Windows Terminal에서 파워쉘 명령어에 조금 친숙해졌다. 


## 참고 URL
- [Windows Terminal GihHub](https://github.com/microsoft/terminal)
- [Windows 터미널이란?](https://docs.microsoft.com/ko-kr/windows/terminal/)
- [[PowerShell] 파워쉘에서 한글 깨짐 문제 해결](https://psychoria.tistory.com/737)
- [Windows PowerShell 기본]({{site.baseurl}} {% link _posts/2020-07-16-WindowsPowerShell.md %})