---
layout: post
title: "letsencrypt 인증서 발급하기"
description: "letsencrypt 인증서 발급하기"
img: "letsencryption.png"
date: 2019-04-20 20:05:00 +0900
last_modified_at: 2021-03-23 15:05:00 +0900
tags: [letsencrypt, HTTPS, 인증서] 
related: letsencrypt
categories: dev
---

아이폰 엔터프라이즈 앱의 인하우스 배포를 위해서는 도메인이 필요하다.

그래서 letsencrypt 무료 인증서를 받아 사용하기로 한다.

회사 웹서버는 호스팅을 하고 있고 수정이 피곤하여 배포시험을 위한 임시 서버에 인증서만 받아 인하우스 배포를 시험하기로 한다.

letsencrypt 에서 인증서를 받기 위한 절차는 아래와 같다.

생각보다 간단하다.

1. 인증서를 받기 위해 도메인이 있어야 한다. 
2. 메뉴얼 모드로 인증서를 받는다. 
3. 인증서는 90일 마다 갱신해야 한다. 
   - 자동화 스크립트 설정이 가능하다. 
4. 끝. 


## git 저장소에서 letsencrypt clone 

```bash
git clone https://github.com/letsencrypt/letsencrypt
```

## 인증서 신규 발급 

> certonly --manual  
> nginx 또는 아파치 웹서버 구동 환경에서는 "certbot-auto" 스크립트를 사용하여 자동으로 적용까지 가능하다.  


설치 후 인증서만 다운로드 하기 위해 아래의 명령어를 실행. 

> ./letsencrypt-auto certonly --manual  

```bash
[root@localhost letsencrypt]# ./letsencrypt-auto certonly --manual
Bootstrapping dependencies for RedHat-based OSes... (you can skip this with --no-bootstrap)
yum is /bin/yum
To use Certbot, packages from the EPEL repository need to be installed.
Enable the EPEL repository and try running Certbot again.
```

git pull로 최신버전 다운로드 이후 아래와 같이 Bootstrap 관련 경고가 발생하여 **--no-Bootstrap** 옵션을 추가 한다.    
성공하면 /etc/letsencrypt/live/도메인 디렉토리에 인증서가 생성된다. 

> ./letsencrypt-auto certonly --manual --no-bootstrap -d test.duckdns.org

```bash
[@localhost letsencrypt]# ./letsencrypt-auto certonly --manual --no-bootstrap -d test.duckdns.org
Requesting to rerun ./letsencrypt-auto with root privileges...
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): test.domain.com
Cert is due for renewal, auto-renewing...
Renewing an existing certificate
Performing the following challenges:
http-01 challenge for test.domain.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y
```

## 도메인 소유 인증 

letsencrypt에서는 도메인소유 인증을 위해 지정한 도메인의 특정 URL로 요청하여 지정된 문자열을 확인한다. 

```bash
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:
h70rfIM6APUpoMV-1afecuZfUgBw3Bdy4EMq0FftQI4.......tyLmv3IZf3ebDBsJhI
And make it available on your web server at this URL:
http://test.domain.com/.well-known/acme-challenge/h70rfIM6APUpoMV-......4EMq0FftQI4

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```

실서버의 구조를 바꾸기 어려운 경우 파이썬으로 임시 웹서버를 구동하여 진행한다. 

간단하게 파이썬 **SimpleHTTPServer** 모듈로 가능하다    

> python3 에서 SimpleHTTPServer는 http.server 안에 통합되었다. (http://bit.ly/2Zt39My) 

```bash
python -m SimpleHTTPServer 8000 
or 
python3 -m http.server 8000
``` 

도메인소유 인증을 시도하면 아래와 같은 웹 로그를 확인할 수 있다.    
> 4개의 호스트에서 접속을 시도를 한다.  

```bash
python -m SimpleHTTPServer 8803
Serving HTTP on 0.0.0.0 port 8803 ...
34.211.6.84 - - [10/Dec/2020 18:20:42] "GET /.well-known/acme-challenge/DC1WQ5Mdx6C0ffJnFuuizBFqM5wD28wtb-cXioT-O30 HTTP/1.1" 200 -
3.22.70.135 - - [10/Dec/2020 18:20:42] "GET /.well-known/acme-challenge/DC1WQ5Mdx6C0ffJnFuuizBFqM5wD28wtb-cXioT-O30 HTTP/1.1" 200 -
18.196.96.172 - - [10/Dec/2020 18:20:42] "GET /.well-known/acme-challenge/DC1WQ5Mdx6C0ffJnFuuizBFqM5wD28wtb-cXioT-O30 HTTP/1.1" 200 -
66.133.109.36 - - [10/Dec/2020 18:20:42] "GET /.well-known/acme-challenge/DC1WQ5Mdx6C0ffJnFuuizBFqM5wD28wtb-cXioT-O30 HTTP/1.1" 200 -
```

## 인증서 생성 확인 

도메인 소유 인증이 성공하면 아래와같이 간단한 설명을 보여주며 갱신할때는 renew 옵션을 사용한다. 

```bash
Press Enter to Continue
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/test.domain.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/test.domain.com/privkey.pem
   Your cert will expire on 2021-03-10. To obtain a new or tweaked
   version of this certificate in the future, simply run
   letsencrypt-auto again. To non-interactively renew *all* of your
   certificates, run "letsencrypt-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

etc 디렉토리에 아래처럼 4개의 인증서파일이 생성된다 

```bash
[root@localhost letsencrypt]# sudo ls -ltr /etc/letsencrypt/live/test.domain.com
합계 4
-rw-r--r-- 1 root root 682  7월 30  2018 README
lrwxrwxrwx 1 root root  41 12월 10 18:20 privkey.pem -> ../../archive/test.domain.com/privkey7.pem
lrwxrwxrwx 1 root root  43 12월 10 18:20 fullchain.pem -> ../../archive/test.domain.com/fullchain7.pem
lrwxrwxrwx 1 root root  39 12월 10 18:20 chain.pem -> ../../archive/test.domain.com/chain7.pem
lrwxrwxrwx 1 root root  38 12월 10 18:20 cert.pem -> ../../archive/test.domain.com/cert7.pem
```  


## 기존 인증서 재발급 (renew) 

기존에 발급 받은 인증서를 90일 이전에 재발급 할 경우에는 아래와 같이 **--renew-by-default** 옵션을 추가한다.  

> 친절하게 메일을 보내준다.  

```bash
[root@localhost letsencrypt]# ./letsencrypt-auto certonly --renew-by-default --manual --no-bootstrap
./letsencrypt-auto has insecure permissions!
To learn how to fix them, visit https://community.letsencrypt.org/t/certbot-auto-deployment-best-practices/91979/
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): test.domain.com
Renewing an existing certificate for test.domain.com
Performing the following challenges:
http-01 challenge for test.domain.com
```

## 인증서 발급 및 갱신 실패 

개발용을 쓰고 있던 test.iptime.org DDNS로 갱신을 하려고 하니 아래와 같이 CAA record 오류가 발생한다. 

"iptime.org"와 같은 DDNS 서비스인 경우 letsencrypt에서 발급가능한 서브 도메인 개수가 정해져 있어서 그럴 수 있다고 하는데 다시 해보고 안되면 인증서 발급 제한정책을 알아봐야겠다. 

```bash
Waiting for verification...
Challenge failed for domain test.iptime.org
http-01 challenge for test.iptime.org
Cleaning up challenges
Some challenges have failed.

IMPORTANT NOTES:
 - The following errors were reported by the server:
 -
 - Domain: test.iptime.org
 - Type:   caa
 - Detail: CAA record for test.iptime.org prevents issuance.
``` 

### CAA 인증 

인증기관은 인증서를 발급하기 전에 CAA RR(리소스 레코드)를 확인하고 처리해야 하며 Let's Encrypt는 2020년 6월 이후 [End of Life Plan for ACMEv1](https://community.letsencrypt.org/t/end-of-life-plan-for-acmev1/88430) 정책에 따라 ACMEv2로 변경되면서 CAA RR을 확인할 수 없는 도메인에서는 SSL 인증서를 발급받을 수 없다고 한다. 

> CAA는 사이트 소유자가 도메인 이름을 포함한 인증서를 발급할 수 있는 CA(인증기관)을 지정할 수 있도록 하는 DNS 레코드 유형 입니다.  
> 참고 : [인증기관 허가](https://letsencrypt.org/ko/docs/caa/) 
 

"iptime.org" 의 CAA RR을 확인해 보면 아래와 같이 ";"로 설정되어 있는데 인증기관에서 확인을 할 수 없어서 거부되는걸로 보인다. 

> iptime에서 CAA에 letsencrypt 인증서를 허가하도록 설정해야 하는데 아무래도 안하겠지 !! 

```powershell
❯ wsl dig caa iptime.org
. . . 
;; QUESTION SECTION:
;iptime.org.                    IN      CAA

;; ANSWER SECTION:
iptime.org.             33548   IN      CAA     0 issue ";"
iptime.org.             33548   IN      CAA     0 issuewild ";"
```

한가지 궁금한건 "duckdns.org"를 보면 아래와 같이 CAA RR 정보가 안보이는데 인증서발급이 되는걸 보면 없으면 모두 허용인가? 

```powershell
❯ wsl dig caa duckdns.org
. . .
;; QUESTION SECTION:
;duckdns.org.                   IN      CAA

;; AUTHORITY SECTION:
duckdns.org.            597     IN      SOA     ns3.duckdns.org. hostmaster.duckdns.org. 2020191202 6000 120 2419200 600
```

## 참고 
- [Let's Encrypt 시작하기](https://letsencrypt.org/ko/getting-started/)
- [Let's Encrypt로 무료로 HTTPS 지원하기](https://blog.outsider.ne.kr/1178) 
- [Let's encrypt의 인증서를 생성할 때 주의사항](https://findstar.pe.kr/2018/09/08/lets-encrypt-certificates-rate-limit/)
- [인증서갱신](https://letsencrypt.readthedocs.io/en/latest/using.html#re-creating-and-updating-existing-certificates)
- [Install Let's Encrypt to Create SSL Certificates](https://www.linode.com/docs/guides/install-lets-encrypt-to-create-ssl-certificates/)

## 이슈
- [Ubuntu 20.04 LTS on WSL](https://github.com/jitsi/jitsi-meet/issues/6356)
