---
layout: post
title: "TLS Verification Error 21번 오류"
description: "openssl 명령어를 이용해서 HTTPS 오류 및 인증서 관련 디버깅 방법을 정리해본다."
img: "letsencryption.png"
date: 2020-11-16 11:00:00 +0900
last_modified_at: 2020-11-16 11:00:00 +0900
tags: [openssl, letsencypt] # add tag
related: letsencypt
categories: tools
---

HTTPS 서비스를 위해 letsencrypt에서 받은 인증서를 적용하고 인증서 상태 체크를 해보면 아래와 같이 "Verification Error"가 발생하는 경우가 있다. 
<!--more-->

```bash
"Verify return code: 21 (unable to verify the first certificate)"
```

위와 같은 경우는 인증서에 문제가 있다기 보다는 fullchain.pem이 아닌 경우에 발생하는 문제이며 "openssl" 명령어를 이용해서 확인해 본다. 

## X509 Certificate 인증서 정보 보기

```bash
$ openssl x509 -text -noout -in fullchain.pem
```

## HTTPS 연결의 인증서 디버깅 명령어

```bash
$ openssl s_client -debug -connect softroom.duckdns.org:443
```

## Verification error 발생. 

letsencrypt에서 발급받은 인증서 중 chain.pem을 적용한 후 TLS 연결 시험시 아래와 같이 "Verify return code: 21"오류가 발생한다. 

```bash
$ openssl s_client -debug -connect softroom.duckdns.org:443
. . . (skip)
---
SSL handshake has read 2127 bytes and written 488 bytes
Verification error: unable to verify the first certificate
---
SSL-Session:
    . . . (skip)
    Start Time: 1605495880
    Timeout   : 7200 (sec)
    Verify return code: 21 (unable to verify the first certificate)
    Extended master secret: no                                                                                     
```

### Verification OK

letsencrypt에서 발급받은 인증서 중 fullchain.pem을 적용한 후 다시 연결 시험을 해 보면 아래와 같이 OK 상태로 보인다. 

```bash
$ openssl s_client -debug -connect softroom.duckdns.org:443
. . . (skip)
---
SSL handshake has read 3304 bytes and written 488 bytes
Verification: OK
---
SSL-Session:
    . . . (skip)
    Start Time: 1605498237
    Timeout   : 7200 (sec)
    Verify return code: 0 (ok)
    Extended master secret: no
```

### letsencrypt 인증서 파일 

letsencrypt에서 발급받은 인증서 파일은 아래와 같다. 

```bash
cert.pem -> ../../archive/softroom.duckdns.org/cert6.pem
chain.pem -> ../../archive/softroom.duckdns.org/chain6.pem
fullchain.pem -> ../../archive/softroom.duckdns.org/fullchain6.pem
privkey.pem -> ../../archive/softroom.duckdns.org/privkey6.pem
``` 


## 참고 URL
- [OpenSSL command cheatsheet](https://www.freecodecamp.org/news/openssl-command-cheatsheet-b441be1e8c4a/){:target="_blank"}
