---
layout: post
title: "Windows Terminal 써보기"
img: WindowsTerminalPreview.jpg
date: 2020-03-03 11:36:00 +0900
tags: [Windows Terminal Preview] # add tag
categories: dev
---

## Windows Terminal Preview

Windows의 기본 cmd 창이 드디어 쓸만한 도구로 바뀌고 있는 듯 .. 

[마이크로소프트 스토어](https://www.microsoft.com/ko-kr/p/windows-terminal-preview/9n0dx20hk701?activetab=pivot:overviewtab)에서 설치한다. 

## 설정파일은 수동으로 편집 한다. (프리뷰니까.. ) 
![Terminal Setting]({{site.bashurl}}/assets/img/WindowsTerminal_setting.png)
 
> 아래의 경로에 Profiles.json
> C:\Users\CHOI\AppData\Local\Packages\Microsoft.WindowsTerminal_8wekyb3d8bbwe\LocalState\profiles.json 

## ssh 터미널을 연결 하려면 아래와 같이 cmd 를 설정한다.  

> guid는 따로 설정 안해도 됨. 
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
                // Make changes here to the powershell.exe profile
                "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
                "name": "Windows PowerShell",
                "commandline": "powershell.exe",
                "hidden": false
            },
            {
                // Make changes here to the cmd.exe profile
                "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
                "name": "cmd",
                "commandline": "cmd.exe",
                "hidden": false
            },
            {
                "guid": "{c6eaf9f4-32a7-5fdc-b5cf-066e8a4b1e40}",
                "hidden": false,
                "name": "Ubuntu-18.04",
                "source": "Windows.Terminal.Wsl"
            },
            {
                "hidden": false,
                "name": "WIS121",
                "commandline": "ssh lcsapp@192.168.0.121"
            },
            {
                "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
                "hidden": false,
                "name": "Azure Cloud Shell",
                "source": "Windows.Terminal.Azure"
            }
        ]
    },
``` 


## 참고 URL
-  [Windows Terminal Preview 릴리즈](https://www.lesstif.com/pages/viewpage.action?pageId=71401723)
