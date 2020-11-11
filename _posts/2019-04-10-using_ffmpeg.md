---
layout: post
title: "ffmpeg로 영상 변환하기"
description: "ffmpeg로 영상 변환하기"
img: ffmpeg.png
date: 2019-04-10 17:06:00 +0900
last_modified_at: 2020-11-11 14:00:00 +0900
tags: [ffmpeg, ffprobe, ffplay] # add tag
related: ffmpeg
categories: dev
---

## MP4 영상 스트리밍시 seek 가 안되는 경우 

mp4 영상을 ffprobe로 보면 아래와 같이 start 값이 영상의 길이보다 큰 경우가 있다. 
이런 경우에 seek를 하여 재생위치를 변경하면 초기 위치부터 재생이 된다. 

```powershell
$ ffprobe filename.mp4
ffprobe version N-77791-g93ac72a Copyright (c) 2007-2016 the FFmpeg developers
  built with gcc 4.4.7 (GCC) 20120313 (Red Hat 4.4.7-16)
  . . . (skip build information)
Input #0, mov,mp4,m4a,3gp,3g2,mj2, from 'filename.mp4':
  Metadata:
    major_brand     : isom
    minor_version   : 512
    compatible_brands: isomiso2avc1mp41
    encoder         : Lavf57.21.101
  Duration: 00:21:13.89, start: 2010.203000, bitrate: 2033 kb/s
    Stream #0:0(und): Video: h264 (Baseline) (avc1 / 0x31637661), yuv420p(tv, unknown/bt470bg/unknown), 1280x720, 2031 kb/s, 17.49 fps, 180k tbr, 180k tbn, 360k tbc (default)
    Metadata:
      handler_name    : VideoHandler
```

아래 ffmpeg으로 영상을 재생성하면 start 값이 0 으로 설정된다. 
> 그냥 실행하면 터미널에 로그를 찍느라 시간이 오래 걸리므로 로그는 /dev/null 로 보내버린다. 

```powershell
ffmpeg -i input.mp4 -codec copy output.mp4 > /dev/null 2>&1
```

## 영상 자르기 

CCTV와 같은 계속되는 영상을 1시간 단위로 1일동안 보관하고자 할 때 아래와 같이 segment 기능으로 영상을 잘라서 보관할 수 있다. 

> -f segment : 파일을 분할하여 저장
> -segment_list out.list : 파일목록 기록
> -segment_time 3600 : 분할단위는 초(3600초, 1시간)
> -segment_wrap 24 : 분할단위기준으로 파일 개수(파일개수가 넘으면 000번 파일로 다시 기록된다.)

아래는 약 10분짜리 영상을 60초 단위로 분할한다. 

```poewrshell

❯ ffmpeg.exe -i .\BigBuckBunny.mp4 -codec copy -f segment -segment_list out.list -segment_time 60 -segment_wrap 24 out%03d.mp4

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
[segment @ 00000172c17a09c0] Opening 'out.list' for writing
[segment @ 00000172c17a09c0] Opening 'out000.mp4' for writing
Output #0, segment, to 'out%03d.mp4':
  Metadata:
    major_brand     : mp42
    minor_version   : 0
    compatible_brands: isomavc1mp42
    encoder         : Lavf58.45.100
    Stream #0:0(und): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 1280x720 [SAR 1:1 DAR 16:9], q=2-31, 1991 kb/s, 24 fps, 24 tbr, 12288 tbn, 24 tbc (default)
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
[segment @ 00000172c17a09c0] Opening 'out001.mp4' for writing
[segment @ 00000172c17a09c0] Opening 'out002.mp4' for writing
[segment @ 00000172c17a09c0] Opening 'out003.mp4' for writing
[segment @ 00000172c17a09c0] Opening 'out004.mp4' for writing
[segment @ 00000172c17a09c0] Opening 'out005.mp4' for writing
[segment @ 00000172c17a09c0] Opening 'out006.mp4' for writing
[segment @ 00000172c17a09c0] Opening 'out007.mp4' for writing
[segment @ 00000172c17a09c0] Opening 'out008.mp4' for writing
[segment @ 00000172c17a09c0] Opening 'out009.mp4' for writing
frame=14315 fps=0.0 q=-1.0 Lsize=N/A time=00:09:56.45 bitrate=N/A speed=1.36e+03x
video:144985kB audio:9144kB subtitle:0kB other streams:0kB global headers:0kB muxing overhead: unknown
```


## 참고 URL
- [StreamingGuide-FFmpeg](https://trac.ffmpeg.org/wiki/StreamingGuide)
- [Re-stream using FFmpeg with Wowza Streaming Engine](https://www.wowza.com/docs/how-to-restream-using-ffmpeg-with-wowza-streaming-engine) 
- [Video streaming with ffmpeg](https://www.acmesystems.it/ffmpeg)
- [Record fragments of 10 seconds using FFMPEG and DSHOW from webcam](https://superuser.com/questions/1019303/record-fragments-of-10-seconds-using-ffmpeg-and-dshow-from-webcam)


