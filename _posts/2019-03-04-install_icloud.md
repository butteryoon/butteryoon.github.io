---
layout: post
title: "Windows 에서 iCloud 설치 오류"
img: icloud.png
date: 2019-03-04 17:00:00 +0900
tags: [iCloud, 아이클라우드] # add tag
categories: apple
last_modified_at: 2026-07-15 15:40:00 +0900
---

Windows에 iCloud를 설치하다 만난 Media Player 관련 오류와 해결 과정을 정리한다.

<!--more-->

> **[2026-07-15 업데이트]** 현재 Windows용 iCloud는 Microsoft Store를 통해서만 설치할 수 있다. 이 글에서 다룬 독립 설치 파일(standalone installer)의 Media Player 오류는 더 이상 발생하지 않는다. [Microsoft Store의 iCloud](https://apps.microsoft.com/detail/9pktq5699m62)에서 설치하면 된다. 아래 내용은 당시 기록으로 남겨둔다.

## Windows용 iCloud를 설치하려면 Media Player가 필요함. 

![iCloud Error]({{site.baseurl}}/assets/img/iclouderr.png)

Windows에 iCloud를 설치하려는데 위와 같은 오류가 계속 발생한다. 

최근 Windows에 포함되어 있는 Media Player를 쓰지 않아 Windows Media 기능 자체를 OFF 하고 외부 플레이어를 설치해서 쓰고 있는데 [apple 지원 사이트](https://support.apple.com/ko-kr/HT204363)를 보면 Media Play 기능이 있어야 한다고 한다. 

[Windows 10 N 및 KN 버전을 위한 미디어 기능 팩](https://www.microsoft.com/ko-kr/download/details.aspx?id=49919) 에서 관련 패키지를 설치한 후 iCloud를 설치하여 성공 ..  

미디어 기능 팩 설치 후 "Windows 기능 켜기/끄기"에서 Windows Media Player를 활성화 해야 한다.  

![Windows 기능 켜기/끄기]({{site.baseurl}}/assets/img/Windows_onoff.png)
