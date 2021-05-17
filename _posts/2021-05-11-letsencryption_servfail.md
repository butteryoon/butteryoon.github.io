---
layout: post
title: "letsencrypt 인증서 재발급 실패(SERVFAIL)"
description: "letsencrypt-auto 스크립트로 인증서를 재발급할 때 SERVFAIL로 실패하는 경우에 대해 기술한다."
img: "letsencryption.png"
date: 2021-05-11 15:05:00 +0900
last_modified_at: 2021-05-11 15:05:00 +0900
tags: [letsencrypt, HTTPS, 인증서, duckdns.org, cloudflare] 
related: letsencrypt
categories: dev
---

**Letsencrypt** 인증서 갱신할 때 "CAA record for iws.iptime.org prevents issuance"오류가 발생하는 경우가 있어 해결방법을 기술한다. 

<!--more-->

## 인증서 재발급 (renew) 

Letsencrypt로 기존에 발급 받은 인증서를 90일 이전에 재발급 할 경우에는 아래와 같이 **--renew-by-default** 옵션으로 실행하며 진행과정은 초기와 동일하다. 

> **Let's Encrypt Expiry Bot <expiry@letsencrypt.org>**이 친절하게 메일을 보내준다.  

```bash
[root@localhost letsencrypt]# ./letsencrypt-auto certonly --renew-by-default --manual --no-bootstrap -d test.domain.com

./letsencrypt-auto has insecure permissions!
To learn how to fix them, visit https://community.letsencrypt.org/t/certbot-auto-deployment-best-practices/91979/
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c' to cancel): test.domain.com
Renewing an existing certificate for test.domain.com
Performing the following challenges:
http-01 challenge for test.domain.com
```

## 인증서 발급 및 갱신 실패 

개발용으로 쓰고 있던 softroom.duckdns.org DDNS로 갱신을 하려고 하니 아래와 같이 CAA record 오류가 발생하는 경우가 있다. 

**SERVFAIL**이 나오는 경우 DNS를 **"1.1.1.1"** 과 같은 DNSSEC을 지원하는 네임서버로 바꾸고 시도하면 재발급이 완료된다. 

*참고 : [The domain’s name servers may be malfuntioning](https://community.letsencrypt.org/t/the-domains-name-servers-may-be-malfuntioning/121730/3)*

```bash
Waiting for verification...
Challenge failed for domain softroom.duckdns.org
http-01 challenge for softroom.duckdns.org
Cleaning up challenges
Some challenges have failed.

IMPORTANT NOTES:
 - The following errors were reported by the server:

   Domain: softroom.duckdns.org
   Type:   dns
   Detail: DNS problem: SERVFAIL looking up CAA for
   softroom.duckdns.org - the domain's nameservers may be malfunctioning
``` 

## Cloudflare 1.1.1.1 DNS 적용 

클라우드플레어에서 제공하는 DNS를 설정은 [1.1.1.1 Setup on PC](https://1.1.1.1/dns/) 가이드를 참고한다. 

- 설정 > 네트워크 및 인터넷 > 이더넷 > IP 설정 

![1.1.1.1]({{site.baseurl}}/assets/img/cloudflare_dns_setting.webp)


이전에 "iptime.org" DDNS의 경우에는 "CAA RR(리소스 레코드)"가 허용목록에 없어서 발급이 안되는 경우였고 아래 링크에 오류 내용을 적어뒀다. 

*[letsencrypt 인증서 발급하기]({{ site.baseurl }}{% link _posts/2018-10-10-letsencryption.markdown.md %}){:target='_blank"}*

## 참고 

- [Let's Encrypt 시작하기](https://letsencrypt.org/ko/getting-started/)
- [Let's Encrypt로 무료로 HTTPS 지원하기](https://blog.outsider.ne.kr/1178) 
- [Let's encrypt의 인증서를 생성할 때 주의사항](https://findstar.pe.kr/2018/09/08/lets-encrypt-certificates-rate-limit/)
- [Install Let's Encrypt to Create SSL Certificates](https://www.linode.com/docs/guides/install-lets-encrypt-to-create-ssl-certificates/)
