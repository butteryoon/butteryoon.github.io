---
layout: post
title: "ssh 공개키 로그인에서 Permission denied (publickey,password) 오류 해결"
description: "페이지 설명"
img: M_ssh_tunneling.jpg
date: 2021-01-12 13:00:00 +0900
last_modified_at: 2021-01-12 13:00:00 +0900
tags: [ssh, publickey] # add tag
related: ssh
categories: tools
---

ssh 서버에 공개키를 사용해서 로그인할 때 **"Permission denied (publickey,password)."**오류가 발생하면서 로그인이 안되는 경우가 있다. 

물론 패스워드 방식으로는 로그인이 잘 되는데 공개키로 로그인이 안되는 경우의 해결 방안이다.  
<!--more-->

퍼블릭키를 이용해서 패스워드 없이 로그인 기능을 사용하기 위해서는 SSHD 서버가 키를 찾을 수 있도록 디렉토리 퍼미션을 설정해 주어야 한다.  

> .ssh : 700   
> authorized_keys : 600   

## 계정 퍼미션 확인

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

로그인 하려는 계정의 퍼미션을 "700"으로 변경하면 오류 없이 로그인 된다. 

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

## SSH 연결 설정 파일 구성

아래와 같이 호스트별로 다른 키를 사용하고 있는 경우 ssh 클라이언트가 사용할 키를 선택할 수 있도록 "~/.ssh/config"파일에 각 호스트의 개인키를 등록한다. 

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