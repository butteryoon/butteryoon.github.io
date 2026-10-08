---
layout: post
comments: true
title: "PowerShell 프로파일 기본 설정 — 그리고 셸 시작이 6.8초 느려진 이유"
description: "PowerShell 7 프로파일의 네 가지 경로, 5.1과 분리되는 이유, 별칭·함수 설정법을 정리한다. 그리고 프로파일에 환경변수를 넣었다가 셸 시작이 6.8초 느려진 과정을 측정값과 함께 기록한다."
img: powershell_title.jpg
date: 2026-07-15 00:45:00 +0900
last_modified_at: 2026-10-08 22:35:00 +0900
tags: [PowerShell, profile, oh-my-posh, alias, Windows Terminal, 환경변수, 셸 시작속도, powershell] # add tag
related: powershell
categories: tools
redirect_from:
  - /tools/2026/07/14/powershell_profile.html
---

새 PC를 세팅할 때마다 PowerShell 프로파일을 다시 만들게 되는데, 지금 쓰고 있는 프로파일을 기준으로 기본 설정 방법을 정리해둔다. 그리고 이 글을 손보면서 프로파일 로딩 시간을 처음 재 봤는데, **셸 시작이 6.8초 느려져 있었다.** 원인과 고친 방법을 뒤쪽에 함께 적는다.

<!--more-->

## 프로파일은 하나가 아니다

대부분의 글이 `$PROFILE` 하나만 소개하는데, 실제로는 **네 개**가 있다. `$PROFILE`은 그중 하나를 가리키는 문자열일 뿐이고, 나머지는 속성으로 꺼낸다.

```powershell
❯ $PROFILE | Get-Member -MemberType NoteProperty
```

PowerShell 7.6 기준으로 이렇게 나온다.

| 범위 | 경로 |
|---|---|
| `CurrentUserCurrentHost` | `~\Documents\PowerShell\Microsoft.PowerShell_profile.ps1` |
| `CurrentUserAllHosts` | `~\Documents\PowerShell\profile.ps1` |
| `AllUsersCurrentHost` | `C:\Program Files\PowerShell\7\Microsoft.PowerShell_profile.ps1` |
| `AllUsersAllHosts` | `C:\Program Files\PowerShell\7\profile.ps1` |

`$PROFILE`을 그냥 쓰면 **`CurrentUserCurrentHost`** 를 가리킨다. "CurrentHost"란 지금 띄운 호스트, 즉 콘솔(`Microsoft.PowerShell`)을 뜻한다. VS Code 통합 터미널은 호스트가 달라서(`Microsoft.VSCode`) 이 파일을 읽지 않는다. 터미널에서는 되는데 VS Code에서는 별칭이 없다면 십중팔구 이것이 원인이다. 호스트를 가리지 않으려면 `profile.ps1`(`CurrentUserAllHosts`) 쪽에 넣는다.

### PowerShell 5.1과 7은 프로파일을 공유하지 않는다

Windows에 기본 설치된 PowerShell 5.1(`powershell.exe`)과 별도로 설치하는 PowerShell 7(`pwsh.exe`)은 **폴더부터 다르다.**

| 버전 | 프로파일 폴더 |
|---|---|
| Windows PowerShell 5.1 | `~\Documents\WindowsPowerShell\` |
| PowerShell 7 | `~\Documents\PowerShell\` |

`WindowsPowerShell` 과 `PowerShell`, 폴더 이름 한 단어 차이다. 설정이 적용되지 않는다면 지금 쓰는 셸이 어느 쪽인지부터 확인한다.

```powershell
❯ $PSVersionTable.PSVersion
```

이 글은 전부 PowerShell 7 기준이다. 5.1을 쓰고 있다면 [Windows PowerShell 시작하기]({{site.baseurl}}/dev/2020/09/11/WindowsPowerShell.html){:target="_blank"}에서 설치부터 보는 것이 빠르다.

## 프로파일 만들기

파일이 없으면 만들어주고, 편집기로 연다.

```powershell
if (-not (Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
nvim $PROFILE   # 또는 notepad $PROFILE
```

`New-Item -Force`는 **상위 폴더가 없을 때 만들어 주는 용도**로 쓴다. 다만 파일이 이미 있으면 내용을 지우고 새로 만들기 때문에, 위처럼 `Test-Path`로 먼저 막아야 한다. 이미 쓰던 프로파일을 `-Force`로 날리는 사고가 흔하다.

프로파일은 PowerShell이 시작될 때마다 실행되는 스크립트다. 뒤에서 다루지만, **"매번 실행된다"는 점이 이 글의 후반부 전체를 좌우한다.**

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

## 별칭 — PowerShell 7에서는 사정이 다르다

여기서 버전을 확인하고 넘어가야 한다. **PowerShell 7에는 `curl`·`wget` 별칭이 없다.**

```powershell
❯ Get-Alias curl
Get-Alias: ... 'curl' 이름의 별칭을 찾을 수 없습니다.
```

5.1에서는 `curl`이 `Invoke-WebRequest`의 별칭이라 진짜 `curl.exe`를 가리고, 그래서 `Remove-Item Alias:curl` 같은 처리가 필요했다. PowerShell 6에서 이 별칭들이 제거됐기 때문에 **7을 쓴다면 그 줄은 지워도 된다.** 오래된 글을 그대로 옮겨 적다가 남는 흔적인데, 내 프로파일에도 몇 년째 남아 있었다.

반면 `cat`(→`Get-Content`)·`ls`(→`Get-ChildItem`)·`rm`(→`Remove-Item`)은 7에서도 그대로 남아 있다. 이들은 옵션이 걸려 있지 않아 그냥 덮어쓰면 된다.

```powershell
Set-Alias vi nvim
Set-Alias cat bat
Set-Alias magick -Value "C:\Program Files\ImageMagick-7.1.1-Q16-HDRI\magick.exe"
```

다만 **덮어쓸 수 없는 별칭이 따로 있다.** `ReadOnly`나 `Constant` 옵션이 걸린 것들인데, PowerShell 7.6 기준 90개나 된다. `%`(ForEach-Object), `?`(Where-Object), `diff`, `compare` 같은 것들이다. 이쪽을 바꾸려면 `-Force`가 필요하다.

```powershell
❯ Get-Alias | Where-Object { $_.Options -match 'ReadOnly|Constant' } | Measure-Object
Count : 90

❯ Get-Alias cat | Select-Object Name, Definition, Options
Name Definition  Options
---- ----------  -------
cat  Get-Content    None
```

바꾸기 전에 `Options`부터 보는 습관을 들이면, `-Force`를 언제 붙여야 하는지 외울 필요가 없다.

### 별칭으로 안 되는 것은 함수로

별칭은 **인자를 받지 못한다.** `Set-Alias ll "ls -l"` 같은 건 성립하지 않는다. 옵션을 붙이거나 경로를 넘겨야 하면 함수로 만든다.

```powershell
Function cwork { Set-Location "C:\work\workspace\" }
Function cdesk { Set-Location "$env:USERPROFILE\Desktop" }
Function cdocs { Set-Location "$env:USERPROFILE\Documents" }
```

## 환경변수를 프로파일에 넣지 마라

이 글에서 가장 중요한 부분이다. 원래 이 글에는 아래 코드가 "API 키도 프로파일에서 등록할 수 있다"는 설명과 함께 들어 있었다.

```powershell
# 이렇게 하지 말 것
[System.Environment]::SetEnvironmentVariable("MY_API_KEY", "<발급받은 키>", "User")
```

문제가 두 가지다.

**첫째, 매 세션마다 레지스트리에 쓴다.** `"User"` 스코프는 사용자 환경변수를 **영구 저장**하는 호출이다. 레지스트리에 기록하고, 다른 프로그램에 변경을 알리는 `WM_SETTINGCHANGE` 메시지를 브로드캐스트한다. 한 번만 하면 될 일을 셸을 열 때마다 반복한다.

얼마나 비싼지 재 봤다. 세 번 측정해 평균을 냈다.

| 구간 | 소요 |
|---|---|
| 기준 — `pwsh -NoProfile`로 켜고 끄기 | 514ms |
| 기준 + `SetEnvironmentVariable` 5줄 | **3,263ms** |
| 기준 + `oh-my-posh init` | 1,806ms |
| 기준 + `Import-Module WinGet.CommandNotFound` | 1,453ms |

**환경변수 5줄이 2.7초를 쓴다.** 프롬프트 테마보다 두 배 비싸다. 셸을 하루에 서른 번 연다면 80초를 그냥 버리는 셈이다.

**둘째, 키가 평문으로 남는다.** 프로파일은 그냥 텍스트 파일이다. dotfiles 저장소에 올리면 그대로 공개되고, 올리지 않더라도 화면 공유나 백업 동기화 폴더를 타고 나간다. 더구나 `"User"` 스코프로 쓰는 순간 **레지스트리에도 평문으로 복사**된다.

### 대신 이렇게

**한 번만 실행하는 별도 스크립트로 분리한다.** 프로파일이 아니라 손으로 한 번 돌리는 파일이다.

```powershell
# setup-env.ps1 — 새 PC 세팅 때 한 번만 실행
[System.Environment]::SetEnvironmentVariable("MY_API_KEY", (Read-Host "키 입력"), "User")
```

`Read-Host`로 받으면 파일에 값이 남지 않는다. 한 번 등록하면 이후 모든 세션에서 `$env:MY_API_KEY`로 읽힌다. **프로파일에는 아무것도 적을 필요가 없다.**

이 세션에서만 쓸 값이라면 레지스트리를 건드리지 않는 쪽이 맞다.

```powershell
$env:MY_API_KEY = "..."   # 현재 세션 한정, 즉시 끝남
```

제대로 하려면 Microsoft가 제공하는 비밀 저장소를 쓴다. 값이 암호화돼 저장된다.

```powershell
Install-Module Microsoft.PowerShell.SecretManagement, Microsoft.PowerShell.SecretStore
Set-Secret -Name MY_API_KEY -Secret "..."      # 한 번만
$key = Get-Secret -Name MY_API_KEY -AsPlainText  # 필요할 때만
```

이것도 프로파일에 넣지 않는다. 필요한 스크립트에서 그때그때 꺼내 쓴다.

## 프로파일 시작 시간 재기

프로파일은 조용히 느려진다. 한 줄씩 늘어나는 동안 체감이 안 되다가, 어느 날 셸이 답답해진다. 재는 방법은 간단하다.

```powershell
(Measure-Command { pwsh -NoLogo -Command exit }).TotalMilliseconds             # 프로파일 포함
(Measure-Command { pwsh -NoLogo -NoProfile -Command exit }).TotalMilliseconds  # 프로파일 제외
```

내 환경에서는 **7,440ms 대 639ms**였다. 프로파일이 6.8초를 쓰고 있었다는 뜻이다.

어느 줄이 범인인지는 의심 가는 블록만 떼어 `-NoProfile`로 돌려 보면 나온다.

```powershell
(Measure-Command {
  pwsh -NoLogo -NoProfile -Command 'oh-my-posh init pwsh | Invoke-Expression; exit'
}).TotalMilliseconds
```

줄여 나가는 순서는 이렇다.

1. **환경변수 등록을 전부 들어낸다.** 위에서 본 대로 가장 비싸고, 애초에 프로파일에 있을 이유가 없다
2. **쓰지 않는 모듈 import를 지운다.** `Import-Module`은 대부분 수백 ms씩 먹는다
3. **가끔 쓰는 기능은 지연 로딩한다.** 함수 안으로 밀어 넣으면 호출할 때까지 비용이 발생하지 않는다

```powershell
# 매번 import 하지 않고, 처음 부를 때 한 번만
Function kubectl-complete {
  if (-not (Get-Module PSKubectlCompletion)) { Import-Module PSKubectlCompletion }
  ...
}
```

함수와 별칭 **정의** 자체는 거의 공짜다. 수십 개를 넣어도 문제가 되지 않는다. 비용은 거의 전부 **프로파일이 실행하는 명령**에서 나온다. 정의는 남기고 실행을 덜어내면 된다.

## 유틸리티 함수

리눅스 습관대로 쓰거나 자주 확인하는 정보는 함수로 정리해뒀다. 아래는 전부 정의만 하므로 시작 시간에 영향이 없다.

```powershell
Function ver { Write-Output $PSVersionTable }                  # 버전 확인
Function export { Get-ChildItem -Path Env:\ }                  # 환경변수 목록
Function Get-IPinfo { ipconfig | Select-String IPv4 }          # 내부 IP
Function Get-MyIPinfo { curl.exe "http://api.ipify.org?format=json" }  # 공인 IP
Function uptime { Get-CimInstance Win32_OperatingSystem |
  Select-Object LastBootUpTime }
Function Get-Title { Get-Process | Where-Object {$_.mainWindowTitle} |
  Format-Table Id, Name, mainWindowTitle -AutoSize }           # 창 있는 프로세스
Function Get-WiFi-Profile { netsh wlan show profile }          # 저장된 Wi-Fi 목록
```

두 군데를 손봤다.

`curl`은 PowerShell 7에서 별칭이 아니므로 `curl.exe`라고 명시하는 편이 안전하다. 별칭이 없으니 `curl`만 써도 실행 파일이 잡히지만, 5.1에서 돌릴 때를 생각하면 확장자를 붙여두는 쪽이 낫다.

`Get-WmiObject`는 `Get-CimInstance`로 바꿨다. WMI 계열 cmdlet은 PowerShell 6에서 빠지고 CIM cmdlet으로 대체됐다. 환경에 따라 호환용 함수가 남아 있어 그대로 동작하기도 하는데(내 PC에서도 `Microsoft.PowerShell.Management`의 함수로 잡혔다), 기대지 않는 편이 안전하다. CIM 쪽은 반환값도 더 깔끔해서 `LastBootUpTime`이 변환 없이 바로 `DateTime`으로 나온다 — 원래 글에 있던 `ConvertToDateTime` 변환이 통째로 필요 없어진다.

SSH 역터널처럼 긴 명령도 함수로 만들어두면 한 단어로 끝난다.

```powershell
Function oc01 { ssh -R 53022:localhost:22 oc01 }
```

## 적용과 문제 해결

프로파일을 수정한 뒤에는 새 세션을 열거나 다시 로드한다.

```powershell
. $PROFILE
```

앞의 점과 공백이 중요하다. 그냥 `$PROFILE`이라고 치면 경로 문자열만 출력된다. 점은 **현재 세션에 불러온다**는 뜻이고, 이게 빠지면 자식 스코프에서 실행돼 별칭과 함수가 사라진다.

프로파일이 아예 실행되지 않는다면 실행 정책을 확인한다.

```powershell
❯ Get-ExecutionPolicy
Restricted          # 이러면 프로파일도 막힌다

Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

`RemoteSigned`는 내가 만든 로컬 스크립트는 허용하고 인터넷에서 받은 것은 서명을 요구한다. 개인 PC에서 쓸 만한 기본값이다.

## 참고

- [about_Profiles - PowerShell 공식 문서](https://learn.microsoft.com/powershell/module/microsoft.powershell.core/about/about_profiles){:target="_blank"}
- [SecretManagement 개요](https://learn.microsoft.com/powershell/utility-modules/secretmanagement/overview){:target="_blank"}
- [oh-my-posh](https://ohmyposh.dev/){:target="_blank"}
- [PowerToys Command Not Found](https://learn.microsoft.com/windows/powertoys/cmd-not-found){:target="_blank"}
- 관련 글: [Windows PowerShell 시작하기]({{site.baseurl}}/dev/2020/09/11/WindowsPowerShell.html){:target="_blank"} · [PowerShell 네트워크 명령어]({{site.baseurl}}/dev/2020/10/14/WindowsPowerShell_network.html){:target="_blank"}
