---
layout: post
title: "리눅스 명령어 라인"
img: command.png
date: 2018-11-24 04:00:00 +0900
tags: [리눅스, bash, command line] # add tag
categories: dev
---

## 리눅스 Bash 

대부분의 배포판에서 사용하고 있는 GNU [Bash][Bash] 

## 리눅스 터미널 명령어 라인 사용하기


- 다른 서버로 디렉토리 옮기기   

> tar를 이용해서 파일을 묶고 전송한 후 풀어서 저장 
> 장정 : 파일속성이 변하지 않는다. 

```
$ tar cvf - * | ssh root@[Target address] tar xf - -C [Target directory] 
$ tar cvf - server.c | ssh lcsapp@192.168.0.103 tar xvf - -C /LCS/APP/WEBAPP/public/packages
``` 

- 사용자 계정으로 1024보다 작은 포트에 바인딩 할 수 있도록 설정 

> node.js 인경우 HTTPS 포트를 443으로 설정하려면 root 권한이 필요함.
> 서비스 계정으로 구동하려면 아래와 같이 해당 프로세스에 권한을 부여한다.  

```
setcap 'cap_net_bind_service=+ep' /LCS/TOOL/node-v4.6.1-linux-x64/bin/node
```


## 참고 URL
- [Bash scripting cheatsheet](https://devhints.io/bash.html)
- [Node.js; Error: listen EACCES 0.0.0.0:80](https://geunhokhim.wordpress.com/2016/03/29/nodejs-error-listen-eacces-0-0-0-0-80/)


[Bash]: https://www.gnu.org/software/bash/
