---
layout: post
title: "리눅스 터미널 명령어 라인 사용하기"
description: "리눅스서버 관리를 위한 유용한 명령어 라인 툴의 사용법을 정리해본다."
img: command.png
date: 2019-06-07 04:00:00 +0900
last_modified_at: 2020-05-06 18:00:00 +0900
tags: [command-line, linux, bash, script] # add tag
related: command-line
categories: dev
---

리눅스서버 운영에 도움을 줄 수 있는 명령행 툴의 사용법을 기능별로 정리해본다. 리눅스에 기본으로 포함되서 배포되는 툴들도 있지만 별도로 설치해야 하는 툴들도 있다. 

<!--more-->

## date 명령어를 이용한 날짜 변환  

- 유닉스 time_t 시간 값을 날짜형식의 포맷으로 표현하기   

```bash
$ date -d @1553509896 "+%Y-%m-%d %H:%M:%S (%u)"
2019-03-25 19:31:36 (1)
```

- 날짜형식을 time_t 로 표현하기.  

```bash
$ date -d "2019-03-25 19:31:36" +%s
1553509896
```

## xargs 를 이용한 N개의 파일 처리

- N개의 파일에서 특정 message를 시간순으로 표시하기 

```bash
ls -tr debuglog.1[0-9] | xargs -I{} grep messages {} | less 
```

## tar를 이용한 파일관리

- tar를 이용한 디렉토리 복사 

> N대의 서버에 동일한 파일을 복사해야 할 경우 tar와 ssh를 이용해서 파일의 속성 변경없이 파일을 복사한다.  

```bash
$ tar cvf - * | ssh root@[Target address] tar xf - -C [Target directory] 
$ tar cvf - server.c | ssh lcsapp@192.168.0.103 tar xvf - -C /LCS/APP/WEBAPP/public/packages
``` 

- 동일 서버에서 속성을 유지하면서 디렉토리를 복사할 때  

```bash
$ tar cvf - * | tar xvf - -C ../DATA_COPY/
```

## 사용자 계정으로 1024보다 작은 포트에 바인딩 할 수 있도록 설정 

node.js 인경우 HTTPS 포트를 443으로 설정하려면 root 권한이 필요하다. 

루트가 아닌 서비스 계정으로 구동하려면 아래와 같이 해당 프로세스에 권한을 부여한다.  

```bash
# setcap 'cap_net_bind_service=+ep' /LCS/TOOL/node-v4.6.1-linux-x64/bin/node
```

## awk를 이용한 로그 분석하기 

- awk 에서 N개의 딜리미터를 사용  

> 기본으로 [] 사이에 N개의 딜리미터를 사용할 수 있다.  
> 그런데 실제 text에 [ 와 ] 문자를 딜리미터로 쓸 때는 주의가 필요하다.  
> 내부에서 "-F'[[]]' 와 같이 쓰면 동작하지 않는다  
> 그럴때는 "-F'[][]' 와 같이 사용한다. (왜 그런지 까지는 확인 하지 못함)  

```bash
$ cat test.txt | awk -F'[]:.[]' '{print $1" "$2" "$3" "$4" "$5}'
```

- awk 이전 행의 특정 값과 차이 비교

> 아래의 DATA는 RTP Seq의 텍스트라인으로 구성되어 있다.  

```
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=487, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=488, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=489, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=491, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=492, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=493, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=500, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=501, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=502, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=503, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=504, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=505, Time=2269510870 
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=506, Time=2269510870 
```

> Seq 값을 분리하기 위해 = 와 , 구분자로 분리 : -F[=,]  
> 초기 BASE값은 0으로 설정 : BEGIN {BASE=0};  
> 이전 Seq 값과 5이상 차이나면 출력 : {if (GAP>5 || BASE>$6) print $0" "GAP" "BASE};  
> 이후 Seq 값을 BASE 변수에 저장 : {BASE=$6}  

```bash
$ cat data.set | awk -F[=,] 'BEGIN {BASE=0}; {GAP=$6-BASE}; {if (GAP>5 || BASE>$6) print $0" "GAP" "BASE}; {BASE=$6}'  
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=487, Time=2269510870  487 0	: 첫줄은 무시
PT=MPEG-II transport streams, SSRC=0x5CE8F21A, Seq=500, Time=2269510870  7 493
```

## tshark 패킷 캡쳐  

사용자 계정으로 tshark 을 사용하려면 아래와 같이 tshark에서 이용하는 **dumpcap** 바이너리에 권한을 추가해야 한다.

- [tshark을 이용한 패킷 덤프](https://butteryoon.github.io/dev/2019/05/19/pcketdump.html) 참고.  

```bash
# setcap cap_net_raw,cap_net_admin+eip /usr/sbin/dumpcap
```

## fc 커멘드라인 명령어 수정. 

명령어라인에서 파라미터가 많은 명령어를 내렸을 때 오타가 있거나 수정해야 할 부분이 있을 때 사용한다. "fc" 명령어 단독으로 쓰면 바로 이전 명령어가 에디터로 열리고 수정하여 실행할 수 있고 "fc -e - 4248"과 같이 이전 명령어 히스토리 번호로 이전 명령어를 바로 실행할 수도 있다. 

> "fc -l" 히스토리 목록은 "history" 명령어의 목록과 동일하다. 

```bash 
$ fc -l
 4248  bundle exec jekyll serve --host 0.0.0.0 --port 4000 --no-watch
 4249  fc -l
 4250  man bash
```


```bash
$ fc 4248
 edit command line 
 run 
$ fc -e - 4248 
```

## 참고 URL
- [Bash scripting cheatsheet](https://devhints.io/bash.html){:target="_blank"}
- [Node.js; Error: listen EACCES 0.0.0.0:80](https://geunhokhim.wordpress.com/2016/03/29/nodejs-error-listen-eacces-0-0-0-0-80/){:target="_blank"}
- [아카이브 생성 및 해제(linux tar) 사용법](https://jdm.kr/blog/14){:target="_blank"}
- [tshark을 이용한 패킷 덤프](https://butteryoon.github.io/dev/2019/05/19/pcketdump.html){:target="_blank"}
- [Date time in Linux bash](https://unix.stackexchange.com/questions/85982/date-time-in-linux-bash){:target="_blank"}
- [일정 기간내 수정된 파일 찾기](http://thompsonng.blogspot.com/2020/04/linux-find-last-modified-fie-from-n-day.html){:target="_blank"}
- [Linux and Unix fc command tutorial with examples](https://shapeshed.com/unix-fc/){:target="_blank"}
- [An Illustrated Guide to Some Useful Command Line Tools](https://www.wezm.net/technical/2019/10/useful-command-line-tools/){:target="_blank"}
- [My Favorite One Liners](https://muhammadraza.me/2021/Oneliners/){:target="_blank"}


[Bash]: https://www.gnu.org/software/bash/