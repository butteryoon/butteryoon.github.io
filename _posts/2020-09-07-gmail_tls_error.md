---
layout: post
title: "GMail 다른주소에서 메일보내기 오류"
img: "gmail_error-01.png"
date: 2020-09-07 13:45:00 +0900
tags: [gmail, tls] # add tag
categories: tools
---

회사의 메일은 whois에서 호스팅을 하고 지메일과 연동해서 메일을 쓴다. 

메일을 쓸 때는 다른주소에서 메일 보내기 설정으로 사용하고 있는데 오늘 갑자기 아래와 같은 오류가 발생한다. 

```
"TLS Negotiation failed, the certificate doesn't match the host."
``` 

몇번 재시도 및 일정 시간 지난 후에도 동일한 현상이 생겨 혹시나 하고 다른 주소에서 메일 보내기 설정을 다시 해보았다. 

지메일에서 설정 메뉴에서 **"설정 - 모든설정보기 - 계정 및 가져오기 - 다른 주소에서 메일 보내기 - 정보 수정"** 에서 정보를 다시 입력 후 저장. 

이후 메일 보내기 성공. 

일시적 오류인건지 모르겠지만 동일한 오류가 발생하면 한번 재설정이 해보는 것도... 


## 참고 URL 

- [TLS Negotiation failed the certificate doesn’t match the host](https://bobcares.com/blog/tls-negotiation-failed-the-certificate-doesnt-match-the-host/)