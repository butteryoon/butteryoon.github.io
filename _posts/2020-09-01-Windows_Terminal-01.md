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

WSL을 쓰려면 Windows Terminal은 거의 필수 라고 생각하자. 

[What is Terminal](https://devblogs.microsoft.com/commandline/windows-terminal-preview-1-3-release/) 한번 읽어두자. 

### Install  

설치는 [MS Store](https://aka.ms/terminal)에서 할 수도 있고 [GitHub](https://github.com/microsoft/terminal) 페이지를 보면 [Chocolatey](https://chocolatey.org)에서도 할 수 있다. 

Windows 10의 검색창에서 wt 명령어로 실행 가능하다.  

### Windows Terminal 창 분할 

PowerShell, cmd, wsl2 터미널 이렇게 세개의 창을 배치하고 WSL 창에서 간단한 스크립트를 사용할 대 편리하다.   

※ 창분할 기능 [Panes in Windows Terminal](https://docs.microsoft.com/en-us/windows/terminal/panes) 참고. 

![Terminal Layout]({{site.bashurl}}/assets/img/Terminal_Layout.png) 

### 서드파티 툴 

터미널이 쓸 만해 지니 [PowerShell](https://butteryoon.github.io/dev/2020/07/16/WindowsPowerShell.html)에도 관심이 가서 쪼금 건드려 보는 중이다. 

git 프로프트를 편하게 표시해주는 [Windows Terminal Powerline](https://docs.microsoft.com/ko-kr/windows/terminal/tutorials/powerline-setup) 을 설치해보니 Windows Terminal에서 파워쉘 명령어에 조금 친숙해졌다. 

### Setting 

![Terminal Setting]({{site.bashurl}}/assets/img/WindowsTerminal_setting.png)   

Windows Terminal의 설정은 json 파일로 되어 있어 편집이 가능하다. (json 확장자를 Visual Studio Code로 연결해 놓으면 편하게 설정 가능하다.) 

[명령어 팔레트](https://docs.microsoft.com/ko-kr/windows/terminal/command-palette)를 정의하여 Command Line 모드를 사용할 수도 있다.

> 설정파일 : settings.json 

```html
C:\Users\사용자
\AppData\Local\Packages
\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState
\settings.json 
```


### 참고 URL
-  [Windows Terminal GihHub](https://github.com/microsoft/terminal)
-  [Windows 터미널이란?](https://docs.microsoft.com/ko-kr/windows/terminal/)