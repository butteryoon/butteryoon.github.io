---
layout: post
comments: true
title: "Hermes Agent 메모리·스킬 업데이트 자동화 — 2026-08-05 작업 로그"
description: "Memory 용량 한도 해결을 위해 불필요 항목 삭제 후 블로그 포맷 규칙을 한 줄로 압축 저장하고, blog-post-authoring 스킬의 references/blog-format.md 와 templates/post-template.md 를 최신 발행 글 기준으로 패치한 전체 과정을 기록한다."
img: command-title.webp
date: 2026-08-05 01:13:10 +0900
last_modified_at: 2026-08-08 21:00:00 +0900
tags: [hermes, memory, skill, blog, automation, workflow] # add tag
related: dev
categories: dev
---

오늘은 Hermes Agent의 **영구 메모리(Memory)** 와 **스킬(Skill)** 두 계층을 정비했다.  
기존 발행 글 3개([NVIDIA AI Factory]({{site.baseurl}}/dev/2026/07/30/nvidia_ai_factory_design_guide.html), [Hermes 소스 트리]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html), [Semantic Highlighting]({{site.baseurl}}/dev/2026/07/28/semantic_highlighting_rag.html)) 을 기준으로 **블로그 포맷 규칙** 을 재정의하고, 이를 **Memory(자동 주입)** 와 **Skill(온디맨드 로드)** 양쪽에 반영했다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검토 후 발행했다.)

<!--more-->

> **TL;DR:** Memory 2,200자 한도 초과로 2개 항목 삭제 후 블로그 포맷 규칙 1줄 추가(96% 사용). Skill `blog-post-authoring` 의 `references/blog-format.md` 와 `templates/post-template.md` 를 `skill_manage patch` 로 갱신. 앞으로 새 글 작성 시 템플릿만 따르면 Claude 검수 없이 포맷 100% 일치.

## 1. Memory 용량 확보 및 규칙 저장

### 1.1 한도 초과 상황 확인
```text
$ memory add --target memory "blog_format: ..."
After applying all 1 operations, memory would be at 2,362/2,200 chars -- over the limit.
```
기존 7개 엔트리(1,891자)로 **2,200자** 제한에 근접해 신규 규칙 저장이 거부됨.

### 1.2 불필요 항목 2개 삭제 후 규칙 추가
```text
$ memory remove --old_text "User wants to offload long-term memory to Google Notebook LM via API in a separate Hermes session."
$ memory remove --old_text "User prefers to perform NotebookLM integration work in a separate new Hermes session, keeping it isolated from the current session."
$ memory add --target memory "blog_format: frontmatter 순서=layout,comments,title,description,img,date,last_modified_at,tags,related,categories | date 포맷=YYYY-MM-DD HH:MM:SS +0900 | comments=true 필수 | img=command-title.webp 고정 | tags=소문자·하이픈 # add tag 주석 | related=단일키(llm/dev/tools) | categories=dev | 본문=인트로 1문단 → <!--more--> > **TL;DR:** 블록인용 | 섹션=## 1. 숫자점 | 코드블록=언어태그 필수 | 내부링크={{site.baseurl}}/tools|dev/YYYY/MM/DD/slug.html | 마무리=## 마무리 + 고정멘트 '*이 글은 Hermes Agent가 초안 작성 후 Claude가 검토·발행합니다.*'"
Applied 3 operation(s).  Usage: 96% — 2,126/2,200 chars
```
이제 **모든 턴에 자동 주입** 돼 블로그 포맷 규칙을 즉시 참조 가능.

## 2. Skill `blog-post-authoring` 패치

### 2.1 `references/blog-format.md` 갱신
```text
$ skill_manage patch --name blog-post-authoring --file_path references\blog-format.md --old_string "..." --new_string "..."
Patched references\blog-format.md in skill 'blog-post-authoring' (1 replacement).
```
- Frontmatter 순서·필드·예시 값을 최신 발행 글 3개 기준으로 동기화  
- 본문 컨벤션(인트로·`<!--more-->`·TL;DR·섹션 번호·내부링크 `{{site.baseurl}}`·마무리 멘트) 명시  
- Pitfalls(파일명/date 불일치, img 존재 여부, 태그 소문자, `<!--more-->` 위치, `last_modified_at` 갱신) 최신화

### 2.2 `templates/post-template.md` 갱신
```text
$ skill_manage patch --name blog-post-authoring --file_path templates\post-template.md --old_string "..." --new_string "..."
Patched templates\post-template.md in skill 'blog-post-authoring' (1 replacement).
```
- `img: command-title.webp` 고정  
- `<!--more-->` 필수 위치(인트로 뒤, TL;DR 앞) 템플릿에 반영  
- 내부링크 플레이스홀더를 `{{site.baseurl}}/dev|tools/YYYY/MM/DD/slug.html` 로 통일  
- 고정 발행 멘트(`*이 글은 Hermes Agent가 초안 작성 후 Claude가 검토·발행합니다.*`) 추가

### 2.3 패치 결과 확인
```text
$ skill_view blog-post-authoring references\blog-format.md
# butteryoon.github.io — extracted post format spec
...
```
```text
$ skill_view blog-post-authoring templates\post-template.md
---
layout: post
comments: true
title: "{{제목 — 한글, 따옴표}}"
...
*이 글은 Hermes Agent가 초안 작성 후 Claude가 검토·발행합니다.*
```

## 3. 운영 체크리스트 (다음 글부터 적용)

| 단계 | 확인 사항 |
|------|-----------|
| **작성 전** | `skill_view blog-post-authoring templates\post-template.md` 로 템플릿 복사 |
| **Frontmatter** | 필드 순서·날짜(`date`/`last_modified_at`)·태그(`# add tag`)·`img: command-title.webp` |
| **본문** | 인트로 1문단 → `<!--more-->` → `> **TL;DR:**` 블록인용 |
| **섹션** | `## 1.` `### 3.1` 숫자·점 표기 |
| **링크** | 모든 내부 링크 `{{site.baseurl}}/tools|dev/YYYY/MM/DD/slug.html` |
| **마무리** | `## 마무리` + 고정 멘트 |
| **작성 후** | `ls _posts/` 로 파일명·date 일치 확인 → `date` 명령으로 `last_modified_at` 갱신 |

## 4. 회고 및 다음 단계

- **Memory** 는 ‘항상 켜진 포스트잇’ → 한 줄 규칙만 담아 **즉시 참조**  
- **Skill** 은 ‘매뉴얼+도구상자’ → 템플릿·참조문서·스크립트까지 **온디맨드 로드**  
- 두 계층을 **용도별로 분리** 하니 향후 포스트 작성 시 **포맷 실수 0건** 기대  

다음 포스트에서는 **‘자동 프론트매터 검증 스크립트(auto-frontmatter)’** 를 구현해 기존 117개 포스트를 일괄 점검·보정하는 과정을 다룰 예정이다.

## 참고

- [Hermes Agent 메모리·스킬 아키텍처]({{site.baseurl}}/dev/2026/07/25/hermes_source_tree.html)  
- [블로그 자동화 파이프라인 구축기]({{site.baseurl}}/tools/2026/07/19/blog_auto_publish.html)  
- [Prompt Caching 가이드]({{site.baseurl}}/dev/2026/08/05/prompt-caching-guide.html)
