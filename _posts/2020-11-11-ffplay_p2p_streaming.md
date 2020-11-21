---
layout: post
title: "ffmpeg을 이용한 Point to Point 스트리밍"
description: "ffmpeg을 이용한 Point to Point 스트리밍"
img: m_ffmpeg_title.webp
date: 2020-11-11 10:00:00 +0900
last_modified_at: 2020-11-21 18:00:01 +0900
tags: [ffmpeg, ffplay, rtsp, rtmp] # add tag
related: ffmpeg
categories: dev
---

특정 사용자간 실시간 영상 스트리밍 전송 기능을 시험하기 위해 수신 PC에서 ffplay로 서버를 구성하고 전송 PC에서 ffmpeg을 이용하여 실시간 영상을 스트리밍하는 방법을 알아본다. 


## ffplay RTSP LISTEN 모드로 실행. (Server Mode)  

ffplay를 "RTSP LISTEN" 모드로 실행하고 영상이 인입되면 1024x720 해상도로 변경하여 재생한다. 

```powershell
❯ ffplay -rtsp_flags listen rtsp://localhost:8888/live.sdp?tcp -x 1024 -y 720 

[rtsp @ 00000230e3a59c00] WARNING: Path /live.sdp differs from expected /live.sdp?tcp
    Last message repeated 1 times
[rtsp @ 00000230e3a59c00] Updating control URI to rtsp://localhost:8888/live.sdp
Input #0, rtsp, from 'rtsp://localhost:8888/live.sdp?tcp':B f=0/0
  Metadata:
    title           : No Name
  Duration: N/A, start: -0.470998, bitrate: N/A
    Stream #0:0: Video: h264 (High), yuv420p(progressive), 1280x720 [SAR 1:1 DAR 16:9], 24 fps, 24 tbr, 90k tbn, 48 tbc
    Stream #0:1: Audio: aac (LC), 44100 Hz, stereo, fltp
   3.83 A-V:  0.000 fd=   0 aq=   19KB vq=  282KB sq=    0B f=1/1
```

## ffmpeg RTSP Restream (Client Mode)

ffplay를 RTSP Server모드로 구동한 후 ffmpeg로 인코딩된 mp4 파일을 열어 8888 포트로 전송한다. 

```powershell
❯ ffmpeg.exe -re -i .\BigBuckBunny.mp4 -c copy -f rtsp -rtsp_transport tcp rtsp://localhost:8888/live.sdp

ffmpeg version 4.3.1-2020-10-01-full_build-www.gyan.dev Copyright (c) 2000-2020 the FFmpeg developers
  built with gcc 10.2.0 (Rev3, Built by MSYS2 project)
  . . . 컴파일정보 생략
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
Output #0, rtsp, to 'rtsp://localhost:8888/live.sdp':
  Metadata:
    major_brand     : mp42
    minor_version   : 0
    compatible_brands: isomavc1mp42
    encoder         : Lavf58.45.100
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 1280x720 [SAR 1:1 DAR 16:9], q=2-31, 1991 kb/s, 24 fps, 24 tbr, 90k tbn, 24 tbc (default)
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
av_interleaved_write_frame(): Broken pipe00:00:06.15 bitrate=N/A speed=1.01x
Error writing trailer of rtsp://localhost:8888/live.sdp: Broken pipe
frame=  148 fps= 23 q=-1.0 Lsize=N/A time=00:00:06.61 bitrate=N/A speed=1.01x
video:1834kB audio:102kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: unknown
```

## RTSP 트랜잭션 

ffplay에서 유입되는 RTSP 트랜잭션을 보면 아래와 같다. 

```powershell
OPTIONS rtsp://192.168.0.19:8888/live.sdp RTSP/1.0
CSeq: 1
User-Agent: Lavf58.45.100

RTSP/1.0 200 OK
CSeq: 1
Server: Lavf58.45.100
Public: ANNOUNCE, PAUSE, SETUP, TEARDOWN, RECORD

ANNOUNCE rtsp://192.168.0.19:8888/live.sdp RTSP/1.0
Content-Type: application/sdp
CSeq: 2
User-Agent: Lavf58.45.100
Content-Length: 498

v=0
o=- 0 0 IN IP4 127.0.0.1
s=No Name
c=IN IP4 192.168.0.19
t=0 0
a=tool:libavformat 58.45.100
m=video 0 RTP/AVP 96
b=AS:1991
a=rtpmap:96 H264/90000
a=fmtp:96 packetization-mode=1; sprop-parameter-sets=Z2QAH6wkhAFAFuwEQAAAAwBAAAAMI8YMkg==,aO4yyLA=; profile-level-id=64001F
a=control:streamid=0
m=audio 0 RTP/AVP 97
b=AS:125
a=rtpmap:97 MPEG4-GENERIC/44100/2
a=fmtp:97 profile-level-id=1;mode=AAC-hbr;sizelength=13;indexlength=3;indexdeltalength=3; config=1210
a=control:streamid=1
RTSP/1.0 200 OK
CSeq: 2
Server: Lavf58.45.100

SETUP rtsp://192.168.0.19:8888/live.sdp/streamid=0 RTSP/1.0
Transport: RTP/AVP/TCP;unicast;interleaved=0-1;mode=record
CSeq: 3
User-Agent: Lavf58.45.100

RTSP/1.0 200 OK
CSeq: 3
Server: Lavf58.45.100
Transport: RTP/AVP/TCP;unicast;mode=receive;interleaved=0-1
Session: 1455181266

SETUP rtsp://192.168.0.19:8888/live.sdp/streamid=1 RTSP/1.0
Transport: RTP/AVP/TCP;unicast;interleaved=2-3;mode=record
CSeq: 4
User-Agent: Lavf58.45.100
Session: 1455181266

RTSP/1.0 200 OK
CSeq: 4
Server: Lavf58.45.100
Transport: RTP/AVP/TCP;unicast;mode=receive;interleaved=2-3
Session: 1455181266

RECORD rtsp://192.168.0.19:8888/live.sdp RTSP/1.0
Range: npt=0.000-
CSeq: 5
User-Agent: Lavf58.45.100
Session: 1455181266

RTSP/1.0 200 OK
CSeq: 5
Server: Lavf58.45.100
Session: 1455181266

. . . (skip)
PT=DynamicRTP-Type-96, SSRC=0x73150015, Seq=2784, Time=40284601, Mark
. . . (skip)

TEARDOWN rtsp://192.168.0.19:8888/live.sdp RTSP/1.0
CSeq: 6
User-Agent: Lavf58.45.100
Session: 1455181266
```

## 참고 URL
- [ffmpeg StreamingGuide](https://trac.ffmpeg.org/wiki/StreamingGuide)
- [ffplay Documentation](https://ffmpeg.org/ffplay.html) 
- [ffmpeg Windows Builds 다운로드](https://www.gyan.dev/ffmpeg/builds/)
- [FFmpeg for streaming](https://sonnati.wordpress.com/2011/08/30/ffmpeg-–-the-swiss-army-knife-of-internet-streaming-–-part-iv/)
