---
layout: post
comments: true
title: "PowerShell 프로파일 기본 설정"
description: "내가 쓰는 PowerShell 프로파일 기준으로 정리한 기본 설정 방법"
img: powershell_title.jpg
date: 2026-07-15 00:45:00 +0900
last_modified_at: 2026-07-15 00:45:00 +0900
tags: [PowerShell, profile, oh-my-posh, alias, Windows Terminal] # add tag
related: powershell
categories: tools
---

새 PC를 세팅할 때마다 PowerShell 프로파일을 다시 만들게 되는데, 지금 쓰고 있는 프로파일을 기준으로 기본 설정 방법을 정리해둔다.

<!--more-->

## 프로파일 파일 위치

PowerShell 7(pwsh) 기준으로 프로파일 경로는 `$PROFILE` 변수로 확인할 수 있다.

```powershell
❯ $PROFILE
C:\Users\<사용자>\Documents\PowerShell\Microsoft.PowerShell_profile.ps1
```

파일이 없으면 만들어주고, 편집기로 연다.

```powershell
if (-not (Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
nvim $PROFILE   # 또는 notepad $PROFILE
```

프로파일은 PowerShell이 시작될 때마다 실행되는 스크립트라서, 여기에 프롬프트 테마·별칭·함수를 넣어두면 매 세션에서 바로 쓸 수 있다.

## 프롬프트 테마: oh-my-posh

[oh-my-posh](https://ohmyposh.dev/)로 프롬프트를 꾸민다. winget으로 설치하고 프로파일에 초기화 한 줄만 넣으면 된다.

```powershell
winget install JanDeDobbeleer.OhMyPosh
```

```powershell
# $PROFILE
oh-my-posh init pwsh | Invoke-Expression
```

테마를 지정하고 싶으면 `--config`에 테마 파일을 넘긴다. 아이콘이 깨지면 [Nerd Font](https://www.nerdfonts.com/)를 설치하고 터미널 글꼴로 지정해야 한다.

```powershell
oh-my-posh init pwsh --config "$env:POSH_THEMES_PATH\atomic.omp.json" | Invoke-Expression
```

## 명령어 추천: PowerToys Command Not Found

[PowerToys](https://learn.microsoft.com/windows/powertoys/)의 Command Not Found 기능을 켜면, 없는 명령어를 입력했을 때 winget으로 설치할 수 있는 패키지를 추천해준다. PowerToys 설정에서 활성화하면 프로파일에 아래 블록이 자동으로 추가된다.

```powershell
Import-Module -Name Microsoft.WinGet.CommandNotFound
```

## 기본 별칭 정리

PowerShell은 `curl`, `cat` 같은 유닉스 명령어를 자체 cmdlet의 별칭으로 잡아놔서, 진짜 curl이나 [bat](https://github.com/sharkdp/bat) 같은 도구를 쓰려면 기본 별칭을 먼저 지워야 한다.

```powershell
# 기본 별칭 제거
if ( Test-Path Alias:curl ) { Remove-Item Alias:curl }
if ( Test-Path Alias:cat )  { Remove-Item Alias:cat }

# 새 별칭
Set-Alias vi nvim
Set-Alias cat bat
Set-Alias magick -Value "C:\Program Files\ImageMagick-7.1.1-Q16-HDRI\magick.exe"
```

## 디렉토리 이동 함수

자주 가는 디렉토리는 함수로 만들어두면 편하다.

```powershell
Function cwork { Set-Location "C:\work\workspace\" }
Function cdesk { Set-Location "$env:USERPROFILE\Desktop" }
Function cdocs { Set-Location "$env:USERPROFILE\Documents" }
```

## 환경변수 등록

API 키 같은 환경변수도 프로파일에서 등록할 수 있다. `"User"` 스코프로 지정하면 사용자 환경변수로 저장된다.

```powershell
[System.Environment]::SetEnvironmentVariable("MY_API_KEY", "<발급받은 키>", "User")
```

> 프로파일을 dotfiles 저장소 등에 공개할 계획이 있다면 키 값을 프로파일에 직접 적지 말고 별도 파일로 분리하는 것이 안전하다.

## 유틸리티 함수

리눅스 습관대로 쓰거나 자주 확인하는 정보는 함수로 정리해뒀다.

```powershell
Function ver { Write-Output $PSVersionTable }                  # 버전 확인
Function export { Get-ChildItem -Path Env:\ }                  # 환경변수 목록
Function Get-IPinfo { ipconfig | Select-String IPv4 }          # 내부 IP
Function Get-MyIPinfo { curl "http://api.ipify.org?format=json" }  # 공인 IP
Function uptime { Get-WmiObject -Class Win32_OperatingSystem |
  Select-Object @{n='LastBoot';e={$_.ConvertToDateTime($_.LastBootUpTime)}} }
Function Get-Title { Get-Process | Where-Object {$_.mainWindowTitle} |
  Format-Table Id, Name, mainWindowTitle -AutoSize }           # 창 있는 프로세스
Function Get-WiFi-Profile { netsh wlan show profile }          # 저장된 Wi-Fi 목록
```

SSH 역터널을 함수로 만들어두면 원격 접속 준비도 한 단어로 끝난다.

```powershell
Function oc01 { ssh -R 53022:localhost:22 oc01 }
```

## 적용

프로파일을 수정한 뒤에는 새 세션을 열거나 다시 로드한다.

```powershell
. $PROFILE
```

## 참고

- [about_Profiles - PowerShell 공식 문서](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_profiles){:target="_blank"}
- [oh-my-posh](https://ohmyposh.dev/){:target="_blank"}
- [PowerToys Command Not Found](https://learn.microsoft.com/windows/powertoys/cmd-not-found){:target="_blank"}
- [Windows PowerShell 시작하기]({{site.baseurl}}/dev/2020/09/11/WindowsPowerShell.html){:target="_blank"}
