---
layout: post
title: "RFC 형식의 프로토콜 다이어그램 그리기"
description: "RFC 문서의 프로토콜 헤더 다이어그램을 그려주는 오픈소스 툴인 protocol을 설치하고 커스텀 다이어그램을 만들어본다."
img: protocol_title.png
date: 2021-05-09 15:00:00 +0900
last_modified_at: 2021-05-09 15:00:00 +0900
tags: [command-line, rfc, protocol, diagram] # add tag
related: command-line
categories: tools
---

네트워크 통신 프로토콜 다이어그램을 RFC 문서의 프로토콜 헤더 정의와 같이 ASCII 형식으로 그려주는 오픈소스 툴인 **[protocol](http://www.luismg.com/protocol/)**을 설치하고 커스텀 프로토콜 다이어그램을 만들어본다.

자체 정의된 프로토콜 정보를 헤더파일에 넣는 경우가 있는데 기존에는 **[DrawIt](https://www.vim.org/scripts/script.php?script_id=40)** vim 플러그인을 쓰곤 했는데 protocol로 편하게 그릴 수 있을거 같다. 

<!--more-->

## Install

git로 프로젝트를 Clone하고 **"setup.py install"** 명령어로 설치한다. 

```bash
❯ git clone https://github.com/luismartingarcia/protocol.git
Cloning into 'protocol'...
remote: Enumerating objects: 48, done.
remote: Total 48 (delta 0), reused 0 (delta 0), pack-reused 48
Unpacking objects: 100% (48/48), 42.79 KiB | 1.16 MiB/s, done.

❯ cd protocol
❯ sudo ./setup.py install
running install
running build
running build_scripts
running install_scripts
copying build/scripts-2.7/specs.py -> /usr/local/bin
copying build/scripts-2.7/protocol -> /usr/local/bin
copying build/scripts-2.7/constants.py -> /usr/local/bin
changing mode of /usr/local/bin/specs.py to 755
changing mode of /usr/local/bin/protocol to 755
changing mode of /usr/local/bin/constants.py to 755
running install_egg_info
Writing /usr/local/lib/python2.7/dist-packages/protocol-0.1.egg-info
```

## 정의된 프로토콜

기본으로 이더넷을 구성하는 TCP/IP 관련 프로토콜들은 **"protocol ethernet"** 과 같은 명령어로 확인할 수 있다. 

```bash
❯ protocol icmp-timestamp
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      Type     |      Code     |            Checksum           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|           Identifier          |        Sequence Number        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                      Originate Timestamp                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Receive Timestamp                       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Transmit Timestamp                      |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

## 커스텀 프로토콜 

자체 정의된 프로토콜은 **"name:length"** 형식의 파라미터를 콤마(,)로 부분해서 입력할 수 있고 기본형식은 "+-+-+"와 같이 "-,+" 문자가 번갈아가며 그려지는데 아래의 옵션으로 형식을 바꿀수도 있다. 

- --evenchar  <char>  : Character for the even positions of horizontal table borders.  
- --oddchar   <char>  : Character for the odd positions of horizontal table borders.  
- --startchar <char>  : Character that starts horizontal table borders.    
- --endchar   <char>  : Character that ends horizontal table borders.    
- --sepchar   <char>  : Character that separates protocol fields.   



```bash
❯ protocol --oddchar "-" --startchar "+" --endchar "+" \
        "llMagicNumber:64,llTID:64,usSvcID:16,usResult:16,usBodyLen:16,usReserved:16"

 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+---------------------------------------------------------------+
|                                                               |
+                         llMagicNumber                         +
|                                                               |
+---------------------------------------------------------------+
|                                                               |
+                             llTID                             +
|                                                               |
+---------------------------------------------------------------+
|            usSvcID            |            usResult           |
+---------------------------------------------------------------+
|           usBodyLen           |           usReserved          |
+---------------------------------------------------------------+
```

## 참고

- [protocol github](https://github.com/luismartingarcia/protocol){:target="_blank"}
