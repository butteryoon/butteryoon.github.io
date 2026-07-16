---
layout: post
comments: true
title: "블로그를 AI 검색 시대에 맞게 손보기"
description: "mermaid 렌더링, 썸네일 정비, llms.txt와 JSON-LD 적용까지 — 블로그 개선 작업 기록"
img: jekyll-title.png
date: 2026-07-16 23:20:00 +0900
last_modified_at: 2026-07-16 23:20:00 +0900
tags: [jekyll, seo, llms.txt, json-ld, mermaid, claude code] # add tag
related: jekyll
categories: tools
---

지난 이틀간의 대청소([블로그 정리]({{site.baseurl}}/tools/2026/07/14/blog_maintenance.html))에 이어, 오늘은 블로그를 Google과 AI 검색 엔진이 잘 읽어갈 수 있는 구조로 손봤다. 작업은 Claude Code와 Gemini 에이전트를 함께 활용했다.

<!--more-->

> **TL;DR:** mermaid 다이어그램 렌더러를 테마에 추가하고, 발행 글 19건의 썸네일을 주제에 맞는 이미지로 교체했으며, `llms.txt`·JSON-LD 구조화 데이터·robots.txt AI 크롤러 허용까지 적용해 LLM 검색 친화적인 블로그로 만들었다.

## mermaid 다이어그램 렌더링 지원

멀티 에이전트 구조를 다룬 [Caller/Executor 패턴 글]({{site.baseurl}}/dev/2026/07/16/caller_executor_agent.html)을 쓰면서 mermaid 플로우차트를 넣었는데, 이 테마에는 mermaid 렌더러가 없어 코드 블록으로만 표시됐다.

확인해보니 테마의 `_includes/javascripts.html`이 어느 레이아웃에도 include되어 있지 않았고, 참조하던 `assets/js/main.js`도 존재하지 않는 테마 잔재였다. 이 파일을 mermaid 로더로 교체하고 default 레이아웃에 include했다. mermaid 블록이 있는 페이지에서만 CDN에서 라이브러리를 로드하므로 다른 페이지에는 부담이 없다.

```html
<script type="module">
  const blocks = document.querySelectorAll('pre code.language-mermaid, code.language-mermaid');
  if (blocks.length > 0) {
    const mermaid = (await import('https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs')).default;
    blocks.forEach(code => {
      const pre = code.closest('pre') || code;
      const div = document.createElement('div');
      div.className = 'mermaid';
      div.textContent = code.textContent;
      pre.replaceWith(div);
    });
    mermaid.initialize({ startOnLoad: false, theme: 'default' });
    await mermaid.run();
  }
</script>
```

## 썸네일 정비

발행 글 전체의 썸네일을 인벤토리해보니 범용 플레이스홀더(`image-title.jpg`)를 쓰거나 아예 이미지가 없는 글이 19건이었다. 3건은 기존 주제별 이미지를 재사용하고, 나머지는 [Unsplash](https://unsplash.com)에서 주제에 맞는 무료 이미지(키보드, 분석 대시보드, 암호화폐, 서버랙 등) 12종을 받아 교체했다. Unsplash 라이선스는 출처 표기 없이 상업적 사용이 가능하다.

## llms.txt와 JSON-LD — AI 검색 최적화

AI 검색 엔진(ChatGPT Search, Perplexity, Gemini 등)이 사이트를 잘 인용하도록 세 가지를 적용했다.

### llms.txt / llms-full.txt

`llms.txt`는 LLM이 HTML을 긁지 않고도 사이트 구조를 파악할 수 있게 도메인 루트에 두는 커뮤니티 표준 파일이다. Jekyll Liquid 템플릿으로 만들어 빌드할 때마다 자동 갱신되게 했다.

- [`/llms.txt`](https://butteryoon.github.io/llms.txt): 사이트 정보 + 최근 40개 글 목록(제목·URL·설명)
- [`/llms-full.txt`](https://butteryoon.github.io/llms-full.txt): 최근 15개 글의 전문 — LLM 인덱서가 요청 한 번으로 최신 글을 통째로 가져갈 수 있다

```liquid
---
layout: null
permalink: /llms.txt
---
# {% raw %}{{ site.title }}{% endraw %}
## Recent Articles
{% raw %}{% for post in site.posts limit:40 %}
- [{{ post.title }}]({{ site.url }}{{ post.url }}): {{ post.description }}
{% endfor %}{% endraw %}
```

### JSON-LD 구조화 데이터

`head.html`에 글 페이지는 `BlogPosting`, 그 외는 `WebSite` 스키마의 JSON-LD를 주입했다. `dateModified`가 front matter의 `last_modified_at`과 연동되므로, 글을 수정하고 이 값을 갱신하면 검색엔진에 "살아있는 콘텐츠"라는 신호를 준다.

### robots.txt에 AI 크롤러 명시 허용

`GPTBot`, `ChatGPT-User`, `ClaudeBot`, `Claude-Web`, `Google-Extended`, `PerplexityBot`을 명시적으로 허용했다.

## 검증

배포 후 실제 서빙 상태를 확인했다.

```
❯ curl -s -o /dev/null -w "%{http_code}" https://butteryoon.github.io/llms.txt
200
```

- `/llms.txt`(약 9KB), `/llms-full.txt`(약 84KB) 정상 서빙
- 글 페이지의 JSON-LD를 추출해 파싱 — 유효한 JSON, `dateModified`·`image` 매핑 정확
- robots.txt에 AI 크롤러 규칙 반영 확인

## 앞으로의 글쓰기 가이드

AI 검색에 인용되기 좋은 글을 위한 권장사항도 정리해뒀다. 이 글부터 적용해본다.

1. **Answer-First**: 글 시작에 TL;DR 블록으로 핵심 답을 먼저 제시
2. **질문형 헤더**: "SCP 전송 명령" 대신 "SCP로 파일을 어떻게 전송하나요?"처럼 사용자가 물어볼 법한 형태로
3. **정보 밀도 유지**: 실제 명령어, 설정 파일, 에러 출력 같은 구체적 데이터가 LLM 인용에 유리
4. **`last_modified_at` 갱신**: 글을 수정할 때마다 반드시 갱신

## 참고

- [llms.txt 표준](https://llmstxt.org/){:target="_blank"}
- [Google 구조화 데이터: Article](https://developers.google.com/search/docs/appearance/structured-data/article){:target="_blank"}
- [mermaid](https://mermaid.js.org/){:target="_blank"}
