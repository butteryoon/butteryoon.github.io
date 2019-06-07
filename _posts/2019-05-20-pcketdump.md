---
layout: post
title: "tshark을 이용한 패킷덤프"
img: wireshark.png
date: 2019-05-20 00:00:00 +0900
tags: [packet, tshark, wireshark, setcap, dumpcap] # add tag
categories: dev
---

패킷분석이라고 하면 보통 [Wireshark][Wireshark]을 떠올린다. Windows, Linux, MacOS등 대부분의 OS에서 동일한 모양의 GUI 인터페이스를 제공하지만 tshark 이라는 명령행 프로그램도 제공한다. 리눅스 터미널에서 wireshark과 동일한 필터를 사용할 수 있고 GUI를 쓰지 않고 터미널에서 바로 분석을 할 수 있다. 
> tshark 과 dumpcap 은 뭐가 다르지?

## root 권한 없이 tshark 사용하기

리눅스 서버에 root 권한이 없을 때 tshark 사용하려면 아래와 같이 일반유저에게 권한을 부여해준다. 
> 특정유저에게 권한을 부여하는 부분은 아래 [Platform-Specific information about capture privileges](https://wiki.wireshark.org/CaptureSetup/CapturePrivileges) 참고한다.  
> root 계정으로 아래의 명령을 실행한다.  

```
# setcap cap_net_raw,cap_net_admin+eip /usr/sbin/dumpcap
```

## tshark을 이용한 패킷 저장. 

> -P : Decode and display packets even while writing raw packet data using the -w option.  
> -b duration:600 600초 마다 파일 변경  
> -b filesize:10240 1MB마다 파일 변경  

```
$ tshark -ta -P -i em2 port 38010 
	-b duration:600 
	-w stream.pcap
```

## tshark 명령행으로 특정 프로토콜 분석
> HTTP 8080 URL을 캠쳐하면서 http 디코더로 프로토콜 분석  
> Wireshark 에서 "Decode As" 기능과 동일하다.  

```
$ tshark -P -ta -i em2 port 8080 -d tcp.port==8080,http
```

## tshark 명령행으로 DTLS Application Data 분석을
> TLS 패킷 분석을 위해서는 개인키가 필요하며 IP:PORT 정의와 분석할 프로토콜을 정의한다. 
> -o "dtls.keys_list:IPADDR,PORT,PROTOCOL,private_key.pem"  
>
> dtls.record.content_type==23 : Application Data  
> dtls.record.content_type==22 : TLS Handshaek  

```
$ tshark -P -ta -i em2 port 38010 
	-o "ssl.debug_file:ssldebug.log" 
	-o "ssl.desegment_ssl_records: TRUE" 
	-o "ssl.desegment_ssl_application_data: TRUE"  
	-o "dtls.keys_list:192.168.0.123,38010,rtp,private_key.pem" 
	-R "dtls.record.content_type == 23"
```

## 참고 URL
- [Tshark Command Examples](https://linuxsimba.com/tshark-examples)
- [Analyze HTTP Requests With TShark](https://kvz.io/blog/2010/05/15/analyze-http-requests-with-tshark/)
- [Dump and analyze network traffic](https://explainshell.com/explain?cmd=tshark++-d+udp.port%3D%3D8472%2Cvxlan+-r+1.cap+"tcp.analysis.duplicate_ack_num%3D%3D1")
- [TShark Command Line using PowerShell (by Graham Bloice)](https://sharkfesteurope.wireshark.org/assets/presentations17eu/33.7z)
- [Platform-Specific information about capture privileges](https://wiki.wireshark.org/CaptureSetup/CapturePrivileges)
- [Wireshark Packet Capture: Tshark Vs. Dumpcap](https://www.networkcomputing.com/networking/wireshark-packet-capture-tshark-vs-dumpcap)

[Wireshark]: https://www.wireshark.org
