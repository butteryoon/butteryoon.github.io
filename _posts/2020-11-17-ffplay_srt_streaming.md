---
layout: post
title: "ffmpeg을 이용한 SRT 영상 스트리밍 테스트"
description: "Windows에서 ffmpeg로 mp4파일을 실시간으로 읽고 SRT 프로토콜을 이용해서 OBS로 전송해본다."
img: m_ffmpeg_title.webp
date: 2020-11-17 14:00:00 +0900
last_modified_at: 2020-11-21 17:00:01 +0900
tags: [ffmpeg, ffplay, srt, mpegts] # add tag
related: ffmpeg
categories: dev
---

Windows10 환경에서 ffmpeg을 이용해서 mp4 파일을 실시간으로 읽고 [SRT](https://www.aetherit.io/srt-secure-reliable-transport) 프로토콜로 실시간 전송하는 방법을 알아본다.
<!--more-->

SRT 서버모드는 "OBS Studio"에서 입력소스를 "SRT LISTEN모드"로 설정 하거나 ffplay의 Listen 모드를 이용할 수 있다. 


## OBS 입력 소스 SRT 설정 

OBS Studio의 소스목록에서 + 버튼을 클릭하여 "미디어 소스"를 선택 후 아래의 이미지와 같이 입력, 입력형식을 설정한다. 

> 입력 : srt://127.0.0.1:8888?mode=listener  
> 입력 형식 : mpegts

![OBS 소스 설정]({{site.baseurl}}/assets/img/m_obs_source_srt.webp)

## ffplay LISTEN 모드로 설정 

"ffplay"의 Listerner를 이용하여 SRT 서버모드로 구동한다. 

> ffplay 빌드 정보에서 "--enable-libsrt" 옵션이 포함되어 있어야 한다. 

```powershell
❯ ffplay srt://localhost:8888?mode=listener -x 1024 -y 720

ffplay version 4.3.1-2020-10-01-full_build-www.gyan.dev Copyright (c) 2003-2020 the FFmpeg developers
  built with gcc 10.2.0 (Rev3, Built by MSYS2 project)
  . . . (skip build information)
Input #0, mpegts, from 'srt://localhost:8888?mode=listener':f=0/0
  Duration: N/A, start: 1.400000, bitrate: N/A
  Program 1
    Metadata:
      service_name    : Service01
      service_provider: FFmpeg
    Stream #0:0[0x100]: Video: h264 (High) ([27][0][0][0] / 0x001B), yuv420p(progressive), 1280x720 [SAR 1:1 DAR 16:9], 24 fps, 24 tbr, 90k tbn, 48 tbc
    Stream #0:1[0x101](und): Audio: aac (LC) ([15][0][0][0] / 0x000F), 44100 Hz, stereo, fltp, 133 kb/s
   3.36 A-V:  0.004 fd=   0 aq=   21KB vq=  341KB sq=    0B f=0/0
```

## SRT 서버 LISTEN 확인 

SRT 입력소스가 정상으로 적용되면 로컬 호스트에서 LISTEN 포트를 확인할 수 있다. 

```powershell
❯ netstat -an | grep :8888
  UDP    127.0.0.1:8888      *:*
```

## MPEGTS SRT 스트림 전송

"ffmpeg"으로 인코딩된 영상 파일을 입력(-i)받아 실시간(-re)으로 읽고 영상,음성 코덱은 유지하고(-c copy) SRT 프로토콜로 전송을 위해 영상포맷을 변경(-f mpegts)해서 SRT 서버로 전송한다. 

> ffmpeg 빌드 정보에서 "--enable-libsrt" 옵션이 포함되어 있어야 한다. 

```powershell
❯ ffmpeg.exe -re -i .\BigBuckBunny.mp4 -c copy -f mpegts srt://localhost:8888?pkt_size=1316

ffmpeg version 4.3.1-2020-10-01-full_build-www.gyan.dev Copyright (c) 2000-2020 the FFmpeg developers
  built with gcc 10.2.0 (Rev3, Built by MSYS2 project)
  . . . (skip build information)
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from '.\BigBuckBunny.mp4':
  Metadata:
    major_brand     : mp42
    minor_version   : 0
    compatible_brands: isomavc1mp42
    creation_time   : 2010-01-10T08:29:06.000000Z
  Duration: 00:09:56.47, start: 0.000000, bitrate: 2119 kb/s
    Stream #0:0(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, stereo, fltp, 125 kb/s (default)
    Metadata:
      creation_time   : 2010-01-10T08:29:06.000000Z
      handler_name    : (C) 2007 Google Inc. v08.13.2007.
    Stream #0:1(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 1280x720 [SAR 1:1 DAR 16:9], 1991 kb/s, 24 fps, 24 tbr, 24k tbn, 48 tbc (default)
    Metadata:
      creation_time   : 2010-01-10T08:29:06.000000Z
      handler_name    : (C) 2007 Google Inc. v08.13.2007.
Output #0, mpegts, to 'srt://127.0.0.1:8888':
  Metadata:
    major_brand     : mp42
    minor_version   : 0
    compatible_brands: isomavc1mp42
    encoder         : Lavf58.45.100
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 1280x720 [SAR 1:1 DAR 16:9], q=2-31, 1991 kb/s, 24 fps, 24 tbr, 90k tbn, 24k tbc (default)
    Metadata:
      creation_time   : 2010-01-10T08:29:06.000000Z
      handler_name    : (C) 2007 Google Inc. v08.13.2007.
    Stream #0:1(und): Audio: aac (LC) (mp4a / 0x6134706D), 44100 Hz, stereo, fltp, 125 kb/s (default)
    Metadata:
      creation_time   : 2010-01-10T08:29:06.000000Z
      handler_name    : (C) 2007 Google Inc. v08.13.2007.
Stream mapping:
  Stream #0:1 -> #0:0 (copy)
  Stream #0:0 -> #0:1 (copy)
Press [q] to stop, [?] for help
frame= 1238 fps= 24 q=-1.0 size=   14311kB time=00:00:52.05 bitrate=2251.9kbits/s speed=   1x
```

영상 스트림이 전송되면 ffplay 또는 OBS에서 영상 스트림이 재생되며 OBS의 경우 SRT 미디어 소스가 열리지 않는 경우 설정을 OFF 후 ON 하면 다시 동작한다. 


## 참고 URL
- [FFmpeg SRT protocol information](https://ffmpeg.org/ffmpeg-protocols.html#srt)
- [Streaming-With-SRT-Protocol](https://obsproject.com/wiki/Streaming-With-SRT-Protocol)
- [SRT Cookbook](https://srtlab.github.io/srt-cookbook/apps/ffmpeg/)
- [Using ffmpeg and SRT to Transport Video Signal to the Cloud](https://medium.com/@eyevinntechnology/using-ffmpeg-and-srt-to-transport-video-signal-to-the-cloud-7160960f846a)
- [srt-live-transmit](https://github.com/Haivision/srt/blob/master/docs/srt-live-transmit.md)