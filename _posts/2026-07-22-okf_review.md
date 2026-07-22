---
layout: post
comments: true
title: "구글 OKF(Open Knowledge Format)는 RAG를 대체하나?"
description: "구글이 발표한 Open Knowledge Format의 구조와 기존 벡터 DB 기반 RAG 대비 장단점 검토"
img: api_code_title.jpg
date: 2026-07-22 00:30:00 +0900
last_modified_at: 2026-07-22 00:30:00 +0900
tags: [rag, okf, llm, knowledge, vector database, google] # add tag
related: llm
categories: dev
---
"구글의 OKF가 벡터 데이터베이스를 대체한다"는 [Medium 글](https://secret-dev.medium.com/beyond-rag-how-googles-open-knowledge-format-okf-is-replacing-the-vector-database-2ffb5bc2f8eb)을 보고, 공식 스펙과 발표 자료를 직접 확인해서 기존 RAG 방식과 비교 검토해봤다. (이 글의 서베이는 Claude Code 에이전트가 수행했다.)

<!--more-->

> **TL;DR:** OKF는 2026년 6월 Google Cloud가 발표한 오픈 스펙으로, "LLM 위키" 패턴을 마크다운 파일 디렉토리로 표준화한 **지식 포맷**이다. 청킹·임베딩 없이 에이전트가 파일을 직접 읽고 링크를 따라가는 결정적(deterministic) 접근이 강점이지만, 공식 스펙 스스로 밝히듯 **벡터 검색의 대체가 아니라 보완**이다. "RAG를 대체한다"는 제목은 과장으로 보는 것이 맞다. 

## OKF가 뭔가?

[OKF v0.1](https://github.com/GoogleCloudPlatform/knowledge-catalog/tree/main/okf)은 에이전트에게 줄 지식을 표현하는 벤더 중립 스펙이다. 클라우드 서비스도, SDK도 아니고 **그냥 파일 규약**이다.

- **마크다운 파일 디렉토리** = 지식 번들. 개념(테이블, 지표, 런북, API 등) 하나당 파일 하나, **파일 경로가 곧 개념의 식별자**
- **YAML frontmatter**: 필수 필드는 `type` 하나뿐. `title`, `description`, `resource`(원본 링크), `tags`, `timestamp`는 선택
- **일반 마크다운 링크**(`[customers](/tables/customers.md)`)로 개념 간 관계를 표현 — 디렉토리 계층 위에 그래프가 얹힌다
- 예약 파일: `index.md`(내비게이션·점진적 공개), `log.md`(변경 이력)

```text
sales/
├── index.md
├── datasets/orders_db.md
├── tables/orders.md        ← frontmatter + 스키마 + 조인 경로
├── tables/customers.md
└── metrics/weekly_active_users.md
```

tarball로 배포하고, git 저장소에 올리고, 파일시스템에 마운트하면 끝이다. 에이전트는 임베딩 없이 glob·grep·파일 읽기·링크 추적만으로 소비한다.

## 기존 벡터 DB 기반 RAG와 뭐가 다른가?


| 비교 항목      | 벡터 DB 기반 RAG      | OKF (LLM 위키)                  |
| :---------- | :----------------- | :----------------------------- |
| 검색 방식      | 임베딩 유사도 (확률적)     | 파일 경로·링크 추적 (결정적)             |
| 전처리        | 청킹 + 임베딩 필수       | 없음 (파일 그대로)                   |
| 표·구조 데이터   | 청킹이 표 구조를 파괴하기 쉬움 | 마크다운 표 원형 유지                  |
| 갱신         | 재임베딩·인덱스 동기화 필요   | 파일 수정 + git commit            |
| 지식 출처 추적   | 청크 메타데이터에 의존      | frontmatter `resource`·링크로 명시 |
| 이식성        | 벡터 DB 벤더에 종속      | 어떤 시스템에서든 읽기 가능               |
| 대규모 검색     | 강함 (수백만 문서)       | 약함 (grep 수준)                  |
| 모호한 자연어 질의 | 강함 (의미 검색)        | 약함 (경로·키워드 기반)                |


## OKF의 실질적 장점

1. **결정성**: "이 테이블의 조인 키가 뭐야?" 같은 질문에 벡터 검색은 관련 청크를 *확률적으로* 가져오지만, OKF는 `tables/orders.md`를 *정확히* 읽는다. 오래된 청크가 섞여 들어올 여지가 없다.
2. **운영 단순성**: 임베딩 파이프라인·벡터 DB 운영이 통째로 사라진다. 지식 갱신 = 파일 수정 + 커밋이라 버전 관리·리뷰·롤백이 코드와 동일한 흐름이다.
3. **사람과 에이전트가 같은 문서를 본다**: 마크다운이라 사람이 직접 읽고 고칠 수 있다. Claude Code의 `CLAUDE.md`나 에이전트 스킬 문서와 정확히 같은 철학이고, 이 패턴이 실제로 잘 동작한다는 것은 [제안서 파이프라인]({{site.baseurl}}/tools/2026/07/21/proposal_automation.html)에서도 체감했다.
4. **표준화의 가치**: 지금까지 각자 만들던 "LLM용 위키"에 공통 규약이 생기면, 서로 다른 팀·도구가 만든 지식 번들을 번역 없이 교환할 수 있다.

## 그런데 "RAG 대체"는 과장이다

공식 스펙과 구글 블로그를 보면 입장이 명확하다 — **보완이지 대체가 아니다**. 이유는 비교표의 아래 두 줄에 있다.

- OKF의 검색은 결국 경로 탐색과 grep이다. 번들이 수천 파일로 커지면 "환불 정책이 어디 적혀 있지?" 같은 모호한 질의에는 의미 검색이 필요해지고, 공식 문서도 큰 번들에서는 **OKF를 벡터 스토어에 넣어 시맨틱 검색을 얹는 구성**을 언급한다.
- OKF가 잘 맞는 지식은 큐레이션된 정형 지식(데이터 카탈로그, 지표 정의, 런북)이다. 수백만 건의 비정형 문서(민원, 상담 이력, 논문)를 다루는 RAG의 영역은 그대로 남는다.

결국 실무 구도는 [위성영상 글]({{site.baseurl}}/dev/2026/07/16/satellite_mllm_vs_embedding.html)에서 정리했던 하이브리드와 같은 모양이 된다. **큐레이션된 핵심 지식은 OKF로 결정적으로 제공하고, 대량 비정형 데이터는 벡터 검색으로 보완**하는 구성이다.

## 참고

- [OKF 스펙 (GitHub: GoogleCloudPlatform/knowledge-catalog)](https://github.com/GoogleCloudPlatform/knowledge-catalog/tree/main/okf){:target="_blank"}
- [Google Cloud 블로그: How the Open Knowledge Format can improve data sharing](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing/){:target="_blank"}
- [MarkTechPost: Google Cloud Introduces OKF](https://www.marktechpost.com/2026/06/16/google-cloud-introduces-open-knowledge-format-okf-a-vendor-neutral-markdown-spec-for-giving-ai-agents-curated-context/){:target="_blank"}
- [발단이 된 Medium 글](https://secret-dev.medium.com/beyond-rag-how-googles-open-knowledge-format-okf-is-replacing-the-vector-database-2ffb5bc2f8eb){:target="_blank"}

