---
layout: post
comments: true
title: "Andrej Karpathy의 Autoresearch Loop — AI가 AI 연구를 자율 수행하는 700회 실험의 기록"
description: "Karpathy의 GitHub autoresearch 리포 분석. 2일간 700회 실험, 20개 최적화 자율 발견, 11% 학습 속도 향상. program.md=스킬 명세 원형, train.py 단일 파일 수정 루프, val_bpb 단일 메트릭 5분 타임박스 구조를 해설한다."
img: karpathy-autoresearch.webp
date: 2026-08-30 00:48:00 +0900
last_modified_at: 2026-08-30 21:25:00 +0900
tags: [karpathy, autoresearch, autonomous-ai, ai-research, self-improving-ai, llm-agents, mlops] # add tag
related: llm
categories: dev
---

Andrej Karpathy가 2026년 3월 공개한 [**autoresearch**](https://github.com/karpathy/autoresearch){:target="_blank"}는 "AI 에이전트에게 작지만 실제로 돌아가는 LLM 학습 셋업을 주고 하룻밤 자율 실험하게 하기"라는 아이디어를 700회 실험, 20개 최적화 발견, 더 큰 모델에 11% 학습 속도 향상 전이라는 결과로 증명했다. 이 글은 리포의 3개 핵심 파일(`prepare.py`, `train.py`, `program.md`)과 Fortune 기사, 커뮤니티 재현 결과를 종합해 자율 연구 루프의 전체 작동 원리를 해설한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Karpathy의 autoresearch는 **단일 GPU, 5분 타임박스, val_bpb 단일 메트릭** 제약 하에서 `train.py` 하나만 수정하며 **수정→실행→평가→유지/복구** 루프를 700회 자율 반복했다. 핵심은 `program.md`라는 초경량 스킬 명세가 에이전트에게 제약·목표·워크플로·출력 형식·루프 로직을 모두 인코딩했다는 점이다. 현대적 AI 스킬 시스템(Claude Code 스킬, 각종 에이전트 프레임워크의 SKILL.md 등)의 최소 완전 원형이라 할 만하다.

## 1. Autoresearch란 무엇인가?

| 항목 | 값 |
|------|------|
| **리포** | `karpathy/autoresearch` (94.9k star, 13.4k fork) |
| **핵심 아이디어** | "AI 에이전트에게 작지만 실제 LLM 학습 셋업을 주고 하룻밤 자율 실험하게 하기" |
| **실행 기간** | 2일 연속 (2026년 3월) |
| **총 실험** | **700회** |
| **채택된 최적화** | **20개** (keep) |
| **전이 성능** | 동일 20개 트윅을 더 큰 모델에 적용 → **학습 시간 11% 단축** |
| **평가 메트릭** | `val_bpb` (validation bits per byte) — 낮을수록 좋음, 어휘 크기 독립적 |
| **타임박스** | 실험당 **고정 5분** (wall-clock, 시작/컴파일 제외) |

Karpathy는 Fortune 인터뷰에서 이렇게 말했다. *"목표는 단일 박사과정 학생을 에뮬레이션하는 게 아니다. 그들의 연구 커뮤니티 전체를 에뮬레이션하는 것이다."* 비동기 대규모 협업 에이전트 스웜으로의 확장을 예고한 셈이다.

## 2. 시스템 아키텍처: 3개 파일만으로 완성

```
autoresearch/
├── prepare.py      # 고정 — 상수, 데이터 준비, 토크나이저, 데이터로더, 평가 하네스
├── train.py        # 에이전트 수정 대상 — 모델, 옵티마이저, 학습 루프 전부
├── program.md      # 사람 작성 — 에이전트에게 주는 스킬/명세서
├── analysis.ipynb  # 결과 분석 노트북
├── pyproject.toml  # uv 기반 의존성
└── README.md
```

| 파일 | 수정 권한 | 핵심 책임 |
|------|-----------|-----------|
| **prepare.py** | 금지 | 데이터 다운로드, BPE 토크나이저, 데이터로더, `evaluate_bpb` 평가, 하드웨어 상수 |
| **train.py** | 자유 | GPT 아키텍처, Muon+AdamW, 학습 루프, 배치/모델 크기, 어텐션 패턴 — 전부 수정 가능 |
| **program.md** | 사람 작성 | 에이전트 지시서: 설정, 규칙, 목표, 출력 형식, 실험 루프, 로깅, 크래시 처리 |

## 3. program.md — 스킬 명세의 최소 완전 원형

### 3.1 설정 단계 (Setup)

1. **실행 태그 합의**: 날짜 기반(`mar5`), 브랜치 `autoresearch/<tag>` 미존재 확인
2. **브랜치 생성**: master에서 `git checkout -b autoresearch/<tag>`
3. **인스코프 파일 읽기**: README, prepare.py, train.py
4. **데이터 확인**: `~/.cache/autoresearch/` 샤드+토크나이저 — 없으면 `uv run prepare.py`
5. **results.tsv 초기화**: 헤더만 생성 (베이스라인은 첫 실행 후 기록)

### 3.2 실험 실행 규칙

| 허용 (CAN) | 금지 (CANNOT) |
|-----------|---------------|
| `train.py` 수정 — 아키텍처, 옵티마이저, 하이퍼파라미터, 배치, 모델 크기 전부 | `prepare.py` 수정 — 읽기 전용 |
| | 새 패키지 설치 / 의존성 추가 — `pyproject.toml` 기존 것만 |
| | 평가 하네스 수정 — `evaluate_bpb`는 그라운드 트루스 |

목표는 단 하나, 최저 `val_bpb`다. 시간이 고정이므로 학습 시간을 걱정할 필요가 없다.

설계 철학의 핵심인 **단순성 기준(Simplicity Criterion)**은 이렇다.

> "동일 조건이면 단순함이 낫다. 못생긴 복잡성을 추가해서 얻는 약간의 개선? 가치 없음. 코드를 삭제했는데 동일하거나 더 나은 결과? 대환영. 개선이 거의 0인데 코드가 훨씬 단순해졌다? 무조건 킵."

### 3.3 실험 루프 — 자율 루프의 전체 알고리즘

```markdown
LOOP FOREVER:
1. git 상태 확인: 현재 브랜치/커밋
2. 실험 아이디어로 train.py 직접 수정
3. git commit
4. 실험 실행: `uv run train.py > run.log 2>&1` (출력 리다이렉트 — 컨텍스트 오염 방지)
5. 결과 추출: `grep "^val_bpb:\|^peak_vram_mb:" run.log`
6. grep이 비었으면 크래시 → `tail -50 run.log`로 스택트레이스 읽고 수정 시도
   - 몇 번 실패하면 포기
7. results.tsv 기록 (git 추적 안 함 — untracked)
8. val_bpb 개선(감소) → 브랜치 "전진" (커밋 유지)
9. val_bpb 동일/악화 → git reset으로 원상 복구
```

타임아웃은 5분 목표에 10분 초과 시 강제 종료 후 실패 처리다. 크래시가 나면 타이포나 import 누락 같은 단순 버그는 수정 후 재실행하고 아이디어 자체가 결함이면 "crash"로 기록하고 다음으로 넘어간다.

그리고 **NEVER STOP 원칙**이 있다.

> "실험 루프 시작 후 인간에게 '계속할까?' 묻지 마라. 인간은 자고 있을 수 있다. 자율적으로 무한히 계속하라. 아이디어가 떨어지면 더 깊이 생각하라 — 논문을 읽고, 파일을 다시 읽고, 아슬아슬하게 실패한 시도들을 조합하고, 급진적인 아키텍처 변경을 시도하라. 인간이 수동으로 중단할 때까지 영원히 돈다."

## 4. 기술 스택 상세 (program.md + README 기반)

### 4.1 고정된 학습 셋업 (prepare.py)

- **데이터**: nanochat 스타일 데이터, 파일당 샤드 분할
- **토크나이저**: BPE, vocab_size=8192
- **시퀀스 길이**: `MAX_SEQ_LEN` (기본 1024)
- **배치**: `DEVICE_BATCH_SIZE`, `TOTAL_BATCH_SIZE` (2의 거듭제곱)
- **평가**: `EVAL_TOKENS` 토큰으로 검증 손실 계산
- **어텐션**: `WINDOW_PATTERN` = "SSSL" (밴디드 어텐션 교차 패턴)

### 4.2 모델·옵티마이저 (train.py에서 수정 가능)

- **기본**: Depth=8의 소형 GPT
- **옵티마이저**: Muon + AdamW 하이브리드
- **학습률**: 스케줄링 포함 (에이전트 조정 가능)
- **정규화**: Weight decay, gradient clipping

### 4.3 평가 메트릭: `val_bpb`

검증 데이터에서 재는 바이트당 비트 수다. 낮을수록 좋다. 어휘 크기에 독립적이어서 임베딩 차원 같은 아키텍처를 바꿔도 공정하게 비교할 수 있다.

## 5. 실험 결과: 700회 루프가 낳은 것

### 5.1 Karpathy 본인 실행 (2026-03)

| 지표 | 값 |
|------|-----|
| 총 실험 | **700회** (2일간) |
| 채택된 최적화 (keep) | **20개** |
| **전이 검증** | 더 큰 모델에 동일 20개 트윅 적용 → **학습 시간 11% 단축** |

### 5.2 커뮤니티 재현

- **Tobi Lütke(Shopify CEO)**: 하룻밤 37회 실험으로 19% 성능 향상 재현
- **포크**: macOS(MPS/MLX), Windows(RTX), AMD(ROCm) 대응 포크 존재 (아래 재현 가이드 참조)

## 6. AutoML과의 결정적 차이 — Karpathy 본인 반박

| 측면 | 기존 AutoML (NAS 등) | Autoresearch |
|------|---------------------|--------------|
| **탐색 전략** | 랜덤/진화/베이지안 최적화 | LLM이 코드 읽고 가설 세워 직접 수정 |
| **지식 활용** | 과거 실험 수치만 | 논문 읽고, 코드 이해하고, 가설 형성 |
| **수정 범위** | 정의된 서치 스페이스 내 | train.py 전체 — 임의 코드 변경 가능 |
| **인터넷 접근** | 보통 없음 | 웹 검색 가능 (논문, 블로그, 문서) |
| **학습 방식** | 블랙박스 함수 최적화 | 이유 있는 추론 + 시행착오 |

Karpathy는 Fortune 인터뷰에서 이렇게 단언했다. *"당시의 Neural architecture search는 이것에 비하면 완전히 쓸모없는 별도 카테고리다. 이건 임의의 코드를 쓰는 실제 LLM이 이전 실험에서 학습하고 인터넷에도 접근한다. 비교조차 안 된다."*

## 7. program.md는 "SKILL.md"의 원형이다

program.md의 구성을 뜯어 보면 오늘날 에이전트 스킬 시스템(Claude Code의 스킬, 각종 프레임워크의 SKILL.md)의 구성 요소와 정확히 겹친다.

| autoresearch | 스킬 시스템 일반 | 비고 |
|--------------|--------------|------|
| Setup 단계 | 초기화 워크플로 | 환경 검증, 작업공간 준비 |
| Experimentation 규칙 | rules / constraints | 할 수 있는 것과 없는 것 명시 |
| Simplicity Criterion | best practices | 복잡도-성능 트레이드오프 가이드 |
| Output format | 표준 출력 형식 | 결과 포맷 통일 |
| Logging (results.tsv) | 메트릭 로그 | 구조화된 실험 기록 |
| Experiment Loop | 자율 반복 워크플로 | 수정→실행→평가→유지/복구 |
| NEVER STOP | 자율 에이전트 모드 | 인간 개입 없이 무한 루프 |
| Crash handling | 에러 핸들링/재시도 | 수정 가능한 버그와 근본 결함 분류 |

114줄짜리 마크다운 하나가 제약, 목표, 워크플로, 출력, 루프를 모두 담았다. 스킬 명세의 최소 기능 완전 세트다.

## 8. 재현 가이드

```bash
# 1. uv 설치
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. 리포 클론
git clone https://github.com/karpathy/autoresearch
cd autoresearch

# 3. 의존성 설치
uv sync

# 4. 데이터 준비 + 토크나이저 (~2분)
uv run prepare.py

# 5. 단일 실험 수동 실행 (~5분) — 베이스라인 확인
uv run train.py

# 6. 에이전트 자율 실행 (Claude Code / Codex 등)
#    리포에서 에이전트 실행 후:
#    "Hi, have a look at program.md and let's kick off a new experiment! let's do the setup first."
```

소형 하드웨어용 포크: `miolini/autoresearch-macos`(MPS), `trevin-creator/autoresearch-mlx`(MLX), `jsegov/autoresearch-win-rtx`(CUDA), `andyluo7/autoresearch`(ROCm)

## 9. 한계점 및 비판적 시각

| 한계 | 설명 |
|------|------|
| 단일 GPU / 소형 모델 | 프론티어 모델(수천 GPU 학습)과의 갭이 크다 |
| 고정 5분 타임박스 | 대형 모델에선 의미 있는 학습이 불가능하다 |
| 단일 메트릭(val_bpb) | 실제 서비스는 다중 목적 최적화가 필요하다 |
| 인간 피드백 루프 없음 | 완전 자율이라 사람의 선호가 반영되지 않는다 |
| 실험 트래킹 부재 | results.tsv는 git 추적조차 안 된다 — MLflow/W&B급 트래킹과는 거리가 있다 |

## 10. 마무리: autoresearch가 증명한 것

1. LLM이 연구자처럼 사고하며 코드를 수정할 수 있다 — 임의의 Python을 읽고, 가설을 세우고, 실험하고, 결과에서 학습한다
2. 수정→실행→평가→유지/복구라는 단순 루프만으로 700회를 자율 수행했다 — 인간 개입 제로
3. 발견한 최적화가 더 큰 모델로 전이됐다(11% 속도 향상) — 소형 실험의 대형 일반화 검증
4. `program.md`는 스킬 명세의 최소 완전 세트다 — 제약, 목표, 워크플로, 출력, 루프를 모두 포함
5. 다음 단계는 "단일 연구자"에서 "연구 커뮤니티"로 확장하는 것이다 — 에이전트 간 발견 공유가 관건

## 참고 자료

| 자료 | 링크 |
|------|------|
| GitHub 리포 | [karpathy/autoresearch](https://github.com/karpathy/autoresearch){:target="_blank"} |
| program.md 전문 | [program.md](https://github.com/karpathy/autoresearch/blob/master/program.md){:target="_blank"} |
| Fortune 기사 | [Fortune 2026-03-17](https://fortune.com/2026/03/17/andrej-karpathy-loop-autonomous-ai-agents-future/){:target="_blank"} |
| The New Stack 해설 | [Karpathy's Autonomous Experiment Loop](https://thenewstack.io/karpathy-autonomous-experiment-loop/){:target="_blank"} |
