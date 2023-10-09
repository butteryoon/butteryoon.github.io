---
layout: post
comments: true
title: "ssh 공개키 인증시 Permission denied (publickey,password) 오류 해결"
description: "ssh 공개키 인증시 Permission denied (publickey,password) 오류 해결"
img: M_ssh_tunneling.jpg
date: 2021-01-12 13:00:00 +0900
last_modified_at: 2023-10-10 03:00:00 +0900
tags: [ssh, publickey, authorized_keys] # add tag
related: ssh
categories: tools
---

ssh 서버에 공개키를 사용해서 로그인할 때 **"Permission denied (publickey,password)."**오류가 발생하면서 로그인이 안되는 경우가 있다. 

물론 패스워드 방식으로는 로그인이 잘 되는데 공개키로 로그인이 안되는 경우의 해결이며 몇 가지 설정이 필요하다. 

<!--more-->

## ssh_config 설정

ssh를 퍼블릭키로 접속하기 위해선 sshd 설정에 **PubkeyAuthentication** 인증 설정을 허용하고 sshd 를 재시작 해야 한다. 

```bash
$ cat /etc/ssh/sshd_config | grep Pubkey
PubkeyAuthentication yes
```

## .ssh 디렉토리 퍼미션 확인

공개키 이용 인증 설정 적용 후 **sshd** 서버가 키를 찾을 수 있도록 ssh 클라이언트 실행을 위한 디렉토리 퍼미션을 설정되어 있어야 한다. 

> chmod 700 ~/.ssh
> chmod 600 ~/.ssh/authorized_keys

위의 퍼미션 정보에 따라 ssh 서버의 .ssh 디렉토리의 퍼미션을 "700"으로 변경한다. 

.ssh 디렉토리 퍼미션 변경 후에도 퍼미션 오류가 계속 발생하는 경우 아래 디렉토리 퍼미션 처럼 **..** 상위 디렉토리(사용자의 홈 디렉토리) 퍼미션도 700으로 변경해야 한다.  

```bash
[butteryoon@DEV .ssh]$ ls -al
합계 20
drwx------  2 butteryoon butteryoon 4096  1월 12 12:35 .
drwxrwxr-x 39 butteryoon butteryoon 4096  1월 12 12:35 ..
-rw-------  1 butteryoon butteryoon  397  1월 12 12:35 authorized_keys
-rw-------  1 butteryoon butteryoon 3243 11월 19  2018 id_rsa
-rw-r--r--  1 butteryoon butteryoon  738 11월 19  2018 id_rsa.pub
```

## 계정 퍼미션 700으로 변경  

서버에서 로그인 하려는 계정의 퍼미션을 "700"으로 변경하면 오류 없이 로그인 된다. 

> 퍼미션 설정시 755 또는 750 으로 group/other가 write 권한이 없으면 공개키인증을 사용할 수 있다. 


```bash
[butteryoon@DEV .ssh]$ chmod 700 ../
[butteryoon@DEV .ssh]$ ls -al
합계 20
drwx------  2 butteryoon butteryoon 4096  1월 12 12:35 .
drwx------ 39 butteryoon butteryoon 4096  1월 12 12:35 ..
-rw-------  1 butteryoon butteryoon  397  1월 12 12:35 authorized_keys
-rw-------  1 butteryoon butteryoon 3243 11월 19  2018 id_rsa
-rw-r--r--  1 butteryoon butteryoon  738 11월 19  2018 id_rsa.pub
```

## SSH 클라이언트 연결 설정 파일

아래와 같이 호스트별로 다른 키를 사용하고 있는 경우 ssh 클라이언트가 사용할 키를 선택할 수 있도록 **~/.ssh/config** 파일에 각 호스트의 개인키를 등록한다. 

```powershell
❯ cat C:\Users\softr\.ssh\config
#
# GitHub
#
Host github.com
        HostName github.com
        PreferredAuthentications publickey
        IdentityFile ~/.ssh/butteryoon_rsa
#
# bitbucket
#
Host bitbucket.org
        HostName bitbucket.org
        PreferredAuthentications publickey
        IdentityFile ~/.ssh/yoon_note_rsa
#
# gitlab
#
Host gitlab.com
        HostName gitlab.com
        PreferredAuthentications publickey
        IdentityFile ~/.ssh/yoon_note_rsa
#
# IWS-DEV
#
Host iws103
        HostName 192.168.0.103
        User butteryoon
        PreferredAuthentications publickey
        IdentityFile ~\.ssh\id_rsa
```

## 참고 URL
- [SSH 사용중 ssh-keygen -t rsa, ssh-keygen -t dsa 사용중 오류에 대한 증상](https://nabiro.tistory.com/40){:target="_blank"}
