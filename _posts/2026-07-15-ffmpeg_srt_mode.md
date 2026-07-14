---
layout: post
comments: true
title: "ffmpeg로 SRT 스트리밍 해보기 (listener / caller 모드)"
description: "ffmpeg로 mp4 파일을 SRT 프로토콜로 실시간 전송하고 ffplay로 재생해본다."
img: m_ffmpeg_srt_title.webp
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-15 00:00:00 +0900
tags: [ffmpeg, ffplay, srt, mpegts, streaming] # add tag
related: ffmpeg
categories: dev
---

2020년에 ffmpeg로 SRT 스트리밍을 테스트하면서 적어둔 초안을 보완해서 발행한다. mp4 파일을 ffmpeg로 읽어 SRT 프로토콜로 전송하고, ffplay로 받아서 재생하는 가장 단순한 구성을 정리한다.

<!--more-->

## SRT 프로토콜이란

SRT(Secure Reliable Transport)는 Haivision이 공개한 오픈소스 저지연 스트리밍 프로토콜이다. UDP 기반이지만 패킷 재전송(ARQ)과 타임스탬프 기반 전송으로 불안정한 공용 인터넷 구간에서도 안정적인 전송이 가능하고, AES 암호화도 지원한다. RTMP를 대체하는 기여(contribution)용 프로토콜로 방송 업계에서 많이 쓰이고 있으며, OBS Studio도 SRT 송출/수신(ingest)을 지원한다.

SRT 연결에는 세 가지 모드가 있다.

- **listener** : 서버처럼 포트를 열고 상대의 접속을 기다린다.
- **caller** : 클라이언트처럼 상대 주소로 접속한다. (기본값)
- **rendezvous** : 양쪽이 동시에 서로에게 접속한다. NAT 환경에서 유용하다.

ffmpeg의 SRT URL 형식은 `srt://hostname:port?option1=value1&option2=value2` 이고, `mode` 외에도 `latency`(재전송 버퍼 지연), `streamid`, `pkt_size`, `passphrase`(암호화) 같은 옵션을 쿼리스트링으로 지정할 수 있다.

## SRT Server(listener) Mode

mp4 파일을 실시간 속도(`-re`)로 읽어서 재인코딩 없이(`-c copy`) MPEG-TS 컨테이너로 감싸 SRT listener로 대기시킨다.

```
❯ ffmpeg -re -i .\BigBuckBunny.mp4 -c copy -f mpegts "srt://:8888?mode=listener"
```

- `-re` : 파일을 실제 재생 속도로 읽는다. 라이브 스트리밍 흉내를 낼 때 필수다.
- `-c copy` : 트랜스코딩 없이 스트림을 그대로 복사한다.
- `-f mpegts` : SRT는 보통 MPEG-TS를 실어 나르므로 컨테이너를 mpegts로 지정한다.
- `mode=listener` : 8888 포트를 열고 클라이언트 접속을 기다린다. 호스트를 생략하면 모든 인터페이스에 바인딩된다.

## SRT Client(caller) Mode

다른 터미널에서 ffplay로 접속해서 재생한다. `mode`를 생략하면 기본값이 caller라서 그냥 주소만 주면 된다.

```
❯ ffplay srt://localhost:8888
```

반대로 ffmpeg 쪽을 caller로, 수신 쪽(ffplay 또는 OBS의 Media Source)을 listener로 구성할 수도 있다. 예를 들어 OBS에서 `srt://0.0.0.0:8888?mode=listener`로 입력 소스를 만들어 두고, ffmpeg가 `srt://obs호스트:8888` 로 밀어 넣는 식이다.

## 참고
- [ffmpeg Protocols Documentation - srt](https://ffmpeg.org/ffmpeg-protocols.html#srt){:target="_blank"}
- [SRT Alliance](https://www.srtalliance.org/){:target="_blank"}
- [Haivision srt (GitHub)](https://github.com/Haivision/srt){:target="_blank"}
