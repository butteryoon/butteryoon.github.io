---
layout: post
comments: true
title: "REST Client for Visual Studio Code"
description: "REST Client for Visual Studio Code"
img: tools-title.jpg
date: 2023-07-22 11:00:00 +0900
last_modified_at: 2023-07-24 11:35:00 +0900
tags: [devtool, Visual Studio Code, vscode, http, restful] # add tag
related: devtool
categories: tools
---

"Visual Studio Code"에서 HTTP API를 테스트하는 방법에 대해 알아본다. 

<!--more-->

## HTTP API 테스트

HTTP API 테스트는 주로 "postman"을 사용하는데 폐쇠망에서 테스트 하다보면 툴 설치가 허가되지 않는 경우들이 있다. "Visual Studio Code"의 "postman" 익스텐션은 로그인을 하지 않으면 쓸수 없어서 다른 클라이언트를 찾아보기로 한다.

## "REST Client" 확장 설치

인터넷이 되는 환경에서는 확장관리 기능(Ctrl+Shift+X)로 간단하게 설치 가능하지만 오프라인 환경에서 설치하려면 "vsix" 파일을 다운로드하여 설치해야 한다. 

> [Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=humao.rest-clie) 페이지에서 [Download Extension](https://marketplace.visualstudio.com/_apis/public/gallery/publishers/humao/vsextensions/rest-client/0.25.1/vspackage) "vsix" 파일을 다운로드 한다. 


다운로드 후에 "VSCode"에서 명령팔레트(Ctrl+shift+P)를 열고 "VSIX"를 검색하여 다운로드 한 파일을 선택해서 설치할 수 있다. 

## REST 테스트 

새 파일을 만들고 "언어모드"를 HTTP로 선택하고 아래와 같이 HTTP METHOD를 적으면 적어주면 "Send Request" 글자가 나타난다. 각 요청은 "###"으로 구분된다. 

> 파일의 확장명을 ".http"로 저장해도 자동으로 인식한다. 

```bash
### Get My Public IP Address

GET  https://api.ipify.org?format=json

### 

GET https://api.ipify.org?format=text HTTP/1.1
```

GET 이나 POST 를 입력하면 바로 위에 "Send Request" 글자가 나타나는데 클릭하면 오른쪽 pane이 열리면서 Response 헤더와 데이터가 표시된다. 

![Send Request]({{site.baseurl}}/assets/img/rest-client.png)

## 참고

- [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client){:target="_blank"}
- [[Visual Studio Code] REST Client를 활용한 REST API 테스트](https://blog.jiniworld.me/144){:target="_blank"}
