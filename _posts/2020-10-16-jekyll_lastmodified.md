---
layout: post
title: "Jekyll 블로그에 수정일자 표시하기"
description: "블로그 페이지와 sitemap.xml의 lastmod 태그를 수정일자로 변경"
img: jekyll-title.png
date: 2020-10-16 18:00:00 +0900
last_modified_at: 2020-10-23 01:00:00 +0900
tags: [jekyll, sitemap, last-modified-at] # add tag
related: jekyll
categories: tools
---

Github Pages 블로그에서 글의 수정일자를 표시하고 sitemap.xml 파일의 lastmod 태그를 수정일자로 표시해 보자. 
<!--more-->

## 페이지 변수 추가 

Jekyll은 페이지 머릿말에 사용자 변수를 추가해서 사용할 수 잇다.  

마크다운 파일에 수정일자 변수를 추가해준 다음 post.html에서 타이틀 표시 아래 수정일자를 가져와서 표시하도록 한다. 

last_modified_at 변수 추가 하고 수정일자 추가한다. 

```markdown
date: 2020-10-16 18:00:00 +0900
last_modified_at: 2020-10-22 18:00:00 +0900
```

## post.html 제목아래 수정 일자 표시

"layouts/post.html" 파일의 "page-title" 표시부분 아래 "page.date"를 "page.last_modified_at"으로 바꿔준다. 

{% raw %}
```liquid
<header class="header-page">
  <h1 class="page-title">{{page.title}}</h1>
  {% if page.last_modified_at %}
    <div class="page-date"><span>{{page.last_modified_at | date: '%Y, %b %d'}}&nbsp;&nbsp;&nbsp;&nbsp;</span></div>
  {% else %}
    <div class="page-date"><span>{{page.date | date: '%Y, %b %d'}}&nbsp;&nbsp;&nbsp;&nbsp;</span></div>
  {% endif %}
</header>
```
{% endraw %}

## sitemap.xml 

위와 같이 수정한 후 "sitemap.xml" 파일을 보면 <lastmod> 태그가 수정일자로 변경되어 있다. 

```xml
<url>
<loc>http://0.0.0.0:4000/tools/2020/10/16/jekyll_lastmodified.html</loc>
<lastmod>2020-10-23T01:00:00+09:00</lastmod>
</url>
```

## 참고 URL

- [Adding last modified date to Jekyll](https://tomkadwill.com/adding-last-modified-date-to-jekyll)
* [파일의 마지막 수정일을 sitemap.xml에 사용해보자](https://dev-yakuza.github.io/ko/jekyll/jekyll-last-modified-at/)
