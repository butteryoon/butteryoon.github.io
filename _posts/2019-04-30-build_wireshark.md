---
layout: post
title: "Build tshark"
img: "M_wireshark.jpg"
date: 2019-04-30 15:32:00 +0900
tags: [packet, tshark] # add tag
categories: dev
---

## 리눅스에서 tshark only 컴파일

설치환경 

```
Red Hat Enterprise Linux Server release 6.7 (Santiago)
wireshark-2.6.8.tar.xz
```

빌드시도 
> tshark 만 빌드하려고 cmake 옵션 설정 후 make 
> 1차 실패 .. 

``` 
# cmake -DBUILD_wireshark=OFF -DBUILD_tshark=ON
# make 

...
[ 16%] Making dissectors.c
Traceback (most recent call last):
  File "/usr/local/src/wireshark-2.6.8/tools/make-regs.py", line 123, in <module>
    make_dissectors(outfile, infiles)
  File "/usr/local/src/wireshark-2.6.8/tools/make-regs.py", line 65, in make_dissectors
    output += gen_prototypes(protos)
  File "/usr/local/src/wireshark-2.6.8/tools/make-regs.py", line 24, in gen_prototypes
    output += "void {}(void);\n".format(f)
ValueError: zero length field name in format
make[2]: *** [epan/dissectors/dissectors.c] Error 1
make[1]: *** [epan/dissectors/CMakeFiles/dissectors.dir/all] Error 2
make: *** [all] Error 2
```




## 참고 URL
- [Tshark Command Examples](https://linuxsimba.com/tshark-examples)
- [Analyze HTTP Requests With TShark](https://kvz.io/blog/2010/05/15/analyze-http-requests-with-tshark/)
- [Dump and analyze network traffic](https://explainshell.com/explain?cmd=tshark++-d+udp.port%3D%3D8472%2Cvxlan+-r+1.cap+"tcp.analysis.duplicate_ack_num%3D%3D1")
