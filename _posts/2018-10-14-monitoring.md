---
layout: post
title: "sar 를 이용한 리눅스 시스템 모니터링"
description: "sar를 이용한 리눅스 시스템 기본 항목인 CPU, Memory, Swap 사용량 및 NIC의 트래픽량을 실시간으로 표시하고 awk를 이용해서 출력을 바꿔본다."
img: monitoring.jpg
date: 2018-10-14 09:00:01 +0900
last_modified_at: 2021-04-25 20:00:01 +0900
tags: [command-line, linux, top, sar, awk, netstat, ss] # add tag
related: command-line
categories: dev
---


시스템이 행업 상태에 빠질 때가 있다.   
콘솔에 응답이 없는 상황이라 재부팅 외에는 방법이 없기 때문에 장애 발생 시점의 현장보존(?)을 할 수 없다.  
중요한 건 부팅이 되고 터미널이 다시 연결되어도 막상 수집할 수 있는 정보가 많지 않다는 거(아니 많긴 한데.. 알 수 없는 정보가 너무 많다.)  
"Kernel Crash dump"가 설정되어 있다면 혹시 모르지만 그것도 안되어 있는 경우가 많다. 

그래서 정리해본다.
<!--more-->

**시스템 상태 정보** 를 얻을 수 있는 시스템 명령어는 대략 아래와 같다. 

* top 
* vmstat
* iostat
> [io/vmstat examples](https://www.thegeekstuff.com/2011/07/iostat-vmstat-mpstat-examples/?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed%253A+TheGeekStuff+%2528The+Geek+Stuff%2529) 
* netstat 
> 요즘은 `ss` 명령어를 사용한다.

```bash
$ netstat -anpo | grep PROCESS_NMAE 

$ ss -anpo | grep ESTAB 
``` 

## sar 명령어  

- sar 명령어로는 특정시점의 시스템자원 사용률에 대한 정보를 얻을 수 있다. 
- sar 정보는 crontab 에 보통 10분간격으로 통계값을 sar일자 파일로 저장되고 저장된 파일에서 특정 시간범위의 값을 얻을 수 있다. 
- 보통 리눅스를 설치하면 기본으로 설정되어 있으며 보관기간은 한달 정도. ※ /etc/cron.d/sysstat 파일을 참고
- sar 명령시 제일 앞에 시간이 보이지 않으면 "export LANG=C" 후 실행.  
> LANG 변수가 euc-kr이면 앞의 시간이 안보이는 경우가 있다.  
> 아래의 awk 인자의 순서는 모두 제일 앞에 시간 필드(HHYYSS PM)가 있는 경우로 가정한다. 


### 개별 CPU Usage   

분석 일자의 09시 ~ 14시 까지의 CPU 사용율 중 %user 값이 10이 넘는 줄만 출력한다. 

```bash
$ sar -P ALL  -u -f /var/log/sa/sa16 -s 09:00:00 -e 14:00:00 | awk '{if ( $4 > 10 ) print $0}' 

06:20:01 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
06:30:01 PM      44     52.25      0.00     47.75      0.00      0.00      0.00
06:40:01 PM       5     99.29      0.00      0.70      0.00      0.00      0.00
06:40:01 PM      44     52.27      0.00     47.73      0.00      0.00      0.00
06:50:01 PM       1     19.69      0.01      0.28      0.12      0.00     79.91
06:50:01 PM       5     73.75      0.00      0.55      0.00      0.00     25.70
06:50:01 PM      44     52.22      0.00     47.78      0.00      0.00      0.00
```

### 개별 CPU Usage (Top 10)

awk를 이용해서 특정값 보다 큰 row만 출력 하고 sort를 이용해서 CPU Usage top10을 출력한다. 

```bash
$ sar -P ALL -u -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 | awk '{if ( $4 > 1 ) print $0}' | sort -k4 | tail -n 10
```

### MEM Usage 

"sar"로 특정 시간대의 메모리 사용량을 출력하고 awk를 이용하여 GB단위로 표시한다. 

리눅스의 실제 메모리 사용량은 "used - ( buffer + cache )"로 계산할 수 있다. 

> 리눅스에서 buffer와 Cache는 디스크 오퍼레이션의 속도를 높이기 위해 파일시스템이나 블럭디바이스의 정보를 유지하며 응용프로그램의 메모리가 부족하면 해제하여 사용한다.  

```bash
$ sar -P ALL -r -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 | awk '{print $0" "($3-$5-$6)/1024/1024 GB}' 

09:00:01 AM kbmemfree kbmemused  %memused kbbuffers  kbcached  kbcommit   %commit 0
09:10:01 AM  77599712  54518952     41.27    352728  25548668  88329836     53.32 27.2918
09:20:01 AM  77521932  54596732     41.32    353064  25556716  96151668     58.04 27.358
09:30:01 AM  77512192  54606472     41.33    353368  25564336  96153916     58.04 27.3597
09:40:01 AM  77572824  54545840     41.29    353680  25573688  88333104     53.32 27.2927
09:50:01 AM  77695988  54422676     41.19    353968  25597760  70013052     42.26 27.152
10:00:01 AM  77694280  54424384     41.19    354000  25598284  70012988     42.26 27.1531
```

### SWAP Usage 

SWAP 사용량 조회 및 1초 단위 실시간 모니터링   

```bash
$ sar -P ALL -S -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 

09:00:01 AM kbswpfree kbswpused  %swpused  kbswpcad   %swpcad
09:10:01 AM  33554428         0      0.00         0      0.00
09:20:01 AM  33554428         0      0.00         0      0.00
09:30:01 AM  33554428         0      0.00         0      0.00
09:40:01 AM  33554428         0      0.00         0      0.00

$ sar -P ALL -S 1  
```

### Network device Usage

bond0 인터페이스의 사용량을 출력하고 Mbps 단위로 바꿔서 출력해본다. 

> mbps = kilobytes * 8 / 1024  

```bash
$ sar -n DEV -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 | egrep 'IFACE|bond0' | awk '{print $0" "$6*8/1024" Mbps"}' 

05:00:01 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s 0 Mbps
05:10:01 PM       em2    260.68     70.86    325.09     55.86      0.00      0.00      3.79 2.53977 Mbps
05:20:01 PM       em2    293.26    224.44    327.45    217.86      0.00      0.00      2.90 2.5582 Mbps
05:30:01 PM       em2    255.16     26.82    326.75      5.61      0.00      0.00      3.16 2.55273 Mbps
05:40:01 PM       em2    275.89     85.13    334.88     66.88      0.00      0.00      3.00 2.61625 Mbps
```

### 네트워크 디바이스 에러 조회 

```bash
$ sar -n EDEV -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 | egrep 'IFACE|bond0' 

05:00:01 PM     IFACE   rxerr/s   txerr/s    coll/s  rxdrop/s  txdrop/s  txcarr/s  rxfram/s  rxfifo/s  txfifo/s 0 Mbps
05:10:01 PM       em2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00 0 Mbps
05:20:01 PM       em2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00 0 Mbps
05:30:01 PM       em2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00 0 Mbps
05:40:01 PM       em2      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00 0 Mbps

$ sar -n SOCK -f /var/log/sa/sa16 -s 09:00:00 -e 12:00:00 

05:00:01 PM    totsck    tcpsck    udpsck    rawsck   ip-frag    tcp-tw
05:10:01 PM       825        20        14         0         0         0
05:20:01 PM       825        20        14         0         0         0
05:30:01 PM       825        20        14         0         0         0
05:40:01 PM       825        20        14         0         0         0
```

## 실시간 모니터링  

sar로 디바이스별 트래픽 정보를 초단위 간격으로 모니터링할 수 있다. 

```bash
$ sar -n DEV 1 
Linux 2.6.32-573.el6.x86_64 (HOSTNAME)       06/11/2019      _x86_64_        (56 CPU)

05:08:27 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
05:08:28 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00
05:08:28 PM       em1      0.00      0.00      0.00      0.00      0.00      0.00      0.00
05:08:28 PM       em2    273.00     14.00    360.77      1.09      0.00      0.00      7.00
05:08:28 PM       em3      0.00      0.00      0.00      0.00      0.00      0.00      0.00
05:08:28 PM       em4      0.00      0.00      0.00      0.00      0.00      0.00      0.00
```

em2 인터페이스의 정보를 1초 단위로 표시하고 awk를 이용하여 rx 트랙픽($6)을 Mbps로 표시 한다.  

```bash
$ sar -n DEV 1 | egrep --line-buffered 'em2|IFACE' | awk '{print $0" "$6*8/1024" Mbps"}' 

07:23:32 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s
07:23:32 PM       em2    254.55     16.16    339.36      1.49      0.00      0.00      1.01 2.65125 Mbps
07:23:33 PM       em2    222.77     12.87    299.01      1.35      0.00      0.00      0.99 2.33602 Mbps
07:23:34 PM       em2    247.47     12.12    335.28      1.32      0.00      0.00      0.00 2.61937 Mbps
07:23:35 PM       em2    333.00    170.00    336.45     73.09      0.00      0.00     10.00 2.62852 Mbps
```

## 참고

- [SAR(System Activity Reporter)를 이용한 시스템 모니터](http://www.cubrid.com/CUBRIDwiki/71317)
- [sar 이용하여 시스템 모니터링하기](http://wiki.tunelinux.pe.kr/pages/viewpage.action?pageId=884938&desktop=true)
- [리눅스 서버 60초안에 상황파악하기](https://b.luavis.kr/server/linux-performance-analysis)
- [Red Hat Enterprise Linux에서 대역폭 감시하기](http://web.mit.edu/rhel-doc/4/RH-DOCS/rhel-isa-ko-4/s1-bandwidth-rhlspec.html)
- [Buffer and Cache](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/5/html/tuning_and_optimizing_red_hat_enterprise_linux_for_oracle_9i_and_10g_databases/chap-oracle_9i_and_10g_tuning_guide-memory_usage_and_page_cache#:~:text=Linux%20always%20tries%20to%20use,which%20saves%20I/O%20operations.)
