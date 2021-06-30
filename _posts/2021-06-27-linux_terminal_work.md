---
layout: post
comments: true
title: "리눅스 터미널 명령어로 서비스로그 분석"
description: "리눅스서버의 서비스로그 분석을 위한 터미널 환경을 및 쉘에서 사용할 수 있는 명령어들을 정리해본다."
img: powershell_title.jpg
date: 2021-06-30 20:00:00 +0900
last_modified_at: 2021-06-30 20:00:00 +0900
tags: [command-line, bash, alias, egrep, regex, tmux, ss, sar] # add tag
related: command-line
categories: tools
---

리눅스 서버에서 동작하는 웹 서버 혹은 백엔드 프로세스의 동작을 로그로 검증해야 할 때 터미널의 명령어를 이용하는 방법과 좀 더 편하고 시간을 줄이기 위한 설정들을 해 보기로 한다. 리눅스에서 bash 쉘 환경이며 *tail, grep, tmux, ss, sar, awk* 등의 명령어를 사용한다. 

<!--more-->

## bash alias  

하나의 서버에서 N개의 서비스가 구동되는 경우 각 서비스의 디버그 로그를 분석할 때 디렉토리를 옮겨가며 실시간으로 트랜잭션 로그를 봐야 할 때가 있다. 서버 구조가 익숙한 경우에는 손이 기억하는 경우가 있지만 그렇지 않은 경우에는 매번 문서를 찾거나 잠시 머릿속을 뒤져야 한다. 그럴 때는 alias를 걸어두자. 

계정의 홈에 설정파일을 만들고 *~/.bash_profile* 파일에 아래와 같이 등록하여 적용한다. 

```bash
if [ -f ~/.aliasrc ]; then
	. ~/.aliasrc
fi
```

기본적인 툴의 옵션을 적용한 자주 사용하는 명령어들은 아래와 같이 파라미터와 같이 alias를 걸어둔다. 

```bash
alias grep='egrep --line-buffered'
alias vi='vim'
alias lsof='lsof -P'
alias listen='ss -t -anpo --listening'
alias estab='ss -t -anpo state established'
alias cpu-stat='sar -u 1 -P all'
alias mem-stat='sar -r 1'
alias nic-stat='sar -n DEV 1 | grep --line-buffered "eno1"'
```

## grep 으로 패턴 찾기 

로그에서 클라이언트 액션에 의한 로그를 찾아야 할 때는 아래와 같이 *grep -E* 파라미터로 regex 패턴을 쓸 수 있다. 

> 보통 egrep 스크립트에 아래의 내용으로 만들어져 있다. 

```bash
[test@localhost ~]$ cat /usr/bin/egrep
#!/bin/sh
exec grep -E "$@"
```

아래와 같은 웹 서버 access 로그가 있을 때 GET 이후 URL 패턴과 파라미터 응답코드를 뽑으려면 아래와 같은 패턴을 쓰면 된다. 

```bash
$ cat access_log | egrep -o ".(GET|POST) ((\/[[:alnum:]_]{1,}){1,}\?([[:alnum:]_-]{1,}\=[[:alnum:]]{1,}.){1,}|(\/[[:alnum:]]{1,}){1,}) (HTTP\/[01].[01]..[[:digit:]]{3})"

"GET /inventoryService/inventory/purchaseItem?userId=20253471&itemId=23434300 HTTP/1.1" 500
"GET /inventory_Service/inventory/purchaseItem?userId=20253471&itemId=23434300 HTTP/1.1" 500
"GET /inventory_Service/inventory/purchaseItem?user_Id=20253471&itemId=23434300 HTTP/1.1" 500
"GET /inventory_Service/inventory/purchaseItem?user-Id=20253471&itemId=23434300 HTTP/1.1" 500
```

실시간으로 해당 명령어를 실행하여 URL을 확인하려면 아래의 명령어를 aliasrc 파일에 등록하여 사용한다. 

tail -f 로 실시간으로 쌓이는 로그를 확인할 때는 egrep *--line-buffered* 옵션을 사용하여 라인단위로 출력되도록 한다. 

```bash
alias acclog='tail -F $APACHE_HOME/logs/access_log | egrep --line-buffered -o ".(GET|POST) ((\/[[:alnum:]_]{1,}){1,}\?([[:alnum:]_-]{1,}\=[[:alnum:]]{1,}.){1,}|(\/[[:alnum:]]{1,}){1,}) (HTTP\/[01].[01]..[[:digit:]]{3})"'
```

아래와 같이 파라미터를 제외한 URL 패턴만 검색하여 요청개수 등을 확인할 때는 아래와 같이 sort 와 uniq 명령어를 사용한다. 

```bash
$ cat access.log | egrep -o "(GET|POST) ((\/[[:alnum:]_]{1,}){1,})" | sort | uniq -c
   1 GET /inventoryService/inventory/purchaseItem
   3 GET /inventory_Service/inventory/purchaseItem
```

## tmux

모니터링시 여러개의 터미널을 열고 상황을 확인해야 할 때 **tmux**를 쓰면 터미널을 용도에 맞게 분할할 수 있고 세션으로 관리되어 필요할 때 구성 그대로 불러올 수 있다. 

> 기본 명령어는 [How to use tmux](https://www.networkworld.com/article/3545370/how-to-use-tmux-to-create-a-multi-pane-linux-terminal-window.html){:target="_blank"} 를 참고한다. 

명령어는 복잡하지 않지만 자주쓰는 항목(New, kill, attach)은 alias로 등록해두고 쓴다. 

> tmux-new : 세션 이름을 입력하면 기본 3개의 창으로 분할하여 세션을 만든다.  
> tmux-run : 현재 세션 목록을 보여주고 세션 이름을 입력하면 해당 세션을 실행한다.    
> tmux-del : 현재 세션 목록을 보여주고 세션 이름을 입력하면 해당 세션을 지운다.  

```bash
# tmux new-session with 3 pane 
alias tmux-new='func() { echo -n "INPUT SESSION NAME : "; read SESSION_NAME; tmux new-session -s ${SESSION_NAME} \; split-window -h \; split-window -v \; attach; }; func'
# tmux kill-session 
alias tmux-del='func() { tmux ls; echo -n "INPUT SESSION NAME : "; read SESSION_NAME; tmux kill-session -t ${SESSION_NAME} ; }; func'
# tmux attach-session 
alias tmux-run='func() { tmux ls; echo -n "INPUT SESSION NAME : "; read SESSION_NAME; tmux attach-session -t ${SESSION_NAME} ; }; func'
```

## Socket Statistics

네트워크 LISTEN 또는 ESTABLISHED 상태의 클라이언트를 찾을 때 ss(socket statistics) 유틸리티를 사용한다. 

> -t : TCP Socket   
> -a : 모든 상태 표시  
> -n : IP/PORT를 resolve 하지 않음   
> -o : TCP 프로토콜의 내부 타이머 정보 표시  
> --listening : LISTEN 상태의 TCP   

아래의 상태는 alias로 등록하여 사용한다. 

```bash
alias listen='ss -t -anpo --listening'
alias estab='ss -t -anpo state established'
```

## 리소스 확인 

시스템의 기본 리소스인 CPU, MEM, NIC 트래픽을 초단위 텍스트 로그로 남길 때 아래의 명령어를 alias로 등록하여 사용한다. 

```bash
alias cpu-stat='sar -u 1 -P all'
alias mem-stat='sar -r 1'
alias nic-stat='sar -n DEV 1 | grep --line-buffered "eno1"'
```

리눅스 시스템은 기본으로 *sadc*를 이용하며 10분마다 상태를 기록하며 아래와 같이 크론잡으로 등록되어 있고 필요하면 아래의 주기를 매분 60으로 적용하여 항상 상태를 기록하게 할 수 있다. 

```bash
# sudo cat /etc/cron.d/sysstat
/10 * * * * root /usr/lib64/sa/sa1 1 1
# * * * * * root /usr/lib64/sa/sa1 1 60
```


## 참고

- [How to use tmux to create a multi-pane Linux terminal window](https://www.networkworld.com/article/3545370/how-to-use-tmux-to-create-a-multi-pane-linux-terminal-window.html){:target="_blank"}
- [How to Use the ss Command on Linux](https://www.howtogeek.com/681468/how-to-use-the-ss-command-on-linux/){:target="_blank"}