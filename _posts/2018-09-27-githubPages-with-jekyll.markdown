---
layout: post
title: "jekyll을 이용하여 Github Page 만들기"
img: software.jpg
date: 2018-09-27 21:25:30 +0900
tags: [gitHub, 블로그, jekyll, 마크다운] # add tag
related: jekyll
categories: blog
---

다시 블로그가 궁금해졌다. 

네이버 블로그, 테스토리, 워드프레스 다양한 블로그가 있지만 정작 글 쓰는 것 보다는 만드는 행위 자체에 의미를 두는 경우가 많다. 

[네이버블로그](https://softroom.blog.me) [티스토리](http://butteryoon.tistory.com) 등 블로그를 몇개 쓰기는 하지만 개인 기록 또는 웹 페이지의 동작방법이 궁금해서 끄적였던 거고 .. 

몇가지 조건이라면 `마크다운`을 좀 더 알고 싶고 이왕이면 좀 더 일반적이지 않은 곳 정도.. 

요즘 [GitHub Pages][GitHubPages]를 이용해 프로젝트 웹 페이지가 많이 만들어져 있어 한번 만들어 보기로 한다. ( 순전히 궁금해서 .. ) 


우선 [jekyll][jekyll] 사이트에 친절하게 나와있으니 읽어보자. 

![Jekyll_Logo](https://jekyllrb-ko.github.io/img/logo-2x.png)

> 페이지의 아래 [GitHub Pages 에 대해 더 자세히 알아보세요](https://pages.github.com/) 링크도 읽어 보고 .. 

![gitgub_logo]({{site.baseurl}}/assets/img/octojekyll.png){:height="80px" width="100px"}

> 이미지는 이렇게 첨부하고 .. 
> 음! 윈도우즈 환경에서 하려니 쉽지는 않네 .. (따로 정리해야 겠다) 

**테마적용**

- [jekyll][jekyll] 공식사이트 
- [Themes](https://jekyllrb-ko.github.io/docs/themes/) 페이지 참고
- [테마 적용하기](https://nesoy.github.io/articles/2016-12/github-Jekyll) 블로그 
- 현재는 http://artemsheludko.com/flexible-jekyll/ 블로그의 테마를 적용하고 있다. 
- feed, categorie 기능 제공한다. 
- 파비콘을 적용하려면 [Favicon & App Icon Generator](https://www.favicon-generator.org)을 참고
- [Github-Pages 에 Jekyll 설치하기](http://dveamer.github.io/homepage/JekyllOnGithubPages.html)

**TODO**

1. 새 파일을 만들때 기본 헤더정보를 자동으로 만들어주는 스크립트
 - YYYY-MM-DD-title.md 형식으로 만들고
 - layout, title, desc, img, date, tags, categories 를 입력 받는다. 
 - [포스트작성하기](https://jekyllrb-ko.github.io/docs/posts/)

2. RSS Feed 만들기
 - [플러그인 없이 Jekyll RSS Feed 만들기](http://dveamer.github.io/homepage/RSS-Feed.html) 
  
3. gist Code 템플릿 변경 
 - [Better-Gist](https://gist.github.com/jnrbsn/578379)

## 참고 URL

 - [jekyll 더 알아보기](https://ehfgk78.github.io/2017/12/27/jekyll-detail/)
 - [Prose.io](https://theorydb.github.io/envops/2019/05/04/envops-blog-posting-prose-io/)
 - [Jekyll: Disqus 연결부터 마이그레이션까지](https://xho95.github.io/blog/jekyll/disqus/migration/2017/01/21/Add-Disqus-to-Jekyll.html)
 
[jekyll]: https://jekyllrb-ko.github.io
[GitHubPages]: https://pages.github.com 


