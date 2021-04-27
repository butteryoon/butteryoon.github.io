---
layout: post
title: "Self-Signed 인증서와 개인키 만들기"
description: "HTTPS 테스트를 위해 openssl로 자체서명된 인증서 및 개인키를 만들고 웹서버에 적용해본다. "
img: "letsencryption.png"
date: 2021-04-27 22:00:00 +0900
last_modified_at: 2021-04-27 22:00:00 +0900
tags: [letsencrypt, openssl, self-signed, cert, key] # add tag
related: letsencrypt
categories: tools
---

HTTPS 테스트를 위해 openssl로 자체서명된 인증서 및 개인키를 만들고 웹서버에 적용해본다. 

duckdns.org와 같은 DDNS 서비스에서 도메인을 만들고 "letsencrypt"로 인증서를 받아서 사용하는 방법도 있으니 참고한다. 

- _[letsencrypt 인증서 발급하기]({{ site.baseurl }}{% link _posts/2018-10-10-letsencryption.markdown.md %}){:target="_blank"}_

<!--more-->

## Self-Signed 인증서와 개인키 만들기 

openssl로 인증서와 개인키를 한번에 생성하기 위해 "-subj" 옵션으로 발급자정보를 파라미터로 넘긴다. 

> -out cert.pem : 인증서  
> -keyout key.pem : 개인키  
> -nodes : PassPhrase 없이 생성  
> -days 3650 : 10년짜리   
> -subj : 발급자정보 

```bash
$ openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -nodes -days 3650 \
 -subj "/C=KR/ST=bundang/L=Seoul/O=butteryoon/OU=org/CN=butteryoon.duckdns.org" 

Generating a 4096 bit RSA private key
..................++
...................................................++
writing new private key to 'key.pem'
-----
```

## Self-Signed 인증서 확인 

만들어진 인증서(cert.pem)파일은 아래의 명령어로 확인해 보니 10년동안 유효한 인증서가 만들어졌다. 

> Not Before: Apr 27 13:54:26 2021 GMT  
> Not After : Apr 25 13:54:26 2031 GMT 

```bash
[opc@instance-20210115-1556 .ssl]$ openssl x509 -text -noout -in cert.pem | more

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            ad:0c:18:67:ae:d8:c8:33
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=KR, ST=bundang, L=Seoul, O=butteryoon, OU=org, CN=butteryoon.duckdns.org
        Validity
            Not Before: Apr 27 13:54:26 2021 GMT
            Not After : Apr 25 13:54:26 2031 GMT
        Subject: C=KR, ST=bundang, L=Seoul, O=butteryoon, OU=org, CN=butteryoon.duckdns.org
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (4096 bit)
                Modulus:
                    00:d4:70:ec:e7:73:02:41:91:bf:e8:d1:03:55:12:
                    f4:4c:b2:6b:fe:34:7b:6e:48:7d:60:47:4d:eb:c9:
```

## 인증서 적용 테스트

인증서 적용 테스트를 위해 python3로 인증서가 적용된 웹서버를 띄워 본다. 

```python
[opc@instance-20210115-1556 www]$ python3
Python 3.6.8 (default, Nov 11 2020, 09:19:43)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-44.0.3)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>>
>>> from http.server import HTTPServer, SimpleHTTPRequestHandler
>>> import ssl
>>> httpd = HTTPServer(('', 5443), SimpleHTTPRequestHandler)
>>> httpd.socket = ssl.wrap_socket (httpd.socket, keyfile='/home/opc/.ssl/key.pem', certfile='/home/opc/.ssl/cert.pem', server_side=True)
>>> httpd.serve_forever()
106.240.249.74 - - [27/Apr/2021 23:40:17] "GET / HTTP/1.1" 200 -
```

## curl https 연결 테스트 

Self-Signed 인증서이기 때문에 "--insecure" 파라미터를 추가하고 "-v"로 진행과정을 보면 TLSv1.2 연결이 확인된다. 

```bash
❯ curl --insecure https://butteryoon.duckdns.org:5443 -v

*   Trying 193.122.124.127:5443...
* TCP_NODELAY set
* Connected to butteryoon.duckdns.org (193.122.124.127) port 5443 (#0)
* ALPN, offering h2
* ALPN, offering http/1.1
* successfully set certificate verify locations:
*   CAfile: /etc/ssl/certs/ca-certificates.crt
  CApath: /etc/ssl/certs
* TLSv1.3 (OUT), TLS handshake, Client hello (1):
* TLSv1.3 (IN), TLS handshake, Server hello (2):
* TLSv1.2 (IN), TLS handshake, Certificate (11):
* TLSv1.2 (IN), TLS handshake, Server key exchange (12):
* TLSv1.2 (IN), TLS handshake, Server finished (14):
* TLSv1.2 (OUT), TLS handshake, Client key exchange (16):
* TLSv1.2 (OUT), TLS change cipher, Change cipher spec (1):
* TLSv1.2 (OUT), TLS handshake, Finished (20):
* TLSv1.2 (IN), TLS handshake, Finished (20):
* SSL connection using TLSv1.2 / ECDHE-RSA-AES256-GCM-SHA384
* ALPN, server did not agree to a protocol
* Server certificate:
*  subject: C=KR; ST=bundang; L=Seoul; O=butteryoon; OU=org; CN=butteryoon.duckdns.org
*  start date: Apr 27 13:54:26 2021 GMT
*  expire date: Apr 25 13:54:26 2031 GMT
*  issuer: C=KR; ST=bundang; L=Seoul; O=butteryoon; OU=org; CN=butteryoon.duckdns.org
*  SSL certificate verify result: self signed certificate (18), continuing anyway.
```

## 브라우저 경고

Self-Signed 인증서는 CA 인증을 받을 수 없기 때문에 브라우저에서는 아래와 같은 경고를 표시하고 사용자의 선택에 의해 접근할 수 있다. 

![Self-Signed Warning]({{site.baseurl}}/assets/img/privatecert_edge_warning.png)

인터넷 익스플로러에서는 인증서를 "신뢰할 수 있는 루트 인증 기관"으로 저장하면 경고 없이 테스트 가능하다. 

![Self-Signed Warning]({{site.baseurl}}/assets/img/privatecert_save-04.png)


## 참고

- [Create Privateivate Cert](https://stackoverflow.com/questions/10175812/how-to-create-a-self-signed-certificate-with-openssl){:target="_blank"}
- [OpenSSL 로 ROOT CA 생성 및 SSL 인증서 발급](https://www.lesstif.com/system-admin/openssl-root-ca-ssl-6979614.html){:target="_blank"}
