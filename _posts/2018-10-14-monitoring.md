---
layout: post
title:  "리눅스 시스템 모니터링"
img: monitoring.jpg
date: 2018-10-14 03:00:01 +0900
tags: [리눅스, top, sar, awk, netstat] # add tag
categories: dev
---


**리눅스 시스템 자원 확인은 위해 정리**

시스템이 행업 상태에 빠지면 힘들어진다. 사실 명확한 원인을 파악하기는 힘들고 재부팅 외에는 방법이 없다. 

콘솔 상태를 확인해도 검은색 화면에서 좀처럼 벗어날 수 없고 전원버튼을 누르는것 외에는 달리 해 볼 수 있는게 없다. 

중요한 건 부팅이 되고 터미널이 다시 연결되어도 막상 수집할 수 있는 정보가 많지 않다는 거(아니 많긴 한데.. 알 수 없는 정보가 너무 많다.) Kernel Crash dump가 설정되어 있다면 혹시 모르지만 그것도 안되어 있는 경우가 많다. 

그래서 정리해본다.

**시스템 상태 정보** 를 얻을 수 있는 시스템 명령어는 대략 아래와 같다. 

1. top
2. vmstat
3. iostat
 - [io/vmstat examples](https://www.thegeekstuff.com/2011/07/iostat-vmstat-mpstat-examples/?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed%253A+TheGeekStuff+%2528The+Geek+Stuff%2529) 
4. netstat 
 - 요즘은 ss 명령어를 사용한다.

```bash
$ netstat -anpo | grep PROCESS_NMAE 
``` 

5. sar
 - sar 명령어로는 특정시점의 시스템자원 사용률에 대한 정보를 얻을 수 있다. 
 - sar 정보는 crontab 에 보통 10분간격으로 통계값을 sar일자 파일로 저장되고 저장된 파일에서 특정 사간범위의 값을 얻을 수 있다. 
 - 보통 리눅스를 설치하면 기본으로 설정되어 있음.

## 개별 CPU Usage 
```bash
$ sar -P ALL  -u -f /var/log/sa/sa16 -s 09:00:00 -e 14:00:00 | awk '{if ( $3 > 10 ) print $0}'
```

## 개별 CPU Usage (Top 10)
> awk를 이용해서 특정값 보다 큰 row만 출력 하고 CPU Usage top10을 출력.

```bash
$ sar -P ALL -u -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 | awk '{if ( $3 > 1 ) print $0}' | sort -k3 | tail -n 10
```

## MEM Usage 
> 실 메모리 사용량 = used - ( buffer + cache ) 

```bash
$ sar -P ALL -r -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 | awk '{print $0" "($3-$5-$6)/1024/1024 GB}'
```

## SWAP Usage
```bash
$ sar -P ALL -S -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 
```

## Network device Usage
> mbps = kilobytes * 8 / 1024

```bash
$ sar -n DEV -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 | egrep 'IFACE|bond0' | awk '{print $0" "$5*8/1024" Mbps"}'
$ sar -n EDEV -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 | egrep 'IFACE|bond0'
$ sar -n SOCK -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00
```
> 실시간으로 모니터링 

```bash
$ sar -n DEV 1 | egrep --line-buffered 'em2' | awk '{print $0" "$6*8/1024" Mbps"}'
```

## 참고 URL
- [SAR(System Activity Reporter)를 이용한 시스템 모니터](http://www.cubrid.com/CUBRIDwiki/71317)
- [sar 이용하여 시스템 모니터링하기](http://wiki.tunelinux.pe.kr/pages/viewpage.action?pageId=884938&desktop=true)
- [리눅스 서버 60초안에 상황파악하기](https://b.luavis.kr/server/linux-performance-analysis)




