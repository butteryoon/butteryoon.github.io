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

Jekyll은 페이지 머리말에 사용자 변수를 추가해서 사용할 수 있고 "page.변수"로 접근해서 쓸 수 있다. 

마크다운 파일에 수정일자 변수를 추가해준 다음 post.html에서 타이틀 표시 아래 수정일자를 가져와서 표시하도록 한다. 

※ 사용자 변수를 써도 되지만 "last-modified-at" 변수를 쓰면 'jekyll-sitemap" 플러그인에서 사이트맵을 만들때 "lastmod" 태그를 수정일자로 표시할 수 있다. 

last_modified_at 변수 추가 하고 수정일자 추가한다. 

```markdown
date: 2020-10-16 18:00:00 +0900
last_modified_at: 2020-10-22 18:00:00 +0900
```

## post.html 제목아래 수정 일자 표시

"layouts/post.html" 파일의 "page-title" 표시부분 아래 "page.date"를 "page.last_modified_at"으로 바꿔준다. 

"last-modified-at" 변수가 없는 페이지를 위해 아래와 같이 "last-modified-at" 변수가 없으면 기존과 같이 "page.date" 변수를 사용하도록 한다. 

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

[jekyll/jekyll-sitemap](https://github.com/jekyll/jekyll-sitemap) 플러그인을 사용하면 내용에 변경이 생기면 sitemap.xml 파일이 만들어진다. 

"jekyll-sitemap" 플러그인의 "lastmod" 생성 정책은 아래와 같다. 

```html
<lastmod> tag
The <lastmod> tag in the sitemap.xml will reflect by priority:

1. The modified date of the file as reported by the filesystem if you have jekyll-last-modified-at plugin installed (not compatible with GitHub Pages auto building)
2. A personalised date if you add the variable last_modified_at: with a date in the Front Matter
3. The creation date of your post (corresponding to the post.date variable)
```

위와 같이 "last-modified-at" 변수를 추가하면 "sitemap.xml"의 "lastmod" 태그가 수정일자로 생성된다. 

```xml
<url>
<loc>http://0.0.0.0:4000/tools/2020/10/16/jekyll_lastmodified.html</loc>
<lastmod>2020-10-23T01:00:00+09:00</lastmod>
</url>
```

## 참고 URL

- [Adding last modified date to Jekyll](https://tomkadwill.com/adding-last-modified-date-to-jekyll) 
- [파일의 마지막 수정일을 sitemap.xml에 사용해보자](https://dev-yakuza.github.io/ko/jekyll/jekyll-last-modified-at/)
