---
layout: post
comments: true
title: "WSL 리눅스에서 메일 보내기"
description: "WSL Ubuntu 터미널에서 msmtp + mutt로 Gmail SMTP를 통해 메일을 보내는 방법"
img: command-title.webp
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-15 00:00:00 +0900
tags: [wsl, mail, msmtp, mutt]
related: mail
categories: tools
redirect_from:
  - /tools/2026/07/14/sendmail_on_wsl.html
---

2020년에 mutt 설치 로그만 붙여놓고 방치했던 초안을 보완해서 발행한다. WSL Ubuntu 터미널에서 스크립트 결과를 메일로 받고 싶을 때, 로컬에 postfix 같은 MTA를 세팅하는 것보다 **msmtp로 Gmail SMTP에 릴레이**하는 쪽이 훨씬 간단하고 요즘 방식이다.
<!--more-->

## 왜 msmtp인가

예전 초안에서는 mutt를 설치하다가 딸려오는 postfix까지 세팅했는데, WSL에서 postfix를 직접 굴리면 가정용 IP라 대부분 스팸 처리되고 관리도 번거롭다. msmtp는 가벼운 SMTP 클라이언트라 설정 파일 하나로 Gmail 계정을 통해 메일을 보낼 수 있다.

## 설치

```bash
$ sudo apt update
$ sudo apt install msmtp msmtp-mta mutt
...
The following NEW packages will be installed:
  msmtp msmtp-mta mutt libtokyocabinet9
Do you want to continue? [Y/n] Y
```

`msmtp-mta`를 같이 설치하면 `sendmail` 명령이 msmtp로 연결되어 mutt나 cron이 그대로 쓸 수 있다.

## Gmail 앱 비밀번호 발급

Gmail은 일반 비밀번호로는 SMTP 로그인이 안 된다. 구글 계정에 2단계 인증을 켠 뒤 [앱 비밀번호](https://myaccount.google.com/apppasswords){:target="_blank"} 페이지에서 16자리 앱 비밀번호를 만들어 쓴다.

## msmtp 설정

`~/.msmtprc` 파일을 만든다.

```
defaults
auth           on
tls            on
tls_starttls   on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.msmtp.log

account        gmail
host           smtp.gmail.com
port           587
from           myid@gmail.com
user           myid@gmail.com
password       발급받은앱비밀번호16자리

account default : gmail
```

비밀번호가 들어 있으니 권한을 꼭 조인다.

```bash
$ chmod 600 ~/.msmtprc
```

## 메일 보내기

msmtp 단독으로도 보낼 수 있고,

```bash
$ echo -e "Subject: WSL test mail\n\n백업 스크립트 완료" | msmtp jsyoon@example.com
```

첨부파일이 필요하면 mutt를 쓴다. `~/.muttrc`에 한 줄만 추가하면 mutt가 msmtp로 발송한다.

```
set sendmail = "/usr/bin/msmtp"
set use_from = yes
set realname = "ButterYoon"
set from = "myid@gmail.com"
```

한 줄 발송 예제.

```bash
$ echo "본문 내용" | mutt -s "제목" -a result.log -- jsyoon@example.com
```

cron 작업 끝에 붙여두면 WSL에서도 작업 결과를 메일로 받아볼 수 있다.

## 참고

- [msmtp - ArchWiki](https://wiki.archlinux.org/title/Msmtp){:target="_blank"}
- [Google 앱 비밀번호로 로그인](https://support.google.com/accounts/answer/185833){:target="_blank"}
- [Mutt 공식 사이트](http://www.mutt.org/){:target="_blank"}
