---
layout: post
title: "ffmpeg로 영상 변환하기"
img: ffmpeg.png
date: 2019-04-10 17:06:00 +0900
tags: [ffmpeg] # add tag
categories: dev
---

## MP4 영상 스트리밍시 seek 가 안되는 경우 

mp4 영상을 ffprobe로 보면 아래와 같이 start 값이 영상의 길이보다 큰 경우가 있다. 
이런 경우에 seek를 하여 재생위치를 변경하면 초기 위치부터 재생이 된다. 

```
$ ffprobe filename.mp4
ffprobe version N-77791-g93ac72a Copyright (c) 2007-2016 the FFmpeg developers
  built with gcc 4.4.7 (GCC) 20120313 (Red Hat 4.4.7-16)
  configuration: --prefix=/home/lcsapp/ffmpeg_build --extra-cflags=-I/home/lcsapp/ffmpeg_build/include --extra-ldflags=-L/home/lcsapp/ffmpeg_build/lib --bindir=/home/lcsapp/bin --pkg-config-flags=--static --enable-gpl --enable-nonfree --enable-libfdk-aac --enable-libx264 --enable-shared
  libavutil      55. 13.100 / 55. 13.100
  libavcodec     57. 22.100 / 57. 22.100
  libavformat    57. 21.101 / 57. 21.101
  libavdevice    57.  0.100 / 57.  0.100
  libavfilter     6. 23.100 /  6. 23.100
  libswscale      4.  0.100 /  4.  0.100
  libswresample   2.  0.101 /  2.  0.101
  libpostproc    54.  0.100 / 54.  0.100
[h264 @ 0x22d0ea0] negative number of zero coeffs at 11 3
[h264 @ 0x22d0ea0] error while decoding MB 11 3
[h264 @ 0x22d0ea0] concealing 3398 DC, 3398 AC, 3398 MV errors in P frame
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

```
ffmpeg -i input.mp4 -codec copy putput.mp4 > /dev/null 2>&1
```

## 참고 URL
- [StreamingGuide-FFmpeg](https://trac.ffmpeg.org/wiki/StreamingGuide)
- [re-stream](https://www.wowza.com/docs/how-to-restream-using-ffmpeg-with-wowza-streaming-engine) 
- [Video streaming with ffmpeg](https://www.acmesystems.it/ffmpeg)


