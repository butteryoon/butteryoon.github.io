---
layout: post
title: "노션(Notion) 우선순위 테이블 만들기."
description: "긴급, 중요 두가지 항목의 선택에 따라 항목의 우선 순위를 간단한 수식으로 체크하여 별표 개수로 표시한다."
img: title-idea.webp
date: 2021-02-07 18:00:00 +0900
last_modified_at: 202l-02-07 18:00:00 +0900
tags: [idea, notion, task] # add tag
related: idea
categories: tools
---


노션(Notion)에서 긴급, 중요 두가지 항목의 선택에 따라 항목의 우선 순위를 간단한 수식으로 체크하여 별표 개수로 표시한다.
<!--more-->

물론 테이블의 속성유형을 선택 이나 다중선택으로 하고 이모지를 이용해서 태그를 만들어 놓아도 되지만 간단한 수식으로 필들를 만들어 놓으면 다른 필드 선택에 의해 자동으로 원하는 우선순위를 지정할 수 있다. 

![notion task]({{site.baseurl}}/assets/img/notion_priority_task.png) 

## 긴급, 중요 체크박스 필드 

우선순위는 3단계로 정의하고 "if" 수식으로 해당 속성의 체크박스 상태를 확인하고  체크된 상태면 TRUE를 리턴한다. 

> if ( 조건식 , "True이면 실행", "False 이면 실행" )  
> 긴급 이고 중요하면 별3개  
> 긴급 이고 중요하지 않으면 별2개   
> 긴급이 아니고 중요하면 별1개  
> 둘다 아니면 별 없음.   

```scheme
if(prop("긴급"), 
	if(prop("중요"), "⭐⭐⭐", "⭐⭐"), 
		if(prop("중요"), "⭐", ""))
```

우선순위 필드를 생성하는 과정을 정리.

<iframe width="560" height="315" src="https://www.youtube.com/embed/kQ1AZA61hfA" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


## 참고 URL
- [Notion Formulas: Prioritize Tasks Automatically](https://www.notion.vip/notion-formulas-prioritize-tasks-automatically/){:target="_blank"}
