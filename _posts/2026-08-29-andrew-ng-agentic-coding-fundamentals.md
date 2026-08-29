---
layout: post
comments: true
title: "Andrew Ng가 말한 '에이전트 시대의 소프트웨어 엔지니어링 기초' — 코딩 에이전트가 대체할 수 없는 것들"
description: "Andrew Ng X 포스트(2026-08-28) 분석. 에이전트 코딩이 구현 비용을 낮춰도 아키텍처·데이터·트레이드오프 판단은 여전히 개발자 몫이다. 풀스택, 데이터 관리, 시스템 설계, 보안/신뢰성, 프로덕션 운영 5대 역량을 정리한다."
img: andrew-ng-agentic-coding.webp
date: 2026-08-29 13:38:00 +0900
last_modified_at: 2026-08-29 13:38:00 +0900
tags: [andrew-ng, agentic-coding, software-engineering, ai-engineering, llm-ops, architecture, data-management] # add tag
related: llm
categories: dev
---

Andrew Ng가 2026년 8월 28일 뉴스레터 The Batch와 [X](https://x.com/AndrewYNg/status/2093388974194872781){:target="_blank"}에 올린 에세이 ["How have software engineering fundamentals changed with agentic coding?"](https://www.deeplearning.ai/the-batch/issue-368/){:target="_blank"}은 에이전트 코딩 시대에 개발자가 **어떤 역량을 갖춰야 하는지**를 5가지 영역으로 정리한 명문이다. X에서 좋아요 4천여 회를 받았다. 핵심 메시지는 단순하다. **"에이전트가 코드를 다 짜줘도, 소프트웨어 기초를 이해 못 하면 나쁜 트레이드오프를 인지조차 못하고 에이전트에게 맡기게 된다."** (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 원문 대조 후 발행했다.)

<!--more-->

> **TL;DR:** Andrew Ng는 에이전트 코딩 시대에도 **소프트웨어 엔지니어링 기초(풀스택, 데이터 관리, 아키텍처 설계, 보안/신뢰성, 프로덕션 운영)**가 더 중요해진다고 주장한다. 구현 비용은 낮아졌지만 **지연시간·가용성·일관성·신뢰성·유지보수성·단순성·비용** 같은 트레이드오프 판단은 여전히 사람 몫이다. "바이브 코딩"만 하는 주니어는 돌아가는 코드를 만들지만 숨은 기술 부채와 장애 시나리오를 놓친다.

## 1. 포스트 배경과 핵심 주장

Andrew Ng는 DeepLearning.AI 창업자, Coursera 공동 창업자, 스탠포드 교수, 전 구글 브레인/바이두 수석 과학자로 AI 교육·연구·산업 모두에서 중심 역할을 해왔다. 그가 2026년 8월 말 **"AI Engineering Skills"** 연구 결과를 공유하며 쓴 이 글은 **코딩 에이전트(GitHub Copilot, Cursor, Claude Code, Codex 등)가 보편화된 지금** 개발자 역량 지도가 어떻게 바뀌어야 하는지를 짚는다.

> **"A novice who vibe codes without understanding software fundamentals can create simple applications, but this often leads to the coding agent making bad tradeoffs in latency, availability, consistency, reliability, maintainability, simplicity, and/or cost."**
>
> — Andrew Ng, X 포스트 본문 중

핵심은 **"트레이드오프를 아는가?"**다. 에이전트는 구현을 가속하지만 **어떤 트레이드오프가 존재하는지, 내 컨텍스트에서 무엇이 옳은지**는 개발자가 판단해야 한다. 모르면 에이전트가 **모르게** 결정한다.

## 2. 5대 핵심 역량 영역 상세

Andrew Ng가 정의한 **"AI Engineering Skills"** 5가지 영역을 온프레미스 LLM 플랫폼 운영 관점에서 재해석한다.

### 2.1 풀스택 애플리케이션 구축 (Building Full-Stack Applications)

| 기존 전문성 | 에이전트 시대 변화 | 필수 이해 요소 |
|-------------|-------------------|----------------|
| 프론트엔드/백엔드/모바일 분리 | 역할 경계 붕괴 → 에이전트가 낯선 파트 보조 | UI 컴포넌트, 캐싱, 페이지 렌더링(SSR/CSR/ISR), API 설계(REST/gRPC/GraphQL), 인증·세션 관리, 비동기 처리, 데이터 영속성, 테스트, 보안, 접근성 |

**온프레미스 관점**: vLLM·KServe 같은 서빙 스택을 운영한다면 대시보드부터 모델 서빙·오케스트레이션까지 **풀스택 이해**가 배포·디버깅·비용 최적화에 직결된다.

### 2.2 데이터 관리 (Managing Data) — **가장 강조된 영역**

> **"Your AI systems will get their own input context from your data source, so if data architecture is chosen poorly, the AI doesn't know what it doesn't know."**

| 하위 주제 | 실무 포인트 |
|-----------|-------------|
| 액세스 패턴 기반 저장 결정 | 읽기/쓰기 비율, 쿼리 패턴, 일관성 요구도 → RDB/문서/KV/그래프 선정 |
| 트랜잭션·동시성·데이터 품질 | ACID vs BASE, 정합성 검증, 스키마 진화(마이그레이션) |
| 프라이버시·거버넌스·컴플라이언스 | PII 마스킹, 접근 제어, 감사 로그, 데이터 라인리지 |
| **에이전트용 데이터 인프라** | 청크 전략, 메타데이터(문서 ID·조문 번호), 버전 관리, **RAGAS Context Recall 보완** |

**온프레미스 관점**: RAG에서는 청크 크기·오버랩·메타데이터 스키마 같은 데이터 설계가 **Faithfulness/Context Recall**을 직접 좌우한다 — 측정 방법은 [RAGAS 평가 방법론]({{site.baseurl}}/dev/2026/08/29/ragas-evaluation-methodology.html) 글에서 다뤘다. "AI는 자기가 모르는 걸 모른다"는 Ng의 지적이 정확히 이 지점이다.

### 2.3 시스템 아키텍처 설계 (Designing System Architectures)

| 결정 축 | 고려 사항 | 에이전트 시대 팁 |
|---------|-----------|------------------|
| 애플리케이션 플랫폼 | K8s(KServe/Knative) vs VM vs 서버리스 | 프로토타입→프로덕션→스케일 단계별 아키텍처 진화 경로 명시 |
| 프론트엔드-백엔드 경계 | BFF 패턴, API 게이트웨이, 상태 배치 | 에이전트에게 **"아키텍처 결정 기록(ADR)"** 형태로 컨텍스트 제공 |
| 분해·모듈러리티 | 모놀리스 → 모듈러 모놀리스 → 마이크로서비스 | 도메인 주도 설계(DDD) 바운디드 컨텍스트 기반 분해 |
| 스택 선정 | 언어/런타임/프레임워크/데이터 기술 | **스파이크(PoC) 후 결정** → 에이전트로 PoC 가속 |

**온프레미스 관점**: GPU 분할·오토스케일링·모니터링 같은 인프라 결정을 **ADR(아키텍처 결정 기록)**로 남기고 에이전트에게 컨텍스트로 주입하면 에이전트가 낸 변경이 기존 결정과 어긋나는 것을 막을 수 있다.

### 2.4 보안·신뢰성 확보 (Making Systems Secure and Reliable)

| 영역 | 기존 | 에이전트 시대 추가 |
|------|------|-------------------|
| 테스트 전략 | 단위/통합/커버리지 | **에이전트 생성 코드 대상**: "조용히 잘못된 동작" 탐지용 속성 기반 테스트, 메타모픽 테스트 |
| 장애 대응 | 재시도/서킷 브레이커 | **블래스트 레디우스 최소화**: 에이전트 호출 실패 시 그레이스풀 디그레이션(폴백 모델/캐시/정적 응답) |
| 시프트 레프트 보안 | SAST/DAST/SCA | **AI 보안 도구 활용**: 코드 취약점 스캔, 공급망 의존성 검사, 클라우드 설정 공격 표면 분석 → **보안 지식 기반 필수** |

덧붙이면 — 에이전트 생성 코드의 버그는 "안 돌아간다"보다 **"조용히 잘못 돌아간다"** 쪽이 많아서 테스트의 무게중심이 실행 여부 확인에서 동작 검증으로 옮겨간다.

### 2.5 프로덕션 운영·스케일링 (Scaling and Operating in Production)

| 단계 | 핵심 역량 | 자동화/도구 |
|------|-----------|-------------|
| 배포 | SDLC 실행, 릴리즈 전략(카나리/블루그린), CI/CD, IaC | ArgoCD/GitOps, Helm/Kustomize |
| 운영 | 옵저버빌리티(메트릭/로그/트레이스), 알림, 인시던트 대응 | Prometheus/Grafana/DCGM, 온콜 로테이션, 런북 |
| 스케일링 | 로드밸런싱, 샤딩/인덱싱/레플리케이션, 아키텍처 변경 | KEDA 이벤트 드리븐 스케일링, vLLM 프리픽스 캐싱 |
| 유지보수 | 버전 관리, 코드 리뷰, 의존성 업데이트, 기술 부채 관리 | Dependabot/Renovate, 아키텍처 피트니스 함수 |

**온프레미스 관점**: 이 블로그의 [주간 논문 서베이·일간 초안 검수 자동화]({{site.baseurl}}/tools/2026/08/24/daily_draft_pipeline_humanize.html)처럼, 반복 운영 업무를 에이전트 파이프라인으로 돌리되 품질 게이트(회귀 평가·검수 규칙)를 함께 두는 것이 이 영역의 실천이다.

## 3. 포스트를 둘러싼 논점들

이 에세이에 대한 반응에서 곱씹을 만한 논점 세 가지:

- **테스트의 재정의** — 에이전트는 테스트를 덜 중요하게 만드는 게 아니라 더 중요하게 만든다. 잡아야 할 버그가 "코드가 돌아가는가"에서 "조용히 잘못된 동작을 하는가"로 바뀌기 때문이다.
- **싸진 것과 안 싸진 것** — 에이전트는 코드 작성을 싸게 만들었지만 **아키텍처 결정을 싸게 만들지는 않았다**. 구현 비용이 떨어질수록 결정의 상대적 비중은 오히려 커진다.
- **트레이드오프 인지가 전부** — 지연시간 vs 비용, 일관성 vs 가용성 같은 트레이드오프를 문서화해 에이전트 프롬프트·컨텍스트에 내장하는 것이 실천법이다.

## 4. 온프레미스 LLM 플랫폼 팀을 위한 액션 아이템

| 영역 | 즉시 실행 액션 | 중장기 과제 |
|------|----------------|-------------|
| **데이터 관리** | 청크 전략/메타데이터 스키마 문서화, 골든셋에 반영 | 에이전트용 데이터 인프라 버전 관리 자동화 |
| **아키텍처** | 워크로드별(학습/추론/평가) 아키텍처 패턴 카탈로그화 | ADR 템플릿 표준화 + 에이전트 컨텍스트 주입 파이프라인 |
| **신뢰성** | 에이전트 파이프라인 전용 SLO/알림/런북 작성 | 그레이스풀 디그레이션 플레이북(폴백 모델/캐시) |
| **운영** | 평가 도구의 CI/CD 게이트화, 비용/지연시간 대시보드 연동 | A/B 테스트 프레임워크로 하이퍼파라미터 자동 튜닝 |
| **역량 개발** | 주니어 대상 **시스템 설계/트레이드오프 분석** 스터디 운영 | 온보딩 기준에 "트레이드오프 판단 역량" 반영 |

## 5. 마무리

Andrew Ng의 이 포스트는 **"코딩 에이전트 도입 = 엔지니어링 역량 하향"**이란 오해를 정면으로 반박한다. 오히려 **시스템적 사고·트레이드오프 판단·데이터/아키텍처 설계 역량**이 **더** 중요해지는 시점임을 명확히 한다.

> **"Understanding software fundamentals (in addition to AI) also helps you figure out what software can and cannot do. This makes them important context for how you use coding agents and shape the build."**
>
> — Andrew Ng, 포스트 마무리

평가 자동화, 골든셋 관리, 비용 최적화, 모니터링 대시보드 같은 운영 투자가 정확히 이 "에이전트 시대의 소프트웨어 기초"의 실천이다 — 에이전트가 코드를 쓰는 시대일수록 "왜 평가·모니터링·데이터 품질에 투자하는가"에 대한 답이 이 에세이에 있다.

다음 글에서는 **에이전트 코딩 시대의 테스트 전략(메타모픽/속성 기반/계약 테스트)**을 다뤄볼 예정이다.

## 참고

- [How have software engineering fundamentals changed with agentic coding? (The Batch #368 원문)](https://www.deeplearning.ai/the-batch/issue-368/){:target="_blank"}
- [Andrew Ng의 X 포스트](https://x.com/AndrewYNg/status/2093388974194872781){:target="_blank"}
- [RAGAS 평가 방법론 (관련글)]({{site.baseurl}}/dev/2026/08/29/ragas-evaluation-methodology.html)
- [Hermes 초안 자동 검수 파이프라인 (관련글)]({{site.baseurl}}/tools/2026/08/24/daily_draft_pipeline_humanize.html)

