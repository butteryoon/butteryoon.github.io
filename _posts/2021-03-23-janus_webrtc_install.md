---
layout: post
title: "Janus WebRTC Gateway 서버 시험"
description: "OCI(오라클 클라우드 인스턴스) Ubuntu 20.04 LTS 배포에 Janus WebRTC Server를 설치하고 시험해본다."
img: webrtc-title.webp
date: 2021-04-03 16:00:00 +0900
last_modified_at: 2021-04-03 16:00:00 +0900
tags: [Janus, webrtc, streaming] # add tag
related: streaming
categories: dev
---

[WebRTC](https://webrtc.org) 기술에 대해 서베이 중 WebRTC Gateway 오픈소스가 있어 살펴본다. 

WebRTC는 기존으로 P2P방식으로 사용자간 실시간 커뮤니케이션을 목적으로 하지만 미디어서버로 동작하며 N명의 사용자에게 브로트캐스팅 하는 목적으로 사용되기도 한다. 

인코딩 SW와 웹플레이어 방식의 실시간 스트리밍은 유튜브나 트위치 같은 퍼블릭 방송 플랫폼에서 사용하고 있지만 1초미만의 지연시간을 가지는 실시간 스트리밍 환경을 구성하기는 힘들었다. 하지만 WebRTC를 이용하면 웹 브라우저 환경에서 간단하게 초저지연 실시간 스트리밍 서비스를 구현할 수 있다. 

WebRTC의 기술적인 사양은 차차 공부해가기로 하고 "WebRTC Server Gateway" 구현체인 "[Janus WebRTC Gateway](https://github.com/meetecho/janus-gateway)"를 설치하고 시험해 보기로 한다. 
<!--more-->

## Janus WebRTC Server install  

"meetecho"에서 [README](https://janus.conf.meetecho.com/docs/README.html)페이지를 참고하여 설치해 보기로 한다. (README 파일이 꽤 친절하다.)

서버는 오라클 인스턴스 프리티어에 설치된 우분투 20.04 배포를 사용한다. 
> Server : Oracle Cloud Instance Ubuntu 20.04 LTE Free Tier   
> WSL 또는 MacOS에서도 설치 가능. 

Ubuntu 20.04 환경에 맞지 않는 부분만 기록해본다. 

## 필수 라이브러리 설치 

"aptitude"를 이용해서 기본 라이브러리 패키지를 설치한다. 

> aptitude는 Ncurses 인터페이스로 패키지를 관리할 수 있다. 
> libsrtp-dev는 수동 설치한다. 

```bash
sudo aptitude install \
  libmicrohttpd-dev libjansson-dev libssl-dev libsrtp2-dev \
  libsofia-sip-ua-dev libglib2.0-dev libopus-dev \
  libogg-dev libcurl4-openssl-dev liblua5.3-dev libconfig-dev \
  pkg-config gengetopt libtool automake
```

## libsrtp-dev v2.2.0

README 파일에서는 "libsrtp-dev"를 설치하라고 하는데 Ubuntu 20.04 LTS에서는 찾을 수 없어 libsrtp2-dev를 설치했다. 

libsrtp-dev는 Janus WebRTC서버에서 Data Channel 데모에서 사용하는데 Ubuntu 20.04에서는 오류가 생겨서 수동으로 기존 libsrtp2-dev를 삭제하고 수동으로 패키지를 설치해야 한다. 

libsrtp2-dev는 "2.3.x"버전인데 아래의 이슈가 있어 "2.2.0"버전으로 설치. 

> [Oops, error creating inbound SRTP session](https://github.com/meetecho/janus-gateway/issues/2024){:target="_blank"} 이슈     
> 2.3.0 버전도 '--enable-openssl' 옵션을 추가하면 된다고 한다. 

```bash
sudo aptitude remove libsrtp2-dev 

wget https://github.com/cisco/libsrtp/archive/v2.2.0.tar.gz
tar xfv v2.2.0.tar.gz
cd libsrtp-2.2.0
./configure --prefix=/usr --enable-openssl
make shared_library && sudo make install
```

## SSL 라이브러리 

OpenSSL 대신 [BoringSSL](https://boringssl.googlesource.com/boringssl/) 라이브러리를 쓸 수 있지만 패키지로 설치된 OpenSSL을 쓰기로 한다.

> BoringSSL는 하트블리드 보안취약점에 대응하기 위해 구글에서 구현하여 배포하는 패키지.   
> BoringSSL을 쓴면 "dtls timeout" 옵션을 설정하여 fast retransmission을 지원할 수 있다고 한다. (우선 패스)

## usrsctp

Data Channel 데모를 위해 필요하니 설치한다. 

```bash
git clone https://github.com/sctplab/usrsctp
cd usrsctp
./bootstrap
./configure --prefix=/usr --disable-programs --disable-inet --disable-inet6
make && sudo make install
```

## Janus Gateway compile

/opt/janus 디렉토리에 패키지를 설치한다. 특정기능을 사용하지 않을 수도 있지만 데모 확인을 위해 기본 설정으로 진행한다. 

```bash
git clone https://github.com/meetecho/janus-gateway.git
cd janus-gateway/
sh autogen.sh
sudo ./configure --prefix=/opt/janus
sudo make
sudo make install
sudo make configs   # 기본설정파일 생성. 
```

## Settings

Janus WebRTC Gateway를 위한 기본 설정파일은 "/opt/janus/etc/janus" 디렉토리에 있고 기본으로는 아래의 설정만 변경한다. 

 - janus.jcfg
 - janus.transport.http.jcfg

### - janus.jcfg  

janus를 백그라운드로 띄운 상태에서 로그를 확인하기 위해 로그 파일 경로를 설정하고 DTLS 연결을 위한 인증서를 등록한다. 

> 인증서를 등록하지 않으면 Janus에서 임시로 셍상힌디.     
> "No cert/key specified, autogenerating some..."  

```bash
general: {
  log_to_file = "/opt/janus/log/janus.log"  # Whether to use a log file or not
}
certificates: {
  cert_pem = "/usr/local/antmedia/conf/fullchain.pem"
  cert_key = "/usr/local/antmedia/conf/privkey.pem"
}
```

### - janus.transport.http.jcfg

"Janus WebRTC Gateway" 서버와 웹서버가 동일서버에 있고 localhost로 테스트 한다면 https 설정과 인증서 설정은 하지 않아도 된다. 

하지만 외부 웹서버에서 WebRTC 디바이스를 오픈하려면 HTTPS가 설정이 필요하다. (크롬 보안설정에서 바꿀 수 있는지는 모르겠다. )

> 인증서는 "Let's Encrypt"에서 발급. 

```bash
general: {
  https = true              # Whether to enable HTTPS (default=false)
  secure_port = 8089        # Web server HTTPS port, if enabled
}
admin: {
  admin_https = true        # Whether to enable HTTPS (default=false)
  admin_secure_port = 7889  # Admin/monitor web server HTTPS port, if enabled
}
certificates: {}
  cert_pem = "/usr/local/antmedia/conf/fullchain.pem"
  cert_key = "/usr/local/antmedia/conf/privkey.pem"
}
```

## janus gateway 서버 실행. 

"janus WebRTC Gateway" 서버를 백그라운드로 실행하고 stun 서버는 구글서버를 이용한다. 

세부 파라미터는 "/opt/janus/bin/janus --help"로 확인한다.   
> -F : jcfg 디렉토리  
> -S : STUN 서버 설정 (구글에서 제공하는 stun서버 설정)
> -b : 백그라운드로 실행. 

```bash
sudo /opt/janus/bin/janus -F /opt/janus/etc/janus -S stun.l.google.com:19302 -b 
```

실행로그를 보면 아래와 같이 ICE 모드의 정보와 STUN 테스트 결과를 볼 수 있다. 

```bash
Initializing ICE stuff (Full mode, ICE-TCP candidates disabled, half-trickle, IPv6 support disabled)
STUN server to use: stun.l.google.com:19302
  >> 172.217.213.127:19302 (IPv4)
Testing STUN server: message is of 20 bytes
  >> Our public address is 193.123.xxx.xxx'
```

각 플러그인의 초기화 로그 아래 각 REST-API를 처리하기 위한 웹서버 포트 정보와 상태가 표시된다. 

> 웹서버가 https면 "janus gateway" REST요 서버도 https로 호출된다.   

```bash 
. . . (skip plugin log)
HTTP webserver started (port 8088, /janus path listener)...
HTTPS webserver started (port 8089, /janus path listener)...
Admin/monitor HTTP webserver started (port 7088, /admin path listener)...
Admin/monitor HTTPS webserver started (port 7889, /admin path listener)...
JANUS REST (HTTP/HTTPS) transport plugin initialized!
Loading transport plugin 'libjanus_nanomsg.so'...
JANUS Nanomsg transport plugin initialized!
Loading transport plugin 'libjanus_websockets.so'...
Nanomsg thread started
^[[33m[WARN]^[[0m libwebsockets has been built without IPv6 support, will bind to IPv4 only
libwebsockets logging: 0
WebSockets server started (port 8188)...
JANUS WebSockets transport plugin initialized!
WebSockets thread started
```

## Demos 웹 서비스 

"Janus WebRTC Gateway"를 구동했으면 DEMO 페이지를 띄워보자. 데모는 "/opt/janus/share/janus/demos/" 디렉토리에 js, html 파일이 있어 Static 웹서버를 구동하여 띄울 수 있다. 

시험을 위해 "python3"로 웹서버를 띄우고 https를 위해 아래와 같이 인증서를 설정해준다. 

```python
from http.server import HTTPServer, SimpleHTTPRequestHandler
import ssl

httpd = HTTPServer(('', 58803), SimpleHTTPRequestHandler)

httpd.socket = ssl.wrap_socket (httpd.socket, keyfile='/usr/local/antmedia/conf/privkey.pem', certfile='/usr/local/antmedia/conf/fullchain.pem', server_side=True)

httpd.serve_forever()
```

아래와 같이 About 페이지가 보이고 "Demos" 링크에서 각 플러그인 기능을 시험해 볼 수 있다. 

![demo_about]({{site.baseurl}}/assets/img/janus_demos_list.webp)

## Janus Directory 

```bash
ubuntu@instance-20210116-0003:/opt/janus$ tree -d
.
├── bin                     # janus 바이너리. 
├── etc
│   └── janus               # make configs로 만들어지는 기본 설정파일
├── include
│   └── janus
│       ├── events
│       ├── loggers
│       ├── plugins
│       └── transports
├── lib
│   └── janus
│       ├── events
│       ├── loggers
│       ├── plugins
│       └── transports
├── log
└── share         
    ├── doc
    │   └── janus-gateway
    ├── janus
    │   ├── demos            # 데모 페이지 
    │   │   ├── css
    │   │   ├── docs
    │   │   ├── surround
    │   │   └── voicemail
    │   ├── duktape
    │   ├── javascript
    │   ├── lua
    │   ├── recordings
    │   └── streams
    └── man
        └── man1
```

## Oops, error creating inbound SRTP session ..  

- [issue](https://github.com/meetecho/janus-gateway/issues/2024)

데이타채널을 사용하는 플러그인 데모에서 srtp 오류가 발생하는 이슈이고 libsrtp-dev 버전이 2.3.x인 경우 발생한다고 한다. 이 문제 때문에 libsrtp-dev 2.2.0 버전을 설치해야 한다. 

```bash
[6715407664108631] Creating ICE agent (ICE Full mode, controlling)
[WARN] [6715407664108631] Failed to add some remote candidates (added 0, expected 1)
[ERR] [dtls.c:janus_dtls_srtp_incoming_msg:923] [6715407664108631] Oops, error creating inbound SRTP session for component 1 in stream 1??
[ERR] [dtls.c:janus_dtls_srtp_incoming_msg:924] [6715407664108631]  -- 1 (srtp_err_status_fail)
[janus.plugin.textroom-0x7f80fc003e10] No WebRTC media anymore
```



aptitude로 설치된 패키지 삭제 후 v2.2.0버전을 컴파일하여 설치한다. 

```bash
ubuntu@instance-20210116-0003:~/Projects/Janus$ sudo aptitude remove libsrtp2-dev
The following packages will be REMOVED:
  libpcap0.8-dev{u} libsrtp2-1{u} libsrtp2-dev
0 packages upgraded, 0 newly installed, 3 to remove and 64 not upgraded.
Need to get 0 B of archives. After unpacking 1317 kB will be freed.
Do you want to continue? [Y/n/?] Y
(Reading database ... 160262 files and directories currently installed.)
Removing libsrtp2-dev (2.3.0-2) ...
Removing libpcap0.8-dev:amd64 (1.9.1-3) ...
Removing libsrtp2-1:amd64 (2.3.0-2) ...
Processing triggers for man-db (2.9.1-1) ...
Processing triggers for libc-bin (2.31-0ubuntu9.1) ...
```


## 참고 URL
- [Janus - General purpose WebRTC server](https://janus.conf.meetecho.com/docs/index.html){:target="_blank"}
- [CentOS 7에 Janus-gateway(WebRTC media server) 설치하기](https://phodobit.kr/51){:target="_blank"}
- [Janus gateway #2 설정 및 실행 (CentOS)](https://blog.naver.com/espeniel/221838760629){:target="_blank"}
- [Oops, error creating inbound SRTP session](https://github.com/meetecho/janus-gateway/issues/2024){:target="_blank"}
- [WebRTC분과 -웹표준기술융합포럼](https://www.facebook.com/groups/rtc.korea){:target="_blank"}
- [WebRTC Janus Gateway 오픈소스 세부 동작 구조](https://videocodec.tistory.com/2103?fbclid=IwAR2kjm46ZK-Lyo1L-VbuLQoB9-osE4pv2GsnxZhUR7-zEjqA8TXkQ4Cv-lY){:target="_blank"}