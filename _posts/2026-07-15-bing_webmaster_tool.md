---
layout: post
comments: true
title: "Bing 웹마스터 도구에 사이트 등록하기"
description: "Bing Webmaster Tools에 블로그를 등록하고 사이트맵을 제출하는 방법"
img: image-title.jpg
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-15 00:00:00 +0900
tags: [bing, webmaster, seo] # add tag
related: jekyll
categories: tools
redirect_from:
  - /tools/2026/07/14/bing_webmaster_tool.html
---

2020년에 초안으로 남겨두었던 글을 보완해서 발행한다. Microsoft Bing 검색에 블로그가 노출되도록 Bing 웹마스터 도구(Bing Webmaster Tools)에 사이트를 등록하고 사이트맵을 제출하는 과정을 정리한다.

<!--more-->

## Bing 웹마스터 도구 접속

[https://www.bing.com/webmasters](https://www.bing.com/webmasters){:target="_blank"}에 접속해서 로그인한다. Microsoft 계정 외에 Google, Facebook 계정으로도 로그인할 수 있다.

## 사이트 등록

사이트를 추가하는 방법은 두 가지다.

![Bing 사이트 추가]({{site.baseurl}}/assets/img/m_bing_addsite01.webp)

### 1. Google Search Console에서 가져오기 (추천)

이미 구글 서치 콘솔에 사이트를 등록해 두었다면 이 방법이 가장 간단하다. **가져오기(Import)** 버튼을 누르고 Google 계정으로 로그인해 권한을 허용하면, 서치 콘솔에 등록된 사이트 목록이 나온다. 가져올 사이트를 선택하면 별도의 소유 확인 절차 없이 등록이 끝난다. 서치 콘솔에 제출해 둔 사이트맵도 함께 가져온다.

### 2. 수동으로 추가하기

수동 등록을 선택하면 사이트 URL을 입력한 뒤 소유 확인을 진행해야 한다. XML 파일(BingSiteAuth.xml) 업로드, HTML 메타태그 삽입, DNS CNAME 레코드 추가 중 하나를 선택할 수 있다. GitHub Pages 블로그라면 메타태그를 `<head>`에 추가하는 방법이 편하다.

![Bing 사이트 추가 완료]({{site.baseurl}}/assets/img/m_bing_addsite02.webp)

## 사이트맵 제출

등록이 끝나면 대시보드 왼쪽 메뉴의 **사이트맵(Sitemaps)** 에서 `https://내도메인/sitemap.xml` 주소를 제출한다. Google Search Console에서 가져오기로 등록했다면 사이트맵이 이미 들어와 있을 수 있으니 목록만 확인하면 된다. 제출 후 수집까지는 보통 몇 시간에서 며칠 정도 걸린다.

## 마무리

URL 검사 도구로 개별 페이지의 색인 상태를 확인할 수 있고, 검색 성능 리포트에서 노출/클릭 통계도 볼 수 있다. 구글 서치 콘솔에 등록해 둔 사이트라면 가져오기 기능 덕분에 몇 분이면 끝나니 함께 등록해 두자.

## 참고

- [Bing Webmaster Tools](https://www.bing.com/webmasters/about?lang=ko){:target="_blank"}
- [Import sites from Search Console to Bing Webmaster Tools](https://blogs.bing.com/webmaster/september-2019/import-sites-from-search-console-to-bing-webmaster-tools){:target="_blank"}
- [Connect Your Site to Bing Webmaster Tools: 2026 Guide](https://www.promodo.com/blog/how-to-add-website-to-bing-webmaster-tools){:target="_blank"}
