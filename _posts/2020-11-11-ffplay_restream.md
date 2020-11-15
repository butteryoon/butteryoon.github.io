---
layout: post
title: "ffmpeg을 이용한 RTMP 영상 스트리밍"
description: "Windows에서 ffmpeg을 이용해서 mp4파일을 실시간으로 읽고 트랜스코딩 해서 RTMP로 실시간 전송하는 방법을 알아본다."
img: ffmpeg.png
date: 2020-11-11 09:00:00 +0900
last_modified_at: 2020-11-11 09:00:01 +0900
tags: [ffmpeg, ffplay, rtsp, rtmp] # add tag
related: ffmpeg
categories: dev
---

Windows10 환경에서 ffmpeg을 이용해서 mp4 파일을 실시간으로 읽고 트랜스코딩 해서 RTMP로 실시간 전송하는 방법을 알아본다.

RTMP 서버는 [Ant Media Server](https://ant-media-docs.readthedocs.io/en/latest/Getting-Started.html)를 이용할 수 있고 OBS를 이용하여 RTMP전송을 시험할 수도 있다. 

## 인코딩된 영상 파일 스트리밍

인코딩된 영상 파일을 입력(-i)받아 실시간(-re)으로 읽고 영상, 응성 코덱은 유지하고(-c copy) RTMP 전송을 위해 영상포맷을 변경(-f flv)해서 RTMP 서버로 전송한다. 

```powershell
❯ ffmpeg.exe -re -i .\BigBuckBunny.mp4 -c copy -f flv rtmp://192.168.0.92:1955/application/stream_key

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
Output #0, flv, to 'rtmp://192.168.0.92:1955/application/stream':
  Metadata:
    major_brand     : mp42
    minor_version   : 0
    compatible_brands: isomavc1mp42
    encoder         : Lavf58.45.100
    Stream #0:0(und): Video: h264 (High) ([7][0][0][0] / 0x0007), yuv420p, 1280x720 [SAR 1:1 DAR 16:9], q=2-31, 1991 kb/s, 24 fps, 24 tbr, 1k tbn, 24k tbc (default)
    Metadata:
      creation_time   : 2010-01-10T08:29:06.000000Z
      handler_name    : (C) 2007 Google Inc. v08.13.2007.
    Stream #0:1(und): Audio: aac (LC) ([10][0][0][0] / 0x000A), 44100 Hz, stereo, fltp, 125 kb/s (default)
    Metadata:
      creation_time   : 2010-01-10T08:29:06.000000Z
      handler_name    : (C) 2007 Google Inc. v08.13.2007.
Stream mapping:
  Stream #0:1 -> #0:0 (copy)
  Stream #0:0 -> #0:1 (copy)
Press [q] to stop, [?] for help
av_interleaved_write_frame(): Unknown errortime=00:04:09.52 bitrate=2184.7kbits/s speed=   1x
[flv @ 000001d65a9bc0c0] Failed to update header with correct duration.
[flv @ 000001d65a9bc0c0] Failed to update header with correct filesize.
Error writing trailer of rtmp://192.168.0.92:1955/application/stream: Error number -10053 occurred
frame= 5984 fps= 24 q=-1.0 Lsize=   66544kB time=00:04:09.77 bitrate=2182.5kbits/s speed=   1x
video:62435kB audio:3829kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: 0.423518%
```

## RTSP 영상스트림 실시간 RTMP 전송

RTSP 영상스트림을 입력(-i)받아 실시간(-re)으로 읽고 음성코덱은 AAC(-c:a aac), 영상코덱은 h264(-c:v h264)로 트랜스코딩 하고 RTMP 전송을 위해 영상포맷을 변경(-f flv)해서 RTMP 서버로 전송한다. 

RTMP 전송 프로토콜에서 사용하는 flv 미디어포맷은 HEVC(h.265) 코덱을 지원하지 않아 AVC(h.264) 코덱으로 변경한다. 

```powershell
❯ ffmpeg.exe -re -i rtsp://iws.localdomain.com:50554/main -c:a aac -c:v h264 -f flv rtmp://192.168.0.92:1955/application/stream

[rtsp @ 0000028c8198dcc0] method SETUP failed: 461 Unsupported Transport
Guessed Channel Layout for Input Stream #0.1 : mono
Input #0, rtsp, from 'rtsp://iws.localdomain.com:50554/main':
  Metadata:
    title           : Media Presentation
  Duration: N/A, start: 0.000000, bitrate: N/A
    Stream #0:0: Video: hevc (Main), yuvj420p(pc, bt709), 1920x1080, 25 tbr, 90k tbn, 90k tbc
    Stream #0:1: Audio: pcm_alaw, 8000 Hz, mono, s16, 64 kb/s
Stream mapping:
  Stream #0:0 -> #0:0 (hevc (native) -> h264 (libx264))
  Stream #0:1 -> #0:1 (pcm_alaw (native) -> aac (native))
Press [q] to stop, [?] for help
[aac @ 0000028c81a50540] Too many bits 8832.000000 > 6144 per frame requested, clamping to max
[libx264 @ 0000028c81e50780] using cpu capabilities: MMX2 SSE2Fast SSSE3 SSE4.2 AVX FMA3 BMI2 AVX2
[libx264 @ 0000028c81e50780] profile High, level 4.0, 4:2:0, 8-bit
[libx264 @ 0000028c81e50780] 264 - core 161 r3018 db0d417 - H.264/MPEG-4 AVC codec - Copyleft 2003-2020 - http://www.videolan.org/x264.html - options: cabac=1 ref=3 deblock=1:0:0 analyse=0x3:0x113 me=hex subme=7 psy=1 psy_rd=1.00:0.00 mixed_ref=1 me_range=16 chroma_me=1 trellis=1 8x8dct=1 cqm=0 deadzone=21,11 fast_pskip=1 chroma_qp_offset=-2 threads=6 lookahead_threads=1 sliced_threads=0 nr=0 decimate=1 interlaced=0 bluray_compat=0 constrained_intra=0 bframes=3 b_pyramid=2 b_adapt=1 b_bias=0 direct=1 weightb=1 open_gop=0 weightp=2 keyint=250 keyint_min=25 scenecut=40 intra_refresh=0 rc_lookahead=40 rc=crf mbtree=1 crf=23.0 qcomp=0.60 qpmin=0 qpmax=69 qpstep=4 ip_ratio=1.40 aq=1:1.00
Output #0, flv, to 'rtmp://192.168.0.92:1955/application/stream':
  Metadata:
    title           : Media Presentation
    encoder         : Lavf58.45.100
    Stream #0:0: Video: h264 (libx264) ([7][0][0][0] / 0x0007), yuvj420p(pc), 1920x1080, q=-1--1, 25 fps, 1k tbn, 25 tbc
    Metadata:
      encoder         : Lavc58.91.100 libx264
    Side data:
      cpb: bitrate max/min/avg: 0/0/0 buffer size: 0 vbv_delay: N/A
    Stream #0:1: Audio: aac (LC) ([10][0][0][0] / 0x000A), 8000 Hz, mono, fltp, 48 kb/s
    Metadata:
      encoder         : Lavc58.91.100 aac
[hevc @ 0000028c81a41cc0] Could not find ref with POC 41.72 bitrate=1109.1kbits/s speed=0.997x
[hevc @ 0000028c81a1da40] Could not find ref with POC 38.07 bitrate=1140.4kbits/s speed=0.999x
[hevc @ 0000028c81a1da40] Could not find ref with POC 39.50 bitrate=1183.1kbits/s speed=0.999x
[hevc @ 0000028c81a52940] Could not find ref with POC 43.01 bitrate=1225.0kbits/s speed=   1x
[hevc @ 0000028c819a1d40] Could not find ref with POC 10.05 bitrate=1227.0kbits/s speed=   1x
frame=35154 fps= 25 q=28.0 size=  209602kB time=00:23:27.23 bitrate=1220.2kbits/s speed=   1x
```

## 참고 URL
- [ffplay Documentation](https://ffmpeg.org/ffplay.html) 
- [ffmpeg Windows Builds 다운로드](https://www.gyan.dev/ffmpeg/builds/)
- [FFmpeg for streaming](https://sonnati.wordpress.com/2011/08/30/ffmpeg-–-the-swiss-army-knife-of-internet-streaming-–-part-iv/)
- [Easily transcode any media to any format using FFmpeg](https://duduf.com/easily-transcode-any-media-to-any-format-using-ffmpeg/)
- [Video Transcoding and Optimization for web with FFmpeg made easy](https://medium.com/abraia/video-transcoding-and-optimization-for-web-with-ffmpeg-made-easy-511635214df0)