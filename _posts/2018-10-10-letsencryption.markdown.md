---
layout: post
title: "letsencrypt 인증서 사용"
img: "letsencryption.png"
date:   2019-04-20 20:05:00 +0900
tags: [letsencrypt, HTTPS, 인증서] 
categories: dev
---

아이폰 엔터프라이즈 앱의 인하우스 배포를 위해서는 도메인이 필요하다네 .. 
그래서 letsencrypt 무료 인증서를 받아 사용하기로 한다. 
회사 웹서버는 호스팅을 하고 있고 음! 어디서?? 모르겠고 ..
우선 인증서만 받아 인하우스 배포를 시험하기로 한다. 

lets encrypt 에서 인증서를 받기 위한 절차는 아래와 같다. 

생각보다 간단하다. 

1. 인증서를 받기 위해 도메인이 있어야 한다. 
2. 메뉴얼 모드로 인증서를 받는다. 
3. 인증서는 90일 마다 갱신해야 한다. 
4. 끝. 

git 저장소에서 letsencrypt를 clone 하고

```
git repository : origin https://github.com/letsencrypt/letsencrypt
```

설치 후 인증서만 다운로드 하기 위해 아래의 명령어를 실행. 
> git pull로 최신버전 다운로드 이후 아래와 같이 Bootstrap 관련 경고가 발생하여 --no-Bootstrap 옵션 추가 

```
[root@localhost letsencrypt]# ./letsencrypt-auto certonly --manual
Bootstrapping dependencies for RedHat-based OSes... (you can skip this with --no-bootstrap)
yum is /bin/yum
To use Certbot, packages from the EPEL repository need to be installed.
Enable the EPEL repository and try running Certbot again.
[root@localhost letsencrypt]# ./letsencrypt-auto certonly --manual --no-bootstrap
```

> 이후는 동일하게 진행한다(root 계정에서 실행)
> 완료이후 /etc/letsencrypt/live/도메인 디렉토리에 인증서가 생성된다. 

```
[@localhost letsencrypt]# ./letsencrypt-auto certonly --manual --no-bootstrap
Requesting to rerun ./letsencrypt-auto with root privileges...
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): test.domain.com
Cert is due for renewal, auto-renewing...
Renewing an existing certificate
Performing the following challenges:
http-01 challenge for host.iptime.org

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y
```
도메인 입력 후 인증 과정을 거친다. 
letsencrypt에서 아래의 URL로 요청했을 때 지정된 문자열을 응답으로 보내야 한다.  
임시로 해당 경로에 문서를 생성. 
```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

h70rfIM6APUpoMV-1afecuZfUgBw3Bdy4EMq0FftQI4.......tyLmv3IZf3ebDBsJhI

And make it available on your web server at this URL:

http://test.domain.com/.well-known/acme-challenge/h70rfIM6APUpoMV-......4EMq0FftQI4

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
```
웹서버를 호스팅하거나 접근하기 어려운 경우 파이썬으로 간단하게 처리 .. 

별도의 서버가 없어도 파이썬 `SimpleHTTPServer` 모듈로 가능. 
> python 3.6.7 버전에서는 모듈을 찾을 수 없다는 오류가 발생한다. (2.7.x 에서는 Ok) 
> python3 에서 SimpleHTTPServer는 http.server 안에 통합되었다. (http://bit.ly/2Zt39My)

```
python -m SimpleHTTPServer 8000 
or 
python3 -m http.server 8000
``` 

위 명령어를 실행하면 간단하게 8000 번 포트로 웹서버를 구동할 수 있다. 

지정된 디렉토리에 안내된 이름으로 지정된 데이타가 포함된 파일을 만들면 지정된 디렉토리에 개인키와 공개키가 생성된다. 
```
Press Enter to Continue
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/test.domain.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/test.domain.com/privkey.pem
   Your cert will expire on 2019-01-08. To obtain a new or tweaked
   version of this certificate in the future, simply run
   letsencrypt-auto again. To non-interactively renew *all* of your
   certificates, run "letsencrypt-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

공개키와 개인키를 원하는 웹서버에 설치하여 사용한다. 

끝.

참고 
[Lets' Encrypt로 무료로 HTTPS 지원하기](https://blog.outsider.ne.kr/1178) 
[Let's encrypt 의 인증서를 생성할 때 주의사항](https://findstar.pe.kr/2018/09/08/lets-encrypt-certificates-rate-limit/)

[인증서갱신](https://letsencrypt.readthedocs.io/en/latest/using.html#re-creating-and-updating-existing-certificates)

