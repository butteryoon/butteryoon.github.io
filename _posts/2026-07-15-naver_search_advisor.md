---
layout: post
comments: true
title: "네이버 서치어드바이저에 사이트 등록하기"
description: "네이버 서치어드바이저에 사이트를 등록하고 소유확인, 사이트맵 제출까지 진행하는 방법"
img: search-title.jpg
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-15 00:00:00 +0900
tags: [naver, search advisor, seo] # add tag
related: search
categories: tools
redirect_from:
  - /tools/2026/07/14/naver_search_advisor.html
---

2020년에 초안으로 남겨두었던 글을 보완해서 발행한다. 네이버 검색에 블로그가 노출되도록 네이버 서치어드바이저에 사이트를 등록하고, 소유확인과 사이트맵 제출까지 진행하는 과정을 정리한다.

<!--more-->

## 서치어드바이저 접속과 사이트 등록

[https://searchadvisor.naver.com](https://searchadvisor.naver.com){:target="_blank"}에 접속해서 네이버 계정으로 로그인한 뒤, 상단의 **웹마스터 도구**로 들어간다. 사이트 URL 입력란에 `https://`를 포함한 전체 주소를 입력하고 등록한다.

## 사이트 소유확인

등록 직후 소유확인 단계가 진행된다. 두 가지 방법 중 하나를 선택하면 된다.

1. **HTML 파일 업로드**: 제공되는 확인용 HTML 파일(naver로 시작하는 파일명)을 내려받아 사이트 루트 경로에 업로드한다. Jekyll 블로그라면 저장소 루트에 파일을 넣어두면 된다.
2. **HTML 메타태그**: `<meta name="naver-site-verification" content="...">` 형태의 태그를 사이트 `<head>` 영역에 추가한다. 파일 업로드가 번거로운 환경이라면 이 방법이 간편하다.

적용 후 **소유확인** 버튼을 누르면 검증이 완료된다.

## 사이트맵 제출

소유확인이 끝나면 왼쪽 메뉴의 **요청 > 사이트맵 제출**로 이동해서 `sitemap.xml` 경로를 입력하고 확인을 누른다. Jekyll 블로그는 `jekyll-sitemap` 플러그인을 쓰면 사이트맵이 자동 생성된다. RSS를 쓴다면 **요청 > RSS 제출**에서 `feed.xml`도 함께 제출해 두면 좋다. 제출 후 실제 검색 노출까지는 며칠에서 2주 정도 걸릴 수 있다.

## 사이트 간단 체크 도구

서치어드바이저에는 사이트의 검색 최적화 상태를 빠르게 점검해주는 [사이트 간단 체크](https://searchadvisor.naver.com/tools/sitecheck){:target="_blank"} 도구가 있다. URL을 입력하면 robots.txt, 사이트 제목/설명, OpenGraph 태그 등의 상태를 확인해준다.

![사이트 간단 체크 결과]({{site.baseurl}}/assets/img/sitecheck_result.png)

체크 결과에서 사이트 제목·설명이나 OpenGraph 항목이 비어 있다면 `<head>`에 title, description 메타태그와 `og:title`, `og:description` 태그를 채워주자. Jekyll이라면 `jekyll-seo-tag` 플러그인으로 한 번에 해결할 수 있다.

## 참고

- [네이버 서치어드바이저](https://searchadvisor.naver.com){:target="_blank"}
- [네이버 사이트 간단 체크](https://searchadvisor.naver.com/tools/sitecheck){:target="_blank"}
- [웹사이트 네이버 서치어드바이저 등록 (TBWA 데이터랩)](https://seo.tbwakorea.com/blog/naver-search-advisor/){:target="_blank"}
