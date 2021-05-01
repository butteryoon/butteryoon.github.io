---
layout: post
title: "Notionlytics : 노션 웹페이지 사용자 추적하기"
description: "Notion 으로 제품소개 페이지를 만들고 Notionlytics 서비스로 기본적인 사용자 추적정보인 페이지뷰, 방문자 통계를 확인해본다."
img: title-idea.webp
date: 2021-04-30 18:00:00 +0900
last_modified_at: 2021-04-31 18:00:00 +0900
tags: [idea, notion, notionlytics] # add tag
related: idea
categories: tools
---

**Notion**은 "웹에서 공유" 기능으로 노션 페이지를 웹 페이지로 만들수 있어 개인 브랜딩 페이지나 임시로 진행하는 이벤트 페이지로 많이 쓰고 있다. 

Notion에서 간단한 제품 소개 페이지를 만들어 웹으로 공유하고 **[Notionlytics](https://app.notionlytics.com){:target="_blank"}** 서비스와 연동하여 페이지뷰 및 사용자 방문 정보를 확인해본다. 
<!--more-->

## 노션 페이지 웹에서 공유하기 

노션으로 만든 페이지는 오른쪽 위의 **공유** 버튼을 눌러 **웹에서 공유** 를 활성화 시키면 URL로 공유할 수 있고 노션 사용자는 댓글을 쓸 수도 있다. 

![share on web]({{site.baseurl}}/assets/img/m_notion_webshare_01.png)

## 노션 웹 페이지 

"웹에서 공유"를 적용하고 **[IWS Live Streaming Platform](https://www.notion.so/softroom/IWS-Live-Streaming-Platform-850f6b837c5a426ead872056dcb81565){:target="_blank"}** 제품소개 페이지의 링크를 열여보면 아래와 같이 댓글, 복제 등의 노션 메뉴가 있는 웹페이지를 확인할 수 있다. 

> 로그인한 노션 유저는 댓글을 달 수 있다. 

![create]({{site.baseurl}}/assets/img/m_notion_webpage.webp)

## 노션리틱스(Notionlytics) 설정

노션리틱스(Notionlytics)서비스는 구글 애널리틱스를 이용해서 기본적인 사용자 통계(페이지뷰, 방문자, 위치)결과를 제공해주며 세부 데이타는 구글 애널리틱스 페이지에서 확인 가능하다. 

### Create 

노션리틱스 서비스는 구글이나 페이스북 계정으로 로그인 후 오른쪽 상단에 **New Tracker** 버튼으로 노션 페이지의 공유 링크를 넣을 수 있다. 

![create]({{site.baseurl}}/assets/img/notionlytics_create.png)

### Connect 

구글계정으로 로그인하면 구글 애널리틱스 정보를 연동하여 트래킹데이타를 저장할 위치를 선택하거나 새로 만들 수 있다. **Create New** 링크로 애널리틱스에 추가 계정을 만들 수 있다. 

노션리틱스에 트래커 코드가 로딩되면 구글 애널리틱스에 추가한 노션 페이지의 이름으로 속성이 추가된다. 

![connect]({{site.baseurl}}/assets/img/notionlytics_connect.png)

### Install

구글 애널리틱스와 연결이 성공하면 "tracker widget code" 링크가 만들어진다. 트래커 링크를 복사한 후 노션 페이지에 붙여넣기 하여 "임베드 생성"을 선택하면 아래와 같이 파란색 이미지링크가 생긴다.

> 노션은 사용자에게 보여지는 부분만 로딩하는 정책이라 "widget code"는 페이지 로딩시 바로 보이는 위치에 넣어야 정확한 추적이 가능하다고 한다. 

![tracker]({{site.baseurl}}/assets/img/notionlytics_image.png)

### Result

노션리틱스의 대시보드의 각 사이트에서 **Result** 링크를 누르면 아래와 같이 페이지뷰 와 세션수를 볼 수 있으며 추가로 사용자, 위치 정보등을 볼 수 있다. 

![stat]({{site.baseurl}}/assets/img/notionlytics_stat.png)

## oopy.io

노션 페이지를 웹사이트로 만들어주는 [Oopy(우피)](https://www.oopy.io){:target="_blank"}라는 서비스가 있는데 노션의 페이지를 "oopy.io" 또는 소유한 도메인으로 연결해주는 서비스를 제공하고 있다. 

소개 페이지에 보면 왓챠, 배민 플랫폼실, 직방 등에서 채용을 위한 별도의 페이지로 이용하고 있는데 채용처럼 이벤트성 페이지를 관리하는 용도로 사용하면 좋아 보인다. 

처음에는 2개의 페이지를 2주간 써 볼 수 있다. 

## 참고

- [Notionlytics](https://app.notionlytics.com){:target="_blank"}
- [노션에서 구글 애널리틱스 사용하기](https://blog.mskim.me/posts/google-analytics-with-notion-so/){:target="_blank"}