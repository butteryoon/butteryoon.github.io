---
layout: post
title: "ffmpeg을 이용한 PC화면 실시간 스트리밍 테스트"
description: "Windows에서 ffmpeg을 이용하여 PC화면을 캡쳐하고 실시간으로 스트리밍하는 방법을 알아본다."
img: m_ffmpeg_title.webp
date: 2020-11-17 13:00:00 +0900
last_modified_at: 2020-11-21 22:00:01 +0900
tags: [ffmpeg, ffplay, gdigrab, directshow] # add tag
related: ffmpeg
categories: dev
---

Windows10 노트북에서 화면을 실시간으로 캡쳐하고 ffmpeg을 이용해서 실시간으로 영상을 스트리밍하는 방법을 알아본다. 
<!--more-->

스크린 캡쳐하는 방법은 "DirectShow"를 이용하는 방법과 "GDI ScreenGrabber"를 이용한 방법이 있다. 

## 영상 재생 설정 

> -probesize 40 
> -x 1280 -y 720

```powershell
❯ ffplay -probesize 40 -x 1280 -y 720 udp://localhost:8888?mode=listener

ffplay version 4.3.1-2020-10-01-full_build-www.gyan.dev Copyright (c) 2003-2020 the FFmpeg developers
  built with gcc 10.2.0 (Rev3, Built by MSYS2 project)
  . . . (skip build informations)
[mpegts @ 000001ff2c3fda80] Stream #0: not enough frames to estimate rate; consider increasing probesize
[mpegts @ 000001ff2c3fda80] decoding for stream 0 failed
Input #0, mpegts, from 'udp://localhost:8888?mode=listener':
  Duration: N/A, start: 1.400000, bitrate: N/A
  Program 1
    Metadata:
      service_name    : Service01
      service_provider: FFmpeg
    Stream #0:0[0x100]: Video: h264 (Main) ([27][0][0][0] / 0x001B), yuv420p(progressive), 1920x1080 [SAR 1:1 DAR 16:9], 30 tbr, 90k tbn, 60 tbc
   7.39 M-V:  0.017 fd=   0 aq=    0KB vq=   50KB sq=    0B f=0/0
```

또는 OBS의 미디어소스를 추가한다.

![OBS 소스 설정]({{site.baseurl}}/assets/img/m_obs_source_udp.webp)


## 노트북 화면 캡쳐 스트리밍 

Windows의 내장 GDI ScreenGrabber를 이용하고 인코딩은 nVidia의 NVENC를 이용해서 영상을 생성한다. 

> -f gdigrab   
> -framerate 30   
> -i desktop   
> -c:v h264_nvenc -c:a aac   
> -f mpegts "udp://localhost:8888"  

```powershell
❯ ffmpeg -f gdigrab -framerate 30 -i desktop -c:v h264_nvenc -c:a aac -f mpegts "udp://localhost:8888"

ffmpeg version 4.3.1-2020-10-01-full_build-www.gyan.dev Copyright (c) 2000-2020 the FFmpeg developers
  built with gcc 10.2.0 (Rev3, Built by MSYS2 project)
  . . . (skip build informations)
[gdigrab @ 000001fe610fe480] Capturing whole desktop as 1920x1080x32 at (0,0)
[gdigrab @ 000001fe610fe480] Stream #0: not enough frames to estimate rate; consider increasing probesize
Input #0, gdigrab, from 'desktop':
  Duration: N/A, start: 1605961680.487105, bitrate: 1990668 kb/s
    Stream #0:0: Video: bmp, bgra, 1920x1080, 1990668 kb/s, 30 fps, 1000k tbr, 1000k tbn, 1000k tbc
Stream mapping:
  Stream #0:0 -> #0:0 (bmp (native) -> h264 (h264_nvenc))
Press [q] to stop, [?] for help
Output #0, mpegts, to 'udp://localhost:8888':
  Metadata:
    encoder         : Lavf58.45.100
    Stream #0:0: Video: h264 (h264_nvenc) (Main), rgb0, 1920x1080, q=-1--1, 2000 kb/s, 30 fps, 90k tbn, 30 tbc
    Metadata:
      encoder         : Lavc58.91.100 h264_nvenc
    Side data:
      cpb: bitrate max/min/avg: 0/0/2000000 buffer size: 4000000 vbv_delay: N/A
frame=  175 fps= 22 q=25.0 Lsize=    1117kB time=00:00:07.80 bitrate=1173.5kbits/s speed=   1x
video:1046kB audio:0kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 6.830150%
```

DirectShow 필터를 이용한 실시간 스트리밍은 화면캡쳐 드라이버가 별도로 필요한거 같아 추후에 다시 알아보자 .. 

> -f dshow   
> -i video="UScreenCapture":audio="마이크 배열(Realtek High Definition Audio)"  
> -c:v h264_nvenc -c:a aac  
> -f mpegts "udp://localhost:8888"  

```
❯ ffmpeg -f dshow -i video="UScreenCapture":audio="마이크 배열(Realtek High Definition Audio)"  -c:v h264_nvenc -c:a aac -f mpegts "udp://localhost:8888"
```

## 참고 URL
- [Capturing your Desktop / Screen Recording](https://trac.ffmpeg.org/wiki/Capture/Desktop)
- [LIST OF AVAILABLE DIRECTSHOW SCREEN CAPTURE FILTERS](http://betterlogic.com/roger/2010/07/list-of-available-directshow-screen-capture-filters/)