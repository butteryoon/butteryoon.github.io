---
layout: post
comments: true
title: "Hermes 초안 자동 검수 파이프라인 구축 — 일간 스케줄러와 한글 윤문 스킬"
description: "Hermes 에이전트가 매일 쓰는 블로그 초안을 자동으로 검수·발행하는 로컬 스케줄러 구축기. 중복·환각 안전장치가 실전에서 잡아낸 것들, humanize-korean 윤문 스킬 통합, 헤드리스 운영에서 만난 함정까지."
img: jekyll-title.png
date: 2026-08-24 01:40:00 +0900
last_modified_at: 2026-08-24 01:40:00 +0900
tags: [claude code, automation, hermes, task scheduler, humanize, ai agent] # add tag
related: jekyll
categories: tools
---

[주간 발행 자동화]({{site.baseurl}}/tools/2026/07/19/blog_auto_publish.html)에 이어 자동화 두 번째 단계다. Hermes 에이전트가 매일 저녁 서베이 초안을 `_drafts/`에 쓰기 시작하면서 사람이 매번 "검토하고 발행해줘"라고 시키는 단계를 없앴다 — 매일 19시에 Claude Code가 헤드리스로 초안을 검수해서 발행하는 파이프라인과, 발행 직전 AI 티를 지우는 한글 윤문 스킬을 붙인 기록이다.

<!--more-->

> **TL;DR:** ① Windows 작업 스케줄러 + `claude -p` 헤드리스로 매일 19시 `_drafts/` 검수·발행 파이프라인을 구축했다. 초안이 없으면 claude 호출 없이 스킵한다. ② 프롬프트에 넣은 안전장치(중복 판정, 원문 대조, 발행 보류)가 첫 실행에서 바로 환각 초안을 걸러냈다. ③ 발행 직전 [humanize-korean](https://github.com/epoko77-ai/im-not-ai){:target="_blank"} 스킬로 번역투·기계적 병렬 같은 AI 티를 윤문한다 — 의미·수치는 건드리지 않는 보수 모드로.

## 왜 일간 파이프라인인가

그동안의 흐름은 이랬다: Hermes가 초안을 쓰면 → 사용자가 알려주고 → Claude Code 세션에서 검수·발행. 사람이 트리거인 구조라 초안이 쌓이면 병목이 사람이 된다. 주간 발행(`_reports/` → 일요일 18시)은 이미 스케줄러로 돌고 있었으니, 같은 패턴을 일간 초안에도 적용했다.

클라우드 루틴 대신 로컬 스케줄러를 쓴 이유도 지난번과 같다 — 조직 계정이라 클라우드 에이전트의 GitHub 연결이 막혀 있고, 결정적으로 **Hermes 초안은 커밋 전 로컬 파일이라 클라우드에서는 보이지도 않는다**.

## 파이프라인 구조

```text
Hermes (매일 ~18:00)          Claude Code (매일 19:00)
_drafts/에 초안 작성    →     초안 유무 확인 (없으면 스킵)
                              → 중복 확인 (_posts에 같은 주제 있으면 보류)
                              → 검수 체크리스트 (원문 대조·링크·front matter)
                              → humanize-korean 윤문
                              → _posts로 이동, 빌드 확인, commit & push
```

스케줄러 스크립트에서 눈여겨볼 부분은 두 가지다.

```powershell
# 초안이 없으면 claude 호출 없이 종료 (토큰 절약)
$drafts = Get-ChildItem "$repo\_drafts" -Filter "*.md" |
  Where-Object { $_.Name -ne "0000-00-00-templete.md" }
if (-not $drafts) { "no drafts - skip" >> $log; exit 0 }

& claude.exe -p $prompt `
  --allowedTools "Skill,Read,Glob,Grep,Write,Edit,WebFetch,WebSearch,Bash(git add:*),Bash(git commit:*),Bash(git push:*),Bash(git mv:*),Bash(git rm:*),Bash(bundle exec jekyll build),Bash(curl:*)"
```

- **게이트를 셸에서 먼저**: 초안 유무는 LLM 없이도 알 수 있으니 스케줄러 스크립트가 먼저 거른다. 매일 도는 작업에서 빈 호출을 없애는 것만으로 비용이 크게 준다.
- **도구 화이트리스트**: 헤드리스에서는 권한 프롬프트에 답할 사람이 없다. 필요한 git 명령과 검증 도구만 열고 임의 셸 실행은 막았다. 윤문 스킬을 호출하려면 `Skill`도 목록에 있어야 한다는 걸 뒤늦게 발견했다.

## 프롬프트의 안전장치가 실전에서 잡은 것

프롬프트에는 그간 수동 검수에서 쌓인 체크리스트가 다 들어 있다: 미래 날짜 금지(GitHub Pages가 조용히 제외), 내부 링크의 실제 발행 URL 확인, 외부 링크 200 확인, 원문 요약형 글이면 원문 전문 대조, 비공개 참조 제거, 그리고 **오염이 심하면 발행하지 말고 보류하라**는 출구.

첫 실행이 곧바로 시험대였다. `_drafts/`에 초안 3건이 있었는데 그중 2건이 같은 NVIDIA 원문을 다룬 중복이었고, 결과는:

- 2건 발행 — 원문과 수치 전수 대조 통과, 404 저장소 링크 정정, 원문에 없는 추정 수치 제거까지 하고 커밋·푸시
- 1건 보류 — 같은 원문의 나중 초안인데 저자를 날조하고 "원문에 수치 없음"이라 주장하는 환각본. 프롬프트의 보류 규칙대로 발행하지 않고 `_drafts/`에 남겼다

흥미로운 교훈: **같은 원문의 초안이 여럿이면 나중 것이 더 낫다고 가정하면 안 된다.** 이 사례에서는 먼저 쓴 초안이 정확했고 나중 것이 환각본이었다.

## humanize-korean — 발행 직전 AI 티 지우기

검수를 통과한 글도 AI가 쓴 글 특유의 티가 남는다. "결론적으로", "~를 통해 ~할 수 있다", "첫째·둘째·셋째", 피동태 남용 같은 것들이다. [im-not-ai](https://github.com/epoko77-ai/im-not-ai){:target="_blank"}의 humanize-korean 스킬은 이런 패턴 70여 종을 스팬 단위로 탐지해서 윤문한다.

```text
/plugin marketplace add epoko77-ai/im-not-ai
/plugin install humanize-korean@im-not-ai
```

마음에 든 건 4대 철칙이다 — 의미 불변, 근거 기반(탐지된 구간만 수정), 장르 유지, 과윤문 금지(변경률 30% 초과 시 경고). 헤드리스로 시험해 보니:

- AI 티 샘플 문단: "결론적으로/~를 통해/~에 있어서/첫째·둘째·셋째"가 전부 자연스러워지고 내용 앵커는 보존 (변경률 24.9%)
- 실제 주간 글: 변경률 4.5%의 보수적 개입 — 진행형·명사화·과도한 대시만 손보고 arXiv 링크 28개, 논문 제목, 수치는 무변경 (자체검증 6/6)

잘 쓴 글일수록 덜 고치는 라우팅(light 1콜 / standard 2콜 / heavy 3+콜)이라 비용도 글 상태에 비례한다.

### 운영에서 만난 함정 두 가지

1. **윤문 리포트 주석**: 스킬이 문서 끝에 `<!-- HUMANIZE-SUMMARY ... -->` 블록을 붙인다. 렌더링엔 안 보여도 HTML 소스에 그대로 실리므로 발행 전에 지워야 한다.
2. **last_modified_at 미래 갱신**: 스킬이 수정 시각을 갱신해 주는 건 고마운데, 미래 시각으로 적히면 GitHub Pages가 글을 빌드에서 제외한다. 이 블로그가 세 번쯤 밟은 함정이라, 파이프라인 프롬프트에 "윤문 후 주석 삭제 + 시각 검증" 후처리 규칙을 명시했다.

## 자잘한 견고화

- **더러운 작업 트리에서도 동기화**: Hermes가 미커밋 파일을 남기면 `git pull`이 실패했다. 스케줄러의 동기화를 `git fetch` + `git merge --ff-only`로 바꿔 미커밋 변경과 무관하게 동작하게 했다.
- **작업 폴더 gitignore**: Hermes와 윤문 스킬이 만드는 `_workspace/`를 `.gitignore`에 추가 — 미추적 파일이 파이프라인을 방해하지 않도록.

## 마무리

이제 이 블로그의 글은 세 층을 거친다. **Hermes가 쓰고, Claude가 검증하고, 윤문 스킬이 다듬는다.** 사람은 트리거가 아니라 예외 처리자다 — 파이프라인이 보류한 환각본을 확인하고, 스케줄이 어긋났을 때만 개입한다. 자동화에서 정말 중요한 건 "잘 발행하는 규칙"보다 "발행하면 안 될 때 멈추는 규칙"이라는 걸, 첫 실행에서 환각본을 걸러낸 보류 규칙이 보여줬다.

## 참고

- [im-not-ai (humanize-korean) 저장소](https://github.com/epoko77-ai/im-not-ai){:target="_blank"}
- [Claude Code 헤드리스 모드 (`claude -p`)](https://code.claude.com/docs/en/sdk/sdk-headless){:target="_blank"}
- [Claude Code로 블로그 주간 발행 자동화하기 (관련글)]({{site.baseurl}}/tools/2026/07/19/blog_auto_publish.html)
- [블로그 정리하기 (관련글)]({{site.baseurl}}/tools/2026/07/14/blog_maintenance.html)
