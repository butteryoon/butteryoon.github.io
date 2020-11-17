---
layout: post
title: "ffmpeg을 이용한 SRT 영상 스트리밍 테스트"
description: "Windows에서 ffmpeg로 mp4파일을 실시간으로 읽고 SRT 프로토콜을 이용해서 OBS로 전송해본다."
img: ffmpeg.png
date: 2020-11-17 14:00:00 +0900
last_modified_at: 2020-11-17 14:00:01 +0900
tags: [ffmpeg, ffplay, srt] # add tag
related: ffmpeg
categories: dev
---

Windows10 환경에서 ffmpeg을 이용해서 mp4 파일을 실시간으로 읽고 [SRT](https://www.aetherit.io/srt-secure-reliable-transport) 프로토콜로 실시간 전송하는 방법을 알아본다.
<!--more-->

OBS Studio에서 입력소스를 SRC로 설정하고 LISTEN모드로 설정한 후 ffmpeg으로 등록한 SRT 입력소스로 스트립을 전송한다. 

## OBS 입력 소스 설정 

OBS Studio의 소스목록에서 + 버튼을 클릭하여 "미디어 소스"를 선택 후 아래의 이미지와 같이 입력, 입력형식을 설정한다. 

> 입력 : srt://127.0.0.1:8888?mode=listener  
> 입력 형식 : mpegts

SRT 입력소스가 정상으로 적용되면 로컬 호스트에서 LISTEN 포트를 확인할 수 있다. 

```powershell
❯ netstat -an | grep :8888
  UDP    127.0.0.1:8888      *:*
```

![OBS 소스 설정]({{site.baseurl}}/assets/img/m_obs_source_srt.webp)


## MPEGTS SRT 전송

ffmpeg으로 인코딩된 영상 파일을 입력(-i)받아 실시간(-re)으로 읽고 영상,음성 코덱은 유지하고(-c copy) SRT 전송을 위해 영상포맷을 변경(-f mpegts)해서 SRT 서버로 전송한다. 

> ffmpeg 빌드 정보에서 "--enable-libsrt" 옵션이 포함되어 있어야 한다. 

```powershell
❯ ffmpeg.exe -re -i .\BigBuckBunny.mp4 -c copy -f mpegts srt://127.0.0.1:8888
ffmpeg version 4.3.1-2020-10-01-full_build-www.gyan.dev Copyright (c) 2000-2020 the FFmpeg developers
  built with gcc 10.2.0 (Rev3, Built by MSYS2 project)
  configuration: --enable-gpl --enable-version3 --enable-static --disable-w32threads --disable-autodetect --enable-fontconfig --enable-iconv --enable-gnutls --enable-libxml2 --enable-gmp --enable-lzma --enable-libsnappy --enable-zlib --enable-libsrt --enable-libssh --enable-libzmq --enable-avisynth --enable-libbluray --enable-libcaca --enable-sdl2 --enable-libdav1d --enable-libzvbi --enable-librav1e --enable-libwebp --enable-libx264 --enable-libx265 --enable-libxvid --enable-libaom --enable-libopenjpeg --enable-libvpx --enable-libass --enable-frei0r --enable-libfreetype --enable-libfribidi --enable-libvidstab --enable-libvmaf --enable-libzimg --enable-amf --enable-cuda-llvm --enable-cuvid --enable-ffnvcodec --enable-nvdec --enable-nvenc --enable-d3d11va --enable-dxva2 --enable-libmfx --enable-libcdio --enable-libgme --enable-libmodplug --enable-libopenmpt --enable-libopencore-amrwb --enable-libmp3lame --enable-libshine --enable-libtheora --enable-libtwolame --enable-libvo-amrwbenc --enable-libilbc --enable-libgsm --enable-libopencore-amrnb --enable-libopus --enable-libspeex --enable-libvorbis --enable-ladspa --enable-libbs2b --enable-libflite --enable-libmysofa --enable-librubberband --enable-libsoxr --enable-chromaprint
  libavutil      56. 51.100 / 56. 51.100
  libavcodec     58. 91.100 / 58. 91.100
  libavformat    58. 45.100 / 58. 45.100
  libavdevice    58. 10.100 / 58. 10.100
  libavfilter     7. 85.100 /  7. 85.100
  libswscale      5.  7.100 /  5.  7.100
  libswresample   3.  7.100 /  3.  7.100
  libpostproc    55.  7.100 / 55.  7.100
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


## 참고 URL
- [Using ffmpeg and SRT to Transport Video Signal to the Cloud](https://medium.com/@eyevinntechnology/using-ffmpeg-and-srt-to-transport-video-signal-to-the-cloud-7160960f846a)
- [SRT for FFmpeg](https://srtlab.github.io/srt-cookbook/apps/ffmpeg/)