---
layout: post
title: "ffmpeg을 이용한 PC화면 유튜브 실시간 스트리밍 테스트"
description: "Windows10에서 ffmpeg을 이용하여 PC화면을 캡쳐하고 실시간으로 유듀브로 라이브 스트리밍하는 방법을 알아본다."
img: m_ffmpeg_title.webp
date: 2021-01-06 13:00:00 +0900
last_modified_at: 2021-01-06 13:00:01 +0900
tags: [ffmpeg, ffplay, gdigrab, directshow, youtube] # add tag
related: ffmpeg
categories: dev
---

Windows10 OS에서 GDI Grabber를 이용해서 화면을 실시간으로 캡쳐하고 ffmpeg로 유튜브 라이브 스트리밍하는 방법을 알아본다.  

<!--more-->

## 노트북 화면 캡쳐 라이브 스트리밍 

Windows10의 GDI ScreenGrabber를 이용하고 인코딩은 nVidia의 NVENC를 이용해서 영상을 생성한다. 

유튜브 라이브 스트리밍은 오디오가 필요하여 DirectShow로 오디오를 추가한다. 

> -f gdigrab -framerate 30 -video_size 1920x1080 -i desktop  
> -f dshow -i audio="마이크 배열(Realtek High Definition Audio)"  
> -c:v h264_nvenc   
> -preset medium -r 30 -g 60 -b:v 4500k  
> -f flv rtmp://a.rtmp.youtube.com/live2/stream-key  

```powershell
❯ ffmpeg -f gdigrab -framerate 30 -video_size 1920x1080 -i desktop \
-f dshow -i audio="마이크 배열(Realtek High Definition Audio)" -deinterlace \
-vcodec h264_nvenc -qp 0 -pix_fmt yuv420p -preset medium -r 30 -g 60 -b:v 4500k \
-acodec libmp3lame -ar 44100 -threads 6 -qscale 3 -b:a 712000 -bufsize 512k \
-f flv rtmp://a.rtmp.youtube.com/live2/stream-key
```

## Display 선택

노트북에 확장 모니터를 사용하고 있는 경우 gdigrab의 "-i desktop" 영상 소스인 경우 기본이 확장 모니터를 포함한 전체 디스플레이 화면이 캡쳐되는데 특정 화면만 캡쳐할 수있는 방법이 없는거 같다(아니 못 찾았다.)

대안으로 특정 프로그램의 Windows 어플리케이션 화면을 캡쳐하는 "-i title=calculator" 옵션을 사용하면 빈 화면만 캡쳐된다. (왜 그럴까?)

옵션을 변경 하다가 유튜브 라이브 스트리밍 스트림상태에서 알려주는 해상도 및 비트레이트 권장사양을 맞추다 보니 아래와 같이 비디오사이즈 옵션을 1920x1080으로 설정하니 주 디스플레이 화면만 캡쳐된다. (왜 그럴까?)

```
-f gdigrab -framerate 30 -video_size 1920x1080 -i desktop
```

## 더 찾아볼것.. 

- gdigrab 에서 특정 디스플레이를 선택하는 방법
- gdigrab 에서 -i title="Windows_Title" 으로 특정 어플리케이션 화면을 캡쳐할 때 빈 환면으로 보이는 현상. 

## 참고 URL
- [Capturing your Desktop / Screen Recording](https://trac.ffmpeg.org/wiki/Capture/Desktop)
- [LIST OF AVAILABLE DIRECTSHOW SCREEN CAPTURE FILTERS](http://betterlogic.com/roger/2010/07/list-of-available-directshow-screen-capture-filters/)
- [network-stream-videoaudio-ffmpeg](https://superuser.com/questions/1152825/network-stream-videoaudio-ffmpeg)