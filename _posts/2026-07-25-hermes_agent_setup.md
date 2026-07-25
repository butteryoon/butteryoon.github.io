---
layout: post
comments: true
title: "Hermes Agent 설치부터 설정까지 - Windows에서 시작하기"
description: "Nous Research의 Hermes Agent를 Windows 11에 설치하고 OpenRouter 모델을 연결한 과정"
img: command-title.webp
date: 2026-07-25 12:50:00 +0900
last_modified_at: 2026-07-25 12:50:00 +0900
tags: [hermes, ai agent, openrouter, cli, nous research, llm] # add tag
related: llm
categories: tools
---

Claude Code, OpenAI Codex 같은 자율형 코딩 에이전트가 많아지는 가운데, **Nous Research의 [Hermes Agent](https://github.com/NousResearch/hermes-agent)**는 "어떤 LLM이든 붙이고, 어디서든 돌아가는" 방향으로 눈에 띈다. Windows 11 노트북에 Hermes를 설치하고 기본 모델(hy3)을 설정한 뒤 OpenRouter 모델까지 연결한 과정을 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 직접 작성했다.)

<!--more-->

> **TL;DR:** git-bash에서 설치 스크립트 한 줄 → `hermes setup` 마법사로 프로바이더·모델 연결 → `hermes`로 실행. 프로바이더 무관 설계라 OpenRouter 하나만 연결해도 수십 개 모델을 갈아끼우며 쓸 수 있다. 설정은 `config.yaml`, 비밀키는 `.env`로 분리하는 것이 하드 규칙이고, Windows에서는 줄바꿈(`Ctrl+Enter`)·BOM·경로 표기 등 몇 가지 특이점만 알아두면 된다.

## Hermes Agent란?

터미널, 데스크톱 앱, 메신저(텔레그램/디스코드/슬랙), IDE까지 **같은 에이전트 코어를 공유**하는 오픈소스 에이전트 프레임워크다 (GitHub ⭐22만). 차별점은 다음과 같다.

- **프로바이더 무관성**: OpenRouter, Anthropic, OpenAI, Google, DeepSeek, xAI, 로컬 모델 등 20개 이상을 자유롭게 교체
- **스킬 기반 자가 학습**: 반복 절차를 skill로 저장해 다음 세션에 자동 로드
- **세션 간 영속 메모리**: 사용자 선호, 환경 정보, 교훈을 기억
- **멀티 프로필**: 독립된 설정·세션·메모리로 여러 인스턴스 운용

"한 번 쓴 도구 호출 경험을 다음에도 쓴다"는 점이 코딩 워크플로우에서 특히 유용하다.

## 설치 (Windows)

Linux/macOS/Windows/WSL을 모두 지원한다. Windows에서는 **PowerShell 네이티브 설치 스크립트**를 제공하며, 이번 설치도 이 방법을 사용했다. 설치기가 uv, Python, venv, 실행 런처까지 한 번에 세팅해준다.

```powershell
# Windows (PowerShell 네이티브)
iex (irm https://hermes-agent.nousresearch.com/install.ps1)
```

git-bash(MSYS)나 WSL 환경이라면 셸 스크립트 버전을 쓴다.

```bash
# Linux / macOS / WSL / git-bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

GUI를 선호하면 공식 사이트의 Hermes Desktop 설치 파일을 받아도 된다. 설치 후 `hermes --help`로 정상 설치를 확인한다.

## 초기 설정 — 모델·프로바이더 연결

설치 직후 대화형 마법사로 모델과 API 키를 연결한다.

```bash
hermes setup     # 모델·프로바이더 선택 마법사
hermes model     # 모델/프로바이더 변경
hermes doctor    # 상태 점검 (health check)
```

이 노트북에서는 기본 모델로 `tencent/hy3`를 설정하고, 프로바이더로 **OpenRouter**를 연결해 다른 모델들도 추가했다. hy3는 [OpenRouter 무료 모델 목록]({{site.baseurl}}/tools/2026/07/15/openrouter_free.html)에도 있는 모델이라, 키 하나로 부담 없이 시작해서 필요할 때 상위 모델로 갈아타는 흐름이 자연스럽다.

설정 관리에는 하드 규칙이 있다: **설정값(모델명, 인터페이스 등)은 `config.yaml`에, API 키 같은 비밀은 `~/.hermes/.env`에** 분리해서 둔다. 그리고 `config.yaml`은 직접 손편집하지 말 것 — 들여쓰기 하나로 라이브 게이트웨이가 깨질 수 있으니 항상 `hermes config set KEY VAL` 명령을 쓴다.

## 실행해 보기

```bash
hermes                            # 대화형 실행 (기본 CLI)
hermes chat -q "프랑스의 수도는?"  # 단발 쿼리
hermes desktop                    # 네이티브 데스크톱 앱
hermes dashboard                  # 웹 관리 패널 + 임베디드 채팅
```

터미널 UI(TUI)를 기본으로 쓰려면 `hermes config set display.interface tui`로 바꾼다.

## OpenRouter 모델 추가하기

키 하나로 수많은 모델을 골라 쓸 수 있어, OpenRouter를 붙여두면 모델 실험이 편해진다. 추가 절차는 네 단계다.

**① API 키 등록** — 키는 반드시 `~/.hermes/.env`에만 둔다 (`config.yaml`에 적지 않는다).

```bash
# ~/.hermes/.env
OPENROUTER_API_KEY=sk-or-v1-xxxxxxxxxxxxxxxx
```

**② 모델 선택** — 마법사에서 고르거나, `provider/model` 형식으로 직접 지정한다.

```bash
hermes setup     # 프로바이더 중 openrouter 선택
hermes model     # 모델 피커에서 원하는 OpenRouter 모델 선택
hermes config set model.main openrouter/tencent/hy3   # 직접 지정
```

**③ 자주 쓰는 모델은 별칭으로** — 세션 내 전환은 `/model <이름>`, 기본값 저장은 `--global`을 붙인다.

```bash
/model openrouter/anthropic/claude-sonnet-4.6            # 세션 내에서만
/model openrouter/anthropic/claude-sonnet-4.6 --global   # 기본값으로 영속
```

매번 풀 네임을 치기 싫으면 `config.yaml`의 `model.aliases`에 등록한다 (`hermes config set` 명령 사용).

```yaml
model:
  aliases:
    fav: openrouter/anthropic/claude-sonnet-4.6
```

**④ 상태 점검** — `hermes doctor`로 API 키·모델 연결 상태를 확인한다.

## Windows에서 걸리는 몇 가지

네이티브 실행 시 특이점들이다. 미리 알아두면 디버깅 시간을 아낀다.

- **줄바꿈 입력**: `Alt+Enter`는 터미널이 풀스크린 전환으로 가로채므로 **`Ctrl+Enter`**를 쓴다.
- **BOM 문제**: 첫 실행에서 `HTTP 400 "No models provided"`가 뜨면 `config.yaml`이 UTF-8 BOM으로 저장된 경우다. `hermes config edit`로 다시 저장하면 해결된다.
- **경로 표기**: `C:/Users/...`처럼 슬래시 표기가 거의 모든 도구·API에서 통한다. 백슬래시보다 슬래시 권장.
- **execute_code 샌드박스**: `WinError 10106`이 나면 환경변수(`SYSTEMROOT` 등) 스크럽 문제다. 블록 안에서 `os.environ`을 찍어 확인한다.

## 마무리

"설치 → `hermes setup`으로 모델 연결 → `hermes` 실행"의 단순한 흐름으로 시작할 수 있고, 프로바이더를 자유롭게 바꿀 수 있어 모델 실험이 잦은 워크플로우에 잘 맞는다. 다음 글에서는 스킬 저장과 멀티 에이전트 오케스트레이션(`delegate_task`, cron) 활용법을 다뤄볼 예정이다.

## 참고

- [Hermes Agent 공식 문서](https://hermes-agent.nousresearch.com/docs/){:target="_blank"}
- [GitHub: NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent){:target="_blank"}
- [OpenRouter 무료 모델을 API로 사용하기 (이전 글)]({{site.baseurl}}/tools/2026/07/15/openrouter_free.html)
