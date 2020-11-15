---
layout: post
title: "MS Clariy 사용자 행동분석 도구 적용하기"
description: "Github Pages에 MS Clariy 사용자 행동분석 도구 적용하기"
img: image-title.jpg
date: 2020-11-13 00:00:00 +0900
last_modified_at: 2020-11-15 00:00:00 +0900
tags: [msclarity, jekyll, github pages] # add tag
related: jekyll
categories: tools
---

Github Pages에 [MS Clariy](https://clarity.microsoft.com) 사용자 행동분석 도구를 적용해본다.   

사실 적용이랄것도 없이 "구글 애널리틱스"와 마찬가지로 제공되는 자바스크립트를 추가하면 끝이다. 

<!--more--> 
## Clarity 추적 스크립트 

Clarity 사이트에 로그인 후 "ADD NEW PROJECT"에 사이트를 추가하고 가이드의 자바스크립트 코드를 블로그에 추가한다. 

> 각 사이트마다 10자리의 코드가 발급된다. 

```javascript
<script type="text/javascript">
        (function(c,l,a,r,i,t,y){
                c[a]=c[a]||function(){(c[a].q=c[a].q||[]).push(arguments)};
                t=l.createElement(r);t.async=1;t.src="https://www.clarity.ms/tag/"+i;
                y=l.getElementsByTagName(r)[0];y.parentNode.insertBefore(t,y);
        })(window, document, "clarity", "script", "[[:alnum:]]{10}");
</script> <!-- End MS clarity -->
```

## _layouts/default.html 

Jekyll에서 기존 구글 애널리틱스 스크립트를 추가한 것과 마찬가지로 _includes/ms-clarity.html 파일을 만들고 아래 템플릿에 추가한다. 

{% raw %}
```html
<!DOCTYPE html>
<html lang="en">
{% include head.html %}
<body>
  <div class="wrapper">
    {{ content }}
  </div>
  {% include analytics.html %}
  {% include ms-clarity.html %}
  <script id="dsq-count-scr" src="//awesome-life.disqus.com/count.js" async></script>
</body>
</html>
```
{% endraw %}

적용 후 약 두 시간 정도 후 분석 결과가 집계되어 표시된다. 

## 참고 URL
- [Microsoft Clarity](https://clarity.microsoft.com){:target="_blank"}
