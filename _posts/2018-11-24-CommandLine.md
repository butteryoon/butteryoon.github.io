---
layout: post
title: "리눅스 명령어 라인"
img: command.png
date: 2018-11-24 04:00:00 +0900
tags: [리눅스, bash, command line] # add tag
categories: dev
---

## 리눅스 터미널 명령어 라인 사용하기


- tar를 이용한 디렉토리 복사 

> tar를 이용해서 파일을 묶고 전송한 후 풀어서 저장  
> 장점 : 파일속성이 변하지 않는다. 
> N대의 서버에 동일한 파일을 복사해야 할 경우 사용할 수 있다. 

```
$ tar cvf - * | ssh root@[Target address] tar xf - -C [Target directory] 
$ tar cvf - server.c | ssh lcsapp@192.168.0.103 tar xvf - -C /LCS/APP/WEBAPP/public/packages
```
> 동일 서버에서 속성을 유지하면서 디렉토리를 복사할 때  

```
$tar cvf - * | tar xvf - -C ../DATA_COPY/
```

- 사용자 계정으로 1024보다 작은 포트에 바인딩 할 수 있도록 설정 

> node.js 인경우 HTTPS 포트를 443으로 설정하려면 root 권한이 필요함.
> 서비스 계정으로 구동하려면 아래와 같이 해당 프로세스에 권한을 부여한다.  

```
$ setcap 'cap_net_bind_service=+ep' /LCS/TOOL/node-v4.6.1-linux-x64/bin/node
```

- awk 에서 N개의 딜리미터를 사용  

> 기본으로 [] 사이에 N개의 딜리미터를 사용할 수 있다.  
> 그런데 실제 text에 [ 와 ] 문자를 딜리미터로 쓸 때는 쫌 주의가 필요하다.  
> 내부에서 "-F'[[]]' 와 같이 쓰면 동작하지 않는다  
> 그럴때는 "-F'[][]' 와 같이 사용한다. (왜 그런지 까지는 확인 하지 못함)  

```
cat test.txt | awk -F'[]:.[]' '{print $1" "$2" "$3" "$4" "$5}'
```


- tshark 패킷 캡쳐  
> 요건 따로 정리해야 겠다. 


## 참고 URL
- [Bash scripting cheatsheet](https://devhints.io/bash.html)
- [Node.js; Error: listen EACCES 0.0.0.0:80](https://geunhokhim.wordpress.com/2016/03/29/nodejs-error-listen-eacces-0-0-0-0-80/)
- [아카이브 생성 및 해제(linux tar) 사용법](https://jdm.kr/blog/14)
-


[Bash]: https://www.gnu.org/software/bash/
