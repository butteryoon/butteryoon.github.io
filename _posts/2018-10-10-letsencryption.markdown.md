---
layout: post
title: "letsencrypt 인증서 사용"
date:   2018-10-10 20:05:00 +0900
categories: dev
---

lets encrypt 에서 인증서를 받기 위한 절차를 적어본다. 

1. 인증서를 받기 위해 도메인이 있어야 한다. 
2. 메뉴얼 모드로 인증서를 받는다. 
3. 인증서는 90마다 갱신해야 한다. 
4. 끝. 

git 저장소에서 letsencrypt를 설치한다. 

```
git repository : origin https://github.com/letsencrypt/letsencrypt
```

설치 후 인증서만 다운로드 하기 위해 아래의 명령어를 실행. 

```
[lcsapp@localhost letsencrypt]$ ./letsencrypt-auto certonly --manual
Requesting to rerun ./letsencrypt-auto with root privileges...
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): iws.iptime.org
Cert is due for renewal, auto-renewing...
Renewing an existing certificate
Performing the following challenges:
http-01 challenge for iws.iptime.org

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
임시로 해당 문서를 생성. 

```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

h70rfIM6APUpoMV-1afecuZfUgBw3Bdy4EMq0FftQI4.......tyLmv3IZf3ebDBsJhI

And make it available on your web server at this URL:

http://iws.iptime.org/.well-known/acme-challenge/h70rfIM6APUpoMV-......4EMq0FftQI4

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -git repository : origin https://github.com/letsencrypt/letsencrypt
```

끝. 
성공했다는 응답
지정된 디렉토리에 개인키와 공개키가 생성된다. 

```
Press Enter to Continue
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/iws.iptime.org/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/iws.iptime.org/privkey.pem
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

※ 참고 [Lets' Encrypt로 무료로 HTTPS 지원하기](https://blog.outsider.ne.kr/1178) 
