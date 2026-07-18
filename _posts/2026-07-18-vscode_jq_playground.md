---
layout: post
comments: true
title: "VS Code에서 jq 필터 실험하기 - jq Playground"
description: "jq 필터를 노트북처럼 실행하며 실험할 수 있는 VS Code 확장 jq Playground 소개"
img: vscode-title.jpg
date: 2026-07-18 18:00:00 +0900
last_modified_at: 2026-07-18 18:00:00 +0900
tags: [jq, vscode, Visual Studio Code, json, jq playground] # add tag
related: jq
categories: tools
---

터미널에서 jq 필터를 다듬다 보면 따옴표 이스케이프와 씨름하며 명령을 반복 실행하게 된다. [예전 글]({{site.baseurl}}/tools/2023/09/24/jq.html)에서 CLI로 텍스트를 JSON으로 변환하는 방법을 다뤘는데, 이번에는 VS Code 안에서 jq 필터를 노트북처럼 실험할 수 있는 **jq Playground** 확장을 소개한다.

<!--more-->

> **TL;DR:** VS Code 마켓플레이스의 **jq Playground — JSON Filter Notebook**(davidnussio) 확장을 설치하면 `.jqpg` 파일에서 jq 필터를 작성하고 `Ctrl+Enter`로 즉시 실행하며 결과를 확인할 수 있다. 파일·URL·셸 명령 출력까지 입력으로 받고, 자동완성과 AI 필터 생성도 지원한다. 참고로 이름이 비슷한 "vscode-jq"(andricDu) 확장은 2020년 이후 방치 상태라 권하지 않는다.

## 어떤 확장을 설치해야 하나?

마켓플레이스에서 "jq"로 검색하면 비슷한 확장이 여럿 나온다.

| 확장 | 게시자 | 상태 |
| :--- | :--- | :--- |
| **jq Playground — JSON Filter Notebook** | davidnussio | 활발히 유지보수 중, 기능 가장 풍부 (권장) |
| vscode-jq | andricDu | 설치 5.7만이지만 2020년 1월 이후 업데이트 없음 |
| jq-vscode | petli-full | 소규모 인라인 처리기 |
| jq Playground | maargenton | 라이브 프리뷰 위주 소규모 확장 |

이름만 보면 "vscode-jq"가 공식처럼 보이지만 6년 넘게 방치된 확장이므로, **davidnussio의 jq Playground**를 설치하면 된다.

## 어떻게 사용하나?

1. 확장 설치 후 명령 팔레트(`Ctrl+Shift+P`)에서 **JQPG: Open Playground Panel** 실행
2. jq 바이너리는 시스템에 설치된 것을 자동 감지하고, 없으면 **JQPG: Download jq binary** 명령으로 받아준다 (zero config)
3. 필터를 입력하고 `Ctrl+Enter`로 실행 — 결과가 출력 패널에 바로 표시된다 (`Shift+Enter`는 사이드 편집기로 출력)

`.jqpg` 확장자 파일로 저장하면 여러 필터를 각자의 데이터 소스와 함께 한 파일에 노트북처럼 모아둘 수 있다.

```text
# 로컬 파일을 입력으로
$ jq '.items[] | {id, name}'
file://./data.json

# URL 응답을 입력으로
$ jq '.[0:3]'
https://api.github.com/users/butteryoon/repos

# 셸 명령 출력을 입력으로
$ jq '.ip'
$ curl -s http://api.ipify.org?format=json
```

## 유용한 기능

- **다양한 입력 소스**: 인라인 JSON, 로컬 파일, 열려 있는 편집기 버퍼, URL, 셸 명령 출력(JSON/JSONL)까지 입력으로 사용할 수 있다
- **자동완성**: jq 내장 함수를 공식 문서·예제와 함께 제안해줘서 매뉴얼을 오가는 횟수가 줄어든다
- **jq 문법 하이라이팅**: 필터와 내장 JSON 모두 하이라이팅된다
- **AI 필터 생성**: 원하는 변환을 자연어로 설명하면 jq 필터를 만들어 플레이그라운드에 넣어준다 (GitHub Copilot 필요, 선택 사항)
- **변수·멀티라인 필터** 지원으로 복잡한 필터도 정리해서 작성할 수 있다

## 마무리

jq 필터는 완성되면 한 줄이지만 완성하기까지의 시행착오가 잦은 도구다. 터미널에서 히스토리를 오르내리는 대신, `.jqpg` 노트북에서 데이터 소스를 고정해두고 필터만 고쳐가며 실행하는 흐름이 확실히 편하다. 완성된 필터를 스크립트나 파이프라인에 옮겨 쓰면 된다.

## 참고

- [jq Playground — Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=davidnussio.vscode-jq-playground){:target="_blank"}
- [GitHub: davidnussio/vscode-jq-playground](https://github.com/davidnussio/vscode-jq-playground){:target="_blank"}
- [jq 매뉴얼](https://jqlang.github.io/jq/manual/){:target="_blank"}
- [TEXT to JSON 변환 (기존 글)]({{site.baseurl}}/tools/2023/09/24/jq.html){:target="_blank"}
