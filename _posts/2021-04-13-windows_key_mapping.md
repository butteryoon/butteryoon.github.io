---
layout: post
title: "Windows 10 키 매핑"
description: "윈도우즈 10 에서 PowerToys로 키보드 매핑을 바꾸는 방법"
img: image-title.jpg
date: 2021-04-13 12:00:00 +0900
last_modified_at: 2021-04-13 12:00:00 +0900
tags: [Windows10, powertoy] # add tag
related: Windows10
categories: tools
---

MACOS를 쓰다보면 한/영 전환키가 "CapsLock" 키로 되어 있어 Windows10과 번갈아서 쓰다보면 은근히 헷갈리기도 해서 Windows의 한/영 전환키를 "CapsLock" 키로 바꿔보기로 한다. 

<!--more--> 

[Microsoft PowerToys](https://docs.microsoft.com/en-us/windows/powertoys/)로 기존 키 매핑을 간단하게 바꿀 수 있다. 

[AutoHotKey](https://www.autohotkey.com)를 이용하는 방법도 있는데 키 매핑만 변경하려면 PowerToys가 더 간단하다. 

## MS PowerToy 설치 

[PowerToys](https://docs.microsoft.com/ko-kr/windows/powertoys/install) 페이지에서 실행파일로 설치한다. 

[Chocolatey](https://community.chocolatey.org/packages/powertoys)를 사용하면 아래와 같이 명령어로 간단하게 설치할 수 있다. 

```powershell
choco install powertoys
```

실행하면 오른쪽 트레이에 표시되며 더블클릭으로 설정창을 열 수 있고 "Keyboard Manager" 메뉴에서 "키 다시 매핑" 기능으로 변경한다. 

CapsLk 키를 한/영 전환키로 변경한다. 

![powertoy key mappinng]({{site.baseurl}}/assets/img/m_powertoy_keymapping.webp)


## 참고 URL
- [Windows 10에서 키를 다시 매핑하는 방법](https://genuine-lamps.com/ko/windows/16053-how-to-remap-keys-on-windows-10.html){:target="_blank"}
