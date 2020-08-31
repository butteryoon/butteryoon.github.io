---
layout: post
title: "Windows Terminal 써보기"
img: "M_WindowsTerminal_setting.jpg"
date: 2020-03-03 11:36:00 +0900
tags: [Windows, Terminal] # add tag
categories: tools
---

Windows의 기본 cmd 창이 드디어 쓸만한 도구로 바뀌고 있다. 

WSL2를 쓰려면 신규 Tminal을 쓰는걸 권장한다. 

[마이크로소프트 스토어](https://www.microsoft.com/ko-kr/p/windows-terminal-preview/9n0dx20hk701?activetab=pivot:overviewtab)에서 설치한다. 

## 설정파일은 수동으로 편집 한다. (프리뷰니까.. )    

아래의 경로에 Profiles.json
C:\Users\CHOI\AppData\Local\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\profiles.json  

## Remote SSH 터미널 

> LocalPortForwarding 기능이 지원되려나. 
> guid는 별도로 설정해야 한다고 하는데 없어도 우선 설정은 된다.  

``` 
{
	"hidden": false,
	"name": "IWS121",
	"commandline": "ssh lcsapp@192.168.0.111"
},
```

## profiles.json 샘플. 

```
    "profiles":
    {
        "defaults":
        {
            // Put settings here that you want to apply to all profiles
            "historySize" : 9001,
            "fontFace": "D2Coding",
            "fontSize": 14,
            "colorScheme" : "Solarized Dark"
        },
        "list":
        [
            {
                "hidden": false,
                "name": "WIS121",
                "commandline": "ssh lcsapp@192.168.0.121"
            }
            
        ]
    },
``` 
### PSReadLine 플러그인. 

Windows Terminal 에서 git를 쓸 때 명령어 프롬프트에 상태를 표시해주는 플러그인 

[자습서: Windows 터미널에서 Powerline 설정](https://docs.microsoft.com/ko-kr/windows/terminal/tutorials/powerline-setup)를 참고한다. 

![powerline]({{site.baseurl}}/assets/img/M_wt_powerline-01.jpg)

### 화면분할 설정

기본 명령어 터미널과 파워쉘 그리고 WSL창을 분할하여 구성할 수 있다. 
[Windows 터미널에 명령줄 인수 사용](https://docs.microsoft.com/ko-kr/windows/terminal/command-line-arguments?tabs=windows) 참고. 

```
wt new-tab "cmd" `; split-pane -p "Windows PowerShell" `; split-pane -H wsl.exe
```

## 참고 URL
-  [Windows Terminal Preview 릴리즈](https://www.lesstif.com/pages/viewpage.action?pageId=71401723)
-  [execution of scripts is disabled on this system.](https://www.hahwul.com/2017/08/powershell-execution-of-scripts-is.html)