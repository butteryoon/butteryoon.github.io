---
layout: post
comments: true
title: "Claude Code로 블로그 정리하기"
description: "Dependabot 취약점 해소 및 저장소 정리 작업 기록"
img: jekyll-title.png
date: 2026-07-14 21:00:00 +0900
last_modified_at: 2026-07-15 16:10:00 +0900
tags: [jekyll, github pages, dependabot, claude code, bundler] # add tag
related: jekyll
categories: tools
---

오랜만에 블로그 저장소를 열어보니 GitHub Dependabot이 취약점 36건(critical 1건 포함)을 알려주고 있었다. Claude Code의 도움을 받아 저장소 정리와 취약점 해소 작업을 진행한 내용을 기록한다.

<!--more-->

## 작업 내용 요약

1. `CLAUDE.md` 생성 및 `README.md`를 블로그 정보에 맞게 수정
2. 미사용 gulp 관련 파일 삭제 (npm 취약점 제거)
3. Ruby gem 의존성 업데이트 (Dependabot 취약점 36건 → 0건)
4. 미래 날짜 글이 사이트에서 안 보이는 문제 확인
5. 기존 글(AI TOOLS)을 현재 시점 내용으로 갱신, 새 글(PaperBanana) 작성
6. `_drafts`에 방치되어 있던 임시 글 11건을 보완해서 발행하고 드래프트 정리
7. 발행 글 전체 검토 후 낡은 글 36건 우선순위별 업데이트 (7/15)
8. 시간대 미설정으로 URL 날짜가 하루 밀리는 문제 수정 (7/15)

## CLAUDE.md와 README 정리

Claude Code의 `/init` 명령으로 저장소 구조를 분석해 `CLAUDE.md`를 생성했다. 빌드/실행 명령, 글 작성 규칙(front matter 템플릿, `<!--more-->` 구분자 등)이 정리되어 있어 이후 세션에서 바로 활용할 수 있다.

`README.md`는 테마([Flexible-Jekyll](https://github.com/artemsheludko/flexible-jekyll)) 원본 내용 그대로였는데, 블로그 소개와 로컬 실행 방법, 글 작성 절차로 교체했다.

## gulp 관련 파일 정리

이 블로그는 Jekyll만으로 빌드되는데, 테마에 포함되어 있던 gulp 3 기반의 `gulpfile.js`와 `package.json`이 남아 있었다. 오래된 npm 의존성이 Dependabot 경고의 원인 중 하나였으므로 두 파일을 삭제하고 `_config.yml`의 exclude 목록도 정리했다.

```bash
git rm gulpfile.js package.json
```

## Ruby gem 취약점 해소

남은 경고는 전부 `Gemfile.lock`의 Ruby gem이었다. `gh` CLI로 확인해보면 다음과 같다.

```bash
gh api repos/OWNER/REPO/dependabot/alerts --paginate \
  -q '.[] | select(.state=="open") | [.security_advisory.severity, .dependency.package.name] | join(" | ")'
```

nokogiri, rexml, faraday, activesupport, webrick, addressable, google-protobuf, concurrent-ruby 등이 취약 버전으로 잠겨 있었다.

`bundle update` 실행 시 Gemfile에 글로벌 source가 없어 아래와 같은 오류가 발생했다.

```
Could not find gem 'jekyll-sitemap' in locally installed gems.
```

Gemfile 상단에 source와 jekyll gem을 명시하고 다시 실행하니 정상적으로 갱신되었다.

```ruby
source "https://rubygems.org"

gem "jekyll"
```

```bash
bundle update
bundle exec jekyll build   # 빌드 정상 확인 후 push
```

push 후 Dependabot 알림을 다시 확인하니 열린 알림이 0건으로 정리되었다.

## 미래 날짜 글은 사이트에 안 보인다

이 글을 처음 올렸을 때 사이트에 보이지 않아 당황했는데, 원인은 front matter의 `date`를 현재보다 늦은 시각으로 적었기 때문이었다. GitHub Pages는 기본 설정(`future: false`)에서 미래 날짜 글을 빌드에서 조용히 제외한다. 에러도 없이 그냥 안 보이므로, 새 글의 `date`는 반드시 현재 시각 이전으로 적어야 한다. 이 내용은 `CLAUDE.md`에도 주의사항으로 추가해뒀다.

## 글 정리

- 기존 [AI TOOLS]({{site.baseurl}}/tools/2025/04/13/aitools.html) 글을 현재 시점에 맞게 갱신했다. 글에서 소개했던 AI Toolkit이 **Foundry Toolkit for VS Code**로 이름이 바뀌고 Bulk Run이 Evaluation으로 통합된 내용을 반영했다.
- 문장으로 아키텍처 다이어그램을 만들어주는 서비스를 소개하는 [PaperBanana]({{site.baseurl}}/tools/2026/07/14/paperbanana.html) 글을 새로 작성했다.

## 드래프트 정리

`_drafts` 디렉토리에 2019~2023년 사이에 쓰다 만 임시 글 19건이 방치되어 있었다. 내용이 있는 11건(Slack webhook, ffmpeg SRT, 터널링 도구, 네이버 서치어드바이저 등)은 2026년 기준으로 사실관계를 다시 확인하고 보완해서 발행했고, 제목만 있고 내용이 없던 8건은 템플릿만 남기고 삭제했다. 버전이 오래됐거나(OpenSIPS 2.4 → 4.0), 이름이 바뀌었거나(MATIC → POL), 서비스 상태가 달라진 것들이 많아 오래된 드래프트는 발행 전에 반드시 현재 상태를 다시 확인해야 한다는 것을 실감했다.

## 발행 글 전수 검토 및 업데이트 (7/15 추가)

드래프트 정리에 이어 발행된 글 80여 건 전체를 검토해서 업데이트가 필요한 글 36건을 우선순위별로 나눠 수정했다.

- **높음 11건**: 폐기된 도구/절차 — `letsencrypt-auto` → snap 기반 certbot 재작성(3건), WSL 최신 설치법(`wsl --install`), SMB1 보안 경고, 종료된 서비스(TeamSQL, Pocket 등) 안내, Owncast/Janus 현재 버전 반영
- **중간 14건**: EOL 환경·바뀐 UI — winget 시대 반영, Tailscale 무료 플랜(현재 Personal, 6명/디바이스 무제한), Oracle 프리티어 축소(2026-06부터 A1 2 OCPU/12GB), UA → GA4, Python 3.12의 `ssl.wrap_socket` 제거 대응 등
- **낮음 11건**: 오타·죽은 링크 수정, 유지보수 중단 도구(Vundle 등) 안내. 콘텐츠 가치가 없는 Jekyll 기본 샘플 글은 삭제

모든 글은 원본 날짜와 URL을 유지한 채 본문 상단에 `[업데이트]` 안내 블록을 넣고 `last_modified_at`만 갱신하는 방식으로 처리했다. letsencrypt-auto 폐기(3건), Vundle 중단(2건)처럼 같은 원인이 여러 글에 걸쳐 있는 경우가 많았다.

## 시간대 미설정으로 URL 날짜가 하루 밀리는 문제 (7/15 추가)

업데이트 반영을 확인하다가 또 하나의 함정을 발견했다. `_config.yml`에 `timezone` 설정이 없으면 GitHub Pages는 **UTC로 빌드**하기 때문에, 게시 시각이 오전 9시(KST) 이전인 글은 사이트 URL의 날짜가 하루 앞으로 밀린다. 예를 들어 `date: 2023-11-14 01:00 +0900`인 글의 실제 URL은 `/2023/11/13/...`이 된다. 로컬 빌드(KST)와 결과가 달라 내부 링크가 조용히 깨지는 원인이 됐다.

```yaml
# _config.yml
timezone: Asia/Seoul
```

Windows 로컬 빌드에서는 timezone 설정 시 `tzinfo-data` gem이 없으면 빌드가 실패하므로 Gemfile에 함께 추가해야 한다.

```ruby
gem 'tzinfo-data', platforms: [:mingw, :x64_mingw, :mswin]
```

## 마무리

테마를 fork해서 쓰는 블로그는 사용하지 않는 빌드 도구 흔적이 남아 취약점 경고의 원인이 되기 쉽다. 실제로 쓰는 의존성만 남기고, `Gemfile.lock`은 주기적으로 `bundle update`로 갱신해주는 것이 좋겠다. 드래프트도 쌓아두면 결국 내용이 낡아버리니 그때그때 발행하거나 정리하는 편이 낫다. 그리고 미래 날짜 글 미노출, 시간대에 따른 URL 밀림처럼 GitHub Pages의 조용한 함정들은 한 번 겪을 때마다 `CLAUDE.md`에 기록해두면 같은 실수를 반복하지 않는다.
