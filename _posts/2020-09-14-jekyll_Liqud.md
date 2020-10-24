---
layout: post
title: "Jekyll 블로그에서 관련글 목록 구성하기"
description: "Jekyll 블로그에서 관련글 목록 구성하기"
img: jekyll-title.png
date: 2020-10-09 20:00:00 +0900
last_modified_at: 2020-10-24 16:00:00 +0900
tags: [wsl, jekyll] # add tag
related: jekyll
categories: tools
---

github page에 블로그를 만들고 Woindows 및 리눅스 관련 튜토리얼 및 문제해결을 위한 삽질과 관련 링크를 정리해 왔다. 

글들을 정리하다보니 연관성이 있는 주제의 페이지 목록을 하단에 표시하고 싶어 간단하게 수정을 해보고자 한다. 
<!--more-->

각 페이지에 관련 글의 링크를 직접 만들 수도 있지만 Jekyll로 만들어진 블로그이다 보니 머릿말에 사용자변수를 추가하는 방법으로 만들어 본다. (Jekyll에서 추천하는 방식이 있을지도 모른다.)


## jekyll 페이지 변수 

jekyll의 디렉토리 구조 및 Liquid 언어를 잘 모르는 상태라 [jekyll 변수](https://jekyllrb-ko.github.io/docs/variables/)의 정보를 확인하고 현재 환경에서 하나하나 수정해가면서 테스트 하기로 한다. 

페이지 변수인 tags를 이용하려고 하니 하나의 글에 여러개의 tag가 있는 경우 글이 중복이 될 수 도 있어  사용자변수를 추가한다. 

> Jekylli의 머릿말은 [YAML](https://yaml.org/) 형식으로 구성되며 추가한 변수는 포스트 내에서 사용할 수 있다. 

페이지 변수에 related 변수를 만들고 tags중 연관글로 표시하고 싶은 태그 하나만 related에 설정한다. 

```md
---
layout: post
title: "Jekyll 블로그에서 관련글 목록 구성하기"
tags: [wsl, jekyll] # add tag
related: jekyll
---
```

전체 사이트에서 tag 변수를 가져와서 페이지의 related 변수의 값과 같은 포스트를 목록으로 표시한다. 

Liquid와 태그기능을 이용한 [포스트 목록 표시하기 Permalink](https://jekyllrb-ko.github.io/docs/posts/#포스트-목록-표시하기)코드를 간단하게 수정하여 적용한다. 

## 코드 샘플 

{% raw %}
```liquid
<ul class="posts-list">
{% for tag in site.tags %}
  {% for post in tag[1] %}
    {% if tag[0] == page.related %}
      <li>
        <a href="{{site.baseurl}}{{post.url}}">{{post.title}}</a>
      </li>
    {% endif %}
  {% endfor %}
{% endfor %}
<ul>
```
{% endraw %} 


## 적용 후 배포 

이 코드는 "layouts/post.html" 파일의 "page-footer" 바로 위에 추가한다. 


시리즈로 작성되는 글인 경우 페이지의 상단에 해당 목차를 넣는 경우도 있는데 이럴경우 정렬기준을 추가해야 한다. 
