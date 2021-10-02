---
layout: post
comments: true
title: "PowerShell 빌트인 alias 바꾸기"
description: "윈도우즈 파워쉘에 기본 alias 설정을 바꿔본다."
img: powershell_title.jpg
date: 2021-10-01 11:00:00 +0900
last_modified_at: 2021-10-01 11:00:00 +0900
tags: [windows, powershell, alias, batcat] # add tag
related: powershell
categories: tools
---

Windows 파워쉘에 빌트인된 Alias 설정을 변경하는 방법에 대해 알아본다. 
<!--more-->

## batcat

윈도우즈 터미널에서 파일의 내용을 확인하고 싶을 때는 `Get-Content` 명령어를 사용한다. 리눅스 터미널에서는 *cat* 명령어를 쓸 수 있는데 파월쉘에도 기본 리눅스 형식의 명령어가 빌트인으로 aliasing 되어 있다. 

[batcat](https://github.com/sharkdp/bat){:target="_blank"} 명령어는 러스트(Rust)로 만들어진 *cat* 클론인데 텍스트의 내용을 보여주는 기본 기능외에 문서 포맷에 따라 구분 하이라이팅을 지원하는 등 쓸모가 많아 빌트인된 *cat* 명령어를 *bat*으로 바꾸려고 한다. 

## Set-Alias 

*cat* 명령어는 빌트인으로 alias 되어 있어 `Set-Alias` 명령어로 설정하려면 아래와 같이 오류가 생긴다. 

```powershell
Set-Alias : 'cat' 별칭에서 AllScope 옵션을 제거할 수 없습니다.
위치 C:\Users\softr\Documents\WindowsPowerShell\Microsoft.PowerShell_profile.ps1:6 문자:1
+ Set-Alias cat bat
+ ~~~~~~~~~~~~~~~~~
    + CategoryInfo          : WriteError: (cat:String) [Set-Alias], SessionStateUnauthorizedAccessException
    + FullyQualifiedErrorId : AliasAllScopeOptionCannotBeRemoved,Microsoft.PowerShell.Commands.SetAliasCommand
```

아래와 같이 설정된 alias 를 지우고 새로 만들어줘야 한다.  

> 파워쉘 버전 5.1에서는 `Remove-Alias` 명령어가 없어 `Remove-Item` 명령어를 사용한다.  

```powershell
    Remove-Item Alias:/cat 
    Set-Alias cat bat
```

## 프로파일에 등록 

위에 설정한 Alias는 현재 터미널에만 적용되기 때문에 프로파일에 등록하기로 한다. 아래와 같이 기존에 cat 으로 설정된 alias가 있으면 삭제하고 새로운 alias를 등록한다. 

```powershell
> code $PROFILE

if ( Test-Path Alias:cat ) { Remove-Item Alias:/cat }
Set-Alias cat bat
```

## tl;dr

파워쉘 버전 5.x 에서는 `Remove-Item Alias:/` 명령어로 기존 Alias를 지우고 `Set-Alias`로 새로운 alias를 만들어준다. 

*curl* 명령어도 *curl -> Invoke-WebRequest*로 되어 있는데 지우고 *curl.exe*을 쓴다. 

## 참고

- [How do I permanently remove a default Powershell alias?](https://superuser.com/questions/883914/how-do-i-permanently-remove-a-default-powershell-alias){:target="_blank"}
