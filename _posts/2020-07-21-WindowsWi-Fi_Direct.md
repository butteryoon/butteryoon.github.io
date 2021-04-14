---
layout: post
title: "Windows 10 Wi-Fi Direct"
description: "윈도우즈10 에서 Wi-Fi 다이렉트 기능 확인."
img: WiFi-Direct-Cover.png
date: 2020-07-21 18:24:00 +0900
last_modified_at: 2021-04-14 22:00:00 +0900
tags: [Windows10, Wi-Fi, Direct, wireless, hotspot] # add tag
related: Windows10
categories: dev
---

## Wi-Fi Direct 카메라 

Windows 10 에서 Wi-Fi Direct 기능 설정을 위한 숟가락질 ..

안드로이드의 카메라에서 실시간 촬영한 영상을 Wi-Fi Direct 기능을 이용해서 Windows 10으로 전송하고 카메라앱에서 실시간으로 영상 재생을 하려 하는데 .. 

## Windows10 Wi-Fi Direct 설정 확인

설정 > 네트워크 및 인터넷 > Wi-Fi 설정에 Wi-Fi Direct 설정이 보이지 않는다. 
Windows의 IP 설정에는 Adapter가 보이는데. 

```powershell
❯ ipconfig /all 

무선 LAN 어댑터 로컬 영역 연결* 2:

   미디어 상태 . . . . . . . . : 미디어 연결 끊김
   연결별 DNS 접미사. . . . :
   설명. . . . . . . . . . . . : Microsoft Wi-Fi Direct Virtual Adapter
   물리적 주소 . . . . . . . . : 76-DF-BF-69-F5-19
   DHCP 사용 . . . . . . . . . : 예
   자동 구성 사용. . . . . . . : 예
```

## Windows wireless hotspot 

잠깐 방향을 틀어 hostednetwork 설정은 해보기로 한다. 

hostednetwork 설정은 제대로 되는데 아래처럼 시작이 안된다.  

```powershell
> netsh wlan show hostednetwork

호스트된 네트워크 설정
-----------------------
모드                   : 허용
SSID 이름              : "WIFI-D"
최대 클라이언트 수      : 100
인증                   : WPA2-개인
암호                   : CCMP

호스트된 네트워크 상태
---------------------
상태                   : 사용할 수 없음 

❯ netsh wlan start hostednetwork
호스트된 네트워크를 시작할 수 없습니다.
그룹 또는 리소스가 요청된 작업을 실행할 올바른 상태에 있지 않습니다.
```

무선랜 드라이버 정보를 보니 "호스트된 네트워크 지원 : 아니요" 라고 나온다.    
드라이버 문제인지 하드웨어 문제인지 모르겠다.  

> netsh wlan show drivers 결과 

![netsh wlan show drivers]({{site.bashurl}}/assets/img/M_hostednetwork.jpg)

Windows 10 에서 기본 제공하는 모바일 핫스팟 설정은 잘 되는 듯 . 

![Mobile HotSpot]({{site.bashurl}}/assets/img/M_W10_mobilehotspot.jpg)

## Feem 

Feem 설치 후 Wi-Fi Direct On으로 하니 Wi-Fi Direct 인터페이스 설정이 보인다.  
뭔가 더 찾아봐야 할 듯.. 

[Feem Download](https://www.feem.io/#download)

## 참고 URL
- [Windows 10 into a wireless hotspot](https://www.windowscentral.com/how-turn-your-windows-10-pc-wireless-hotspot)
- [WiFi Driect for Windows 10](https://www.windowsinside.com/wifi-direct-windows-10)