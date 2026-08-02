---
layout: post
comments: true
title: "오케스트레이션이란 무엇인가 — 루프에서 그래프로, 에이전트 협업의 구조 이해하기"
description: "루프 엔지니어링과 그래프 엔지니어링의 차이, 오케스트레이터 런타임(Run/Task/Dispatch/Gate)의 구조, 그리고 오픈소스 스택으로 직접 구현하는 최소 구성"
img: multi_agent_network.jpg
date: 2026-08-02 23:44:00 +0900
last_modified_at: 2026-08-03 00:07:00 +0900
tags: [ai-agent, orchestration, graph-engineering, orca, multi-agent, llm] # add tag
related: llm
categories: dev
---

최근 유튜브에서 **오르카(Orca) IDE**로 그래프 엔지니어링을 시연하는 영상([-pk2umNC-18](https://youtu.be/-pk2umNC-18))을 봤다.  
클로드 코드(Claude Code) 하나가 **오케스트레이터**가 돼서, 코덱스(Codex) 워커 5개를 병렬로 띄워 허깅페이스 블로그 글 5개를 각각 요약시키고,  
다시 다른 코덱스 세션이 요약본을 모아 트위터용 초안을 쓰고, 마지막 클로드 코드 세션이 **사람(또는 에이전트) 검토**를 맡는 —  
전형적인 **다중 에이전트 그래프** 실행 흐름이었다.

이 영상을 보고 "오케스트레이션이 대체 뭔데?" 하고 구조를 뜯어보기 시작했다.  
이 글은 **오케스트레이션의 핵심 개념을 엔지니어 관점에서 정리한 메모**다. (이 글의 초안은 설치된 Hermes 에이전트가 `nemotron-3-ultra-550b-a55b` 모델로 작성했다.)

<!--more-->

> **TL;DR:** 루프 엔지니어링(단일 에이전트 반복)과 그래프 엔지니어링(다중 에이전트 협업)의 차이를 정리하고, 오케스트레이터 런타임(Run/Task/Dispatch/Message/Gate) 구조를 오르카 문서 기준으로 분석했다. 오픈소스 모델로 직접 구현하려면 State Store·Message Bus·Worker Spawner·Coordinator Loop 네 조각부터 시작하면 된다.

## 1. 루프(Loop) 엔지니어링 — 가장 작은 단위

> **"하나의 에이전트가 목표를 향해 계획→실행→검토를 반복하는 구조"**

```text
[트리거: 스케줄/이벤트] → [계획] → [실행] → [검토: 종료조건 만족?] 
                                    ↑_____________________________│
                                              (아니면 다시 계획)
```

- 클로드 코드 `goal`, 코덱스 `goal`, 랭체인 `AgentExecutor` 등이 이 패턴
- **단일 에이전트, 단일 컨텍스트** 안에서 끝난다
- 한계: 작업이 커지면 컨텍스트 오염, 도구 호출 한도, 추론 품질 저하 발생

---

## 2. 그래프(Graph) 엔지니어링 — 루프를 노드로 연결

> **"하나의 루프로는 부족할 때, 여러 특화 에이전트(노드)를 그래프로 엮는 구조"**

영상 예시 그래프:

```text
T1~T5 (병렬) → T6 (선정·작성) → T7 (검토) → (반려 시 T6 재실행) → 완료
  │              │                │
  ▼              ▼                ▼
Codex×5      Codex×1           Claude×1 (사람 검토)
요약         트윗 초안           승인/반려
```

- **노드** = 특화된 에이전트(또는 결정론적 함수, 사람 체크포인트)
- **엣지** = 상태(state) 전달 + 라우팅 규칙(조건부 분기 포함)
- **상태 객체** = 그래프를 타고 흐르며 누적되는 공유 메모리

핵심 차이:

| 차원 | 루프 엔지니어링 | 그래프 엔지니어링 |
|------|----------------|------------------|
| **주체** | 단일 에이전트 | 다중 에이전트/노드 |
| **제어권** | 에이전트가 경로 결정 | 설계자가 토폴로지 정의, 에이전트는 노드 내부만 자율 |
| **상태** | 대화 히스토리(선형) | 구조화된 상태 객체(DAG 위 흐름) |
| **확장** | 컨텍스트 한계 직면 | 병렬·분기·재시도·사람개입 자연스럽게 표현 |

> **"그래프는 루프를 대체하는 게 아니라, 루프를 노드로 품는 상위 계층이다"**  
> — [Graph Engineering Explained](https://www.louisbouchard.ai/graph-engineering-explained/)

---

## 3. 오케스트레이터(Orchestrator) — 그래프의 런타임 주인

그래프를 **실제로 돌리는 주체**가 오케스트레이터다.  
오르카 문서([Orchestration](https://www.onorca.dev/docs/cli/orchestration))가 정의하는 핵심 엔티티:

| 엔티티 | 역할 |
|--------|------|
| **Run** | 네임스페이스 + 코디네이터 인박스(메시지 큐) |
| **Task** | 작업 명세(spec), 의존성, 상태(`pending→ready→dispatched→completed/failed/blocked`) |
| **Dispatch** | Task의 **한 번 실행 시도**, 특정 터미널/워커에서 생명주기 소유 |
| **Message** | 인박스 메일(`worker_done`, `escalation`, `question`, `heartbeat`…) |
| **Decision Gate** | 코디네이터가 생성하는 블로킹 질문, Task를 대기시킴 |

메시지 플로우(단순화):

```text
Coordinator (오케스트레이터 에이전트)
    │ run-create → task-create×N → worker-start×N
    ▼
Worker A (Codex) ── worker_done ──┐
Worker B (Codex) ── worker_done ──┤→ check --wait --ack (FIFO 처리)
Worker C (Claude) ─ worker_done ──┘
    │
    ▼
Gate 생성(필요시) → 사람/에이전트 판정 → 재시도 또는 완료
```

**워커 계약(Worker Contract)** — 오르카가 강제하는 프로토콜:
1. `worker_done` **딱 한 번** 전송 (`--outcome succeeded|failed` 필수)
2. 요약 본문(`--body`) 포함: 뭐 했는지, 뭐 남았는지
3. `taskId` + `dispatchId` 동봉 (stale retry 방지)
4. 장시간 작업 시 `heartbeat` 주기적 전송
5. 질문은 로컬 TUI 대신 `orca orchestration ask --json` 사용

---

## 4. 왜 그래프/오케스트레이션이 필요한가?

### 4.1 컨텍스트 격리
- 요약 5개 → 각각 독립 컨텍스트에서 수행 → **오염 없음**
- 검토 노드는 요약본만 보면 됨 → **토큰 절약**

### 4.2 병렬 처리
- T1~T5 동시 실행 → **벽시계 시간 1/5로 단축**

### 4.3 명시적 라우팅·재시도
- 검토 반려 → T6만 재실행 (T1~T5 건드리지 않음)
- 게이트로 사람 개입 지점 명시 → **감사 가능·재현 가능**

### 4.4 전문화
- 요약엔 Codex, 작성엔 Codex, 검토엔 Claude(또는 사람) → **모델/프롬프트 최적화 분리**

---

## 5. 오픈소스 모델로 직접 만들려면?

오르카는 서명된 네이티브 바이너리로 배포되므로 내부 런타임을 뜯어 쓰긴 어렵다.  
대신 **같은 패턴을 직접 구현**하는 게 현실적이다.

### 최소 런타임 구성요소

| 컴포넌트 | 추천 스택 | 핵심 포인트 |
|----------|-----------|-------------|
| **State Store** | SQLite / Redis / etcd | Run/Task/Dispatch/Message CRUD + 낙관적 락 |
| **Message Bus** | NATS / Redis Streams / SQLite 트리거 | FIFO Delivery + Ack + Replay |
| **Worker Spawner** | `subprocess` + `pty` / SSH / Docker | `--agent codex|claude|opencode` 인자로 프로세스 격리 |
| **CLI Protocol** | JSON-RPC over stdin/stdout | `--json` 강제, Pydantic/jsonschema로 스키마 검증 |
| **Coordinator Loop** | asyncio (Python) / Go 단일 루프 | `check --wait` 폴링 → 메시지 디스패치 → 게이트 평가 |

### 코디네이터 에이전트 프롬프트 스켈레톤

```python
COORDINATOR_SYSTEM = """
당신은 오케스트레이션 코디네이터입니다.
- Run/Task/Dispatch 상태 머신을 이해하고 있습니다.
- 도구: run_create, task_create, worker_start, check, gate_create, gate_resolve
- 워커에게 preamble 주입: worker_done/heartbeat/ask 프로토콜 강제
- 모든 결정은 JSON 출력, 상태 변경 시 이벤트 로그 기록
"""
```

### 모델 배치 가이드 (2026-08 기준)

| 역할 | 추천 모델 | 이유 |
|------|-----------|------|
| **Coordinator (플래닝/라우팅)** | `qwen3-235b-a22b`, `nemotron-3-ultra`, `glm-4.5` | 긴 컨텍스트, 구조화 출력 우수 |
| **Worker (코딩/요약)** | `qwen3-coder-480b`, `deepseek-coder-v2`, `codellama-70b` | 코드 생성 특화 |
| **Gate/Judge (평가)** | `llama-3.1-70b-instruct`, `gemma-2-27b` | 지시 따름·일관성 좋음 |

> 로컬 실행: `llama.cpp` (GGUF) + `llama-server` → OpenAI-compatible 엔드포인트 → `openai` 파이썬 클라이언트로 통합

---

## 6. 프로토타입 50줄 스켈레톤 (Python asyncio + SQLite)

```python
# orchestrator/core.py
import asyncio, json, uuid, sqlite3
from dataclasses import dataclass
from typing import Literal

@dataclass
class Task:
    id: str
    spec: str
    status: Literal["pending","ready","dispatched","completed","failed","blocked"]
    depends_on: list[str]

class Orchestrator:
    def __init__(self, db_path="orchestrator.db"):
        self.conn = sqlite3.connect(db_path)
        self._init_schema()
        self.msg_bus = asyncio.Queue()

    def _init_schema(self):
        self.conn.executescript("""
        CREATE TABLE IF NOT EXISTS runs (id TEXT PRIMARY KEY, objective TEXT, created_at TEXT);
        CREATE TABLE IF NOT EXISTS tasks (id TEXT PRIMARY KEY, run_id TEXT, spec TEXT, status TEXT, depends_on TEXT);
        CREATE TABLE IF NOT EXISTS dispatches (id TEXT PRIMARY KEY, task_id TEXT, worker_handle TEXT, status TEXT);
        CREATE TABLE IF NOT EXISTS messages (id INTEGER PRIMARY KEY, run_id TEXT, type TEXT, payload TEXT, acked INTEGER DEFAULT 0);
        """)

    async def run_create(self, objective: str) -> str:
        run_id = f"run_{uuid.uuid4().hex[:8]}"
        self.conn.execute("INSERT INTO runs VALUES (?, ?, datetime('now'))", (run_id, objective))
        self.conn.commit()
        return run_id

    async def task_create(self, run_id: str, spec: str, depends_on: list[str] = None) -> str:
        task_id = f"task_{uuid.uuid4().hex[:8]}"
        self.conn.execute(
            "INSERT INTO tasks VALUES (?, ?, ?, 'pending', ?)",
            (task_id, run_id, spec, json.dumps(depends_on or []))
        )
        self.conn.commit()
        return task_id

    async def worker_start(self, task_id: str, agent: str, worktree: str) -> str:
        dispatch_id = f"disp_{uuid.uuid4().hex[:8]}"
        # preamble = {"taskId": task_id, "dispatchId": dispatch_id, "protocol": "worker_done/heartbeat/ask"}
        # subprocess.Popen([agent], cwd=worktree, stdin=PIPE, stdout=PIPE).stdin.write(json.dumps(preamble))
        self.conn.execute("INSERT INTO dispatches VALUES (?, ?, ?, 'running')", (dispatch_id, task_id, agent))
        self.conn.commit()
        return dispatch_id

    async def check_loop(self, run_id: str):
        while True:
            msg = await self.msg_bus.get()  # worker_done, escalation, question…
            await self._handle_message(msg)
            # 의존성 해제 시 ready→dispatched 전이, 게이트 평가 등
```

---

## 7. 정리 — 오케스트레이션을 한 문장으로

> **오케스트레이션 = "여러 자율 루프(에이전트)를 명시적 그래프 토폴로지 위에 올려, 상태·라우팅·재시도·사람개입을 런타임이 보장하게 만드는 것"**

- 루프 하나로는 안 될 때 **그래프로 쪼갠다**
- 그래프를 돌리는 **런타임(오케스트레이터)** 이 상태 머신·메시지 버스·워커 계약 강제
- 오픈소스로 만들려면 **State Store + Message Bus + Worker Spawner + Coordinator Loop** 네 조각부터 시작

---

## 8. 참고 자료

| 문서 | 링크 |
|------|------|
| Orca Orchestration 공식 가이드 | <https://www.onorca.dev/docs/cli/orchestration> |
| Orca Skills 레지스트리 | <https://www.onorca.dev/docs/cli/skills> |
| Orchestration SKILL.md (stub) | <https://raw.githubusercontent.com/stablyai/orca/main/skills/orchestration/SKILL.md> |
| Graph Engineering Explained (Louis Bouchard) | <https://www.louisbouchard.ai/graph-engineering-explained/> |
| 3 Years of Graph Engineering with LangGraph | <https://www.langchain.com/blog/3-years-of-graph-engineering-with-langgraph> |
| Graph Engineering for Multi-Agent Systems (TrueFoundry) | <https://www.truefoundry.com/blog/graph-engineering-enterprise-guide> |
| AI Builder Club Graph Engineering Guide | <https://www.aibuilderclub.com/blog/graph-engineering-guide-2026> |
| 원본 유튜브 영상 | <https://youtu.be/-pk2umNC-18> |

---

## 9. 마무리

이 글에서 다룬 오케스트레이션 개념은 앞서 작성한 글들과 연결된다:

- [Hermes Agent 오케스트레이션 개요]({{site.baseurl}}/tools/2026/07/25/hermes_orchestration.html) — Hermes의 `delegate_task`, `cronjob` 스킬로 멀티에이전트 패턴 구현
- [Caller/Executor 패턴]({{site.baseurl}}/dev/2026/07/16/caller_executor_agent.html) — 단일 루프 내 호출자/실행자 분리 패턴

다음 글에서는 실제 프로토타입을 돌려보며 **State Store(SQLite 스키마) + Message Bus(NATS) + Worker Spawner(llama.cpp 로컬 서빙) + Coordinator Loop(프롬프트 튜닝·관찰가능성)** 네 조각을 하나씩 붙여나가는 구현 일지를 쓸 예정이다.

---

*이 글은 개인 연구 메모를 정리한 것이며, 언급된 도구·모델·링크는 작성 시점(2026-08-02) 기준입니다.*
