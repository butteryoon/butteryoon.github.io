---
layout: post
comments: true
title: "NVIDIA AVO, ARC-AGI-3 100% 달성 — 장기 자율 에이전트의 핵심은 모델이 아니라 시스템이다"
description: "NVIDIA 테크니컬 블로그 해설. AVO(Agentic Variation Operators) 에이전트 시스템이 GPU 커널 최적화와 ARC-AGI-3 인터랙티브 추론이라는 상이한 도메인에서 지속 메모리·슈퍼바이저·자율 실행 루프만으로 100% 성과를 낸 과정을 정리한다."
img: multi_agent_network.jpg
date: 2026-08-25 18:20:00 +0900
last_modified_at: 2026-08-25 18:20:00 +0900
tags: [nvidia, avo, arc-agi-3, agentic-ai, long-horizon-agents, gpu-kernel, claude-opus-5] # add tag
related: llm
categories: dev
---

에이전트 벤치마크 소식 중에서도 눈에 띄는 결과다. NVIDIA 테크니컬 블로그에 올라온 [AVO의 ARC-AGI-3 100% 달성 글](https://developer.nvidia.com/ko-kr/blog/nvidia-avo-reaches-100-on-arc-agi-3-demonstrating-a-frontier-level-general-purpose-architecture-for-long-horizon-autonomous-agents/){:target="_blank"}(2026-08-25, Jasmine Lee·Terry Chen·Yeyin Eva Zhu·Zhifan Ye·Jean-Francois Puget)을 해설한다. 핵심 주장은 간단하다. 장기 자율 에이전트의 성능을 가르는 것은 기반 모델의 단독 능력이 아니다. 지속 메모리와 슈퍼바이저, 자율 실행 루프가 함께 돌아가는 시스템 수준의 진행 메커니즘이 성능을 가른다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** NVIDIA의 AVO(Agentic Variation Operators)는 [ARC-AGI-3](https://arcprize.org/arc-agi/3/){:target="_blank"} 공개 세트 25개 환경의 전체 183개 레벨을 완료하며 **100.00 RHAE** 점수를 기록했다. 같은 아키텍처가 GPU 커널 최적화에서도 7일간 500개 이상의 최적화 방향을 탐색해 **cuDNN 대비 최대 +3.5%, FlashAttention-4 대비 최대 +10.5%**를 달성했다. 기반 모델은 Claude Opus 5(일부 서브셋에서 GPT-5.6 Sol 병행)다. 같은 모델을 단독으로 돌리면 약 30% 수준에 그친다. 시스템이 3배 이상 차이를 벌렸다.

## 1. AVO는 무엇인가

AVO는 지속 메모리(persistent memory), 슈퍼바이저(supervisor), 자체 실행 루프를 갖춘 범용 코딩 에이전트 시스템이다.

- **지속 메모리**: 이전 구현, 평가 결과, 컴파일러·프로파일러 출력, 추론 결과를 보존한다. 세션이 끊겨도 탐색을 처음부터 다시 시작하지 않고 저장된 상태에서 재개한다.
- **슈퍼바이저**: 정체나 반복적인 비생산 사이클을 감지하면 메인 에이전트를 대안 전략으로 전환시킨다.
- **자율 실행 루프**: 가설 수립 → 행동 → 관찰 → 상태 보존 → 수정 → 회복을 사람 개입 없이 장기간 반복한다.

눈에 걸리는 건 도메인 전환 방식이다. 도구와 평가 인터페이스만 교체하고 기반 에이전트 루프는 그대로 유지했다. GPU 커널 최적화는 컴파일러·프로파일러 피드백을, ARC-AGI-3는 환경 전이·행동 결과 피드백을 쓰지만 루프 자체는 동일하다.

## 2. 두 도메인의 성과

GPU 커널 최적화에서는 멀티헤드 어텐션 커널을 대상으로 DGX B200 시스템에서 7일간 연속 구동하며 500개 이상의 최적화 방향을 탐색했다. 그 결과 cuDNN 대비 최대 3.5%, FlashAttention-4 대비 최대 10.5% 빠른 커널이 나왔다.

ARC-AGI-3에서는 공개 세트 25개 환경의 183개 레벨을 전부 완료해 100.00 RHAE를 기록했다. 관찰 모달리티는 64×64 텍스트 그리드다. 이미지 토큰은 전혀 쓰지 않았다. 효율 면에서도 시각 기반 하네스인 VISTA(같은 Claude Opus 5 기반)가 7,542회의 환경 행동을 쓴 작업을 AVO는 6,624회, 약 12% 적은 행동으로 끝냈다.

가장 흥미로운 대조는 모델 단독 성능이다. Claude Opus 5를 high reasoning으로 단독 실행하면 같은 벤치마크에서 약 30% 수준에 그친다. 모델은 같은데 시스템을 씌우니 100%가 됐다. 저자들은 도메인 지식의 전이가 아니라 '지속적 자율 진행 메커니즘'의 전이에서 범용성의 실체를 봤다.

## 3. 직접 구현한다면

이 패턴을 자체 에이전트 스택에 이식하려면 무엇이 필요할까. 원문 기준으로 짚어본다.

- **지속 메모리**: 평가 결과·프로파일러 출력 같은 구조화 로그와 임베딩 검색을 결합한 하이브리드 스토리지가 필요하다. 단일 턴 검색-생성 패러다임의 RAG와 달리 상태를 쌓아가며 다단계 작업을 이어가는 구조다.
- **슈퍼바이저**: 별도 LLM 호출 또는 휴리스틱 룰 기반의 모니터링 루프로 구현한다. 정체 감지 시 대체 전략을 제안하는 메타 에이전트 패턴이다. 가드레일·모니터링 시스템과 결이 같다.
- **도구 인터페이스**: 커널 최적화라면 컴파일러(nvcc), 프로파일러(ncu), 테스트 러너, 벤치마크 스크립트를 래핑해야 한다.

제약도 분명하다. AVO 자체는 NVIDIA 연구 프로젝트로 오픈소스 공개 여부가 확인되지 않았다. 기반 모델도 상용 API다. 아키텍처가 모델에 구애받지 않는다고 하니 로컬 모델로 교체하는 실험은 가능하겠지만 성능 검증이 별도로 필요하다. ARC-AGI-3도 공개 세트만 접근 가능하고 경쟁 세트는 비공개다.

텍스트 전용 모달리티로 시각 벤치마크를 정복했다는 점, 그리고 성능의 대부분이 모델보다 메모리·슈퍼비전·루프에서 나왔다는 점은 에이전트 시스템을 설계하는 입장에서 곱씹을 만하다. [원문](https://developer.nvidia.com/ko-kr/blog/nvidia-avo-reaches-100-on-arc-agi-3-demonstrating-a-frontier-level-general-purpose-architecture-for-long-horizon-autonomous-agents/){:target="_blank"}에 VISTA, Tycho 등 관련 연구 링크가 함께 정리되어 있다.
