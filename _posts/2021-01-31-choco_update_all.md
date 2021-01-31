---
layout: post
title: "choco upgrade all -y"
description: "Windows 패키지 관리자 Chocolatey로 설치된 패키지 전체 업데이트"
img: m_choco_title.webp
date: 2021-01-31 22:00:00 +0900
last_modified_at: 2021-01-31 22:00:00 +0900
tags: [terminal, chocolatey] # add tag
related: terminal
categories: tools
---

Windows 패키지 관리자 Chocolatey로 설치된 패키지 전체 업데이트를 해보고 영상으로 남겨둔다. 
<!--more-->

## 업그레이드 전체 패키지

기본 명령어는 **"choco upgrade all"**을 진행하면 각 패키지 마다 확인을 하기 때문에 **"-y"** 파라미터를 붙여 자동으로 업그레이드 스크립트가 실행되도록 해준다. 

```powershell  
> choco upgrade all -y
Chocolatey v0.10.15
Upgrading the following packages:
all
By upgrading you accept licenses for the packages.
... (skip) 
```

## 업그레이드 목록 

업그레이드가 끝나면 업그레이드 성공 및 실패 목록을 표시한다. 

```powershell 
Upgraded:
 - vim v8.2.2434
 - filezilla v3.52.2
 - git.install v2.30.0.2
 - dbeaver v7.3.3
 - winrar v6.0.0.20210102
 - git v2.30.0.2
 - cpu-z.portable v1.95.0.20210128
 - jre8 v8.0.281
 - microsoft-windows-terminal v1.5.10271.0
 - neovim v0.4.4.20210116
 - vivaldi v3.6.2165.34
 - potplayer v21.01.27
 - vivaldi.portable v3.6.2165.34
 - fzf v0.25.0
 - powertoys v0.29.3
 - teamviewer v15.14.3
 - picpick.portable v5.1.4
 - wireshark v3.4.3
 - slack v4.12.2
 - obs-studio.install v26.1.1
 - obs-studio v26.1.1
 - visualstudio2019community v16.8.4.0
```

## 업그레이드 과정 확인 영상.

{% youtube "https://youtu.be/MPWg6l171jo" %}

## 참고 URL
- [Chocolatey Upgrade](https://docs.chocolatey.org/en-us/choco/commands/upgrade#mainContent){:target="_blank"}
