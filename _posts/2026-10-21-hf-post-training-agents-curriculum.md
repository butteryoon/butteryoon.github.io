---
layout: post
comments: true
title: "Hugging Face의 에이전트 포스트 트레이닝 커리큘럼: SFT부터 하네스 RL까지 5단계"
description: "Hugging Face가 공개한 에이전트 포스트 트레이닝 5단계 커리큘럼. 단계마다 무엇에서 배우는지, 어떤 TRL 트레이너를 쓰는지, 무엇을 준비해야 하는지와 강의·문서·예제 링크를 정리했다."
img: hf_post_training_agents_title.webp
date: 2026-10-21 19:57:00 +0900
last_modified_at: 2026-10-21 20:40:00 +0900
tags: [huggingface, trl, post-training, sft, distillation, grpo, openenv, reinforcement-learning, llm, llm-agent]
related: llm-agent
categories: [huggingface-blog, llm-agent]
source_url: https://huggingface.co/blog/huggingface/train-agents
source_date: 2026-10-20
---

Hugging Face의 ben burtenshaw, Sergio Paniego, merve가 에이전트 포스트 트레이닝 커리큘럼을 한 페이지로 묶어 공개했다. 포스트 트레이닝은 사전학습 다음에 오는 지도 미세조정, 증류, 강화학습 같은 단계를 가리킨다. 에이전트는 여러 턴에 걸쳐 도구를 부르고 결과를 읽은 뒤 다음 행동을 고르므로, 답 하나가 아니라 턴 전체를 놓고 학습시켜야 한다. 커리큘럼 순서는 Hugging Face 유튜브의 라이브스트림 시리즈 'Training Agents'를 따르고 코딩 에이전트 훈련을 공통 예제로 쓴다. 이 글에서는 다섯 단계 각각의 학습 신호, TRL 트레이너, 준비물과 함께 원문이 단계별로 걸어 둔 강의·문서·예제 링크를 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** 커리큘럼은 SFT → 증류 → RL → RL 환경 → 하네스와 확장의 다섯 단계로 짜여 있고, 각 단계가 TRL 트레이너(`SFTTrainer`, `DistillationTrainer`, `GRPOTrainer`, `AsyncGRPOTrainer`) 하나씩과 짝을 이룬다. 어느 단계에서 시작할지는 지금 손에 든 것으로 정하면 된다. 에이전트 궤적 데이터셋이 있으면 SFT, 교사 모델이 있으면 증류, 답을 자동으로 채점할 수 있으면 RL, 실제로 쓰는 에이전트 하네스가 있으면 5단계다.

## 단계별 한눈에 보기

원문 맨 앞의 요약표를 옮겼다.

| 단계 | 모델이 배우는 대상 | TRL 트레이너 | 준비물 |
|---|---|---|---|
| 1. SFT | 더 강한 모델이 만든 에이전트 궤적 | `SFTTrainer` | 에이전트 궤적 데이터셋 |
| 2. 증류 | 교사의 토큰 또는 확률 | `DistillationTrainer` | 프롬프트, 교사 모델 |
| 3. RL | 답안마다 매긴 점수 | `GRPOTrainer` | 프롬프트, 보상 함수 |
| 4. RL 환경 | 환경 안에서 행동한 결과 | `GRPOTrainer` | 태스크, 환경(OpenEnv) |
| 5. 하네스와 확장 | 실제 에이전트의 전체 실행 | `AsyncGRPOTrainer` | 태스크, 환경, 에이전트 하네스 |

강의는 [Post-training Agents 재생목록](https://www.youtube.com/playlist?list=PLo2EIpI_JMQvQZm-kVlz4wY1vWF0LBcf5){:target="_blank"}에 네 번의 세션과 관련 워크숍 두 편이 들어 있다.

## 0단계: 시작 전에

에이전트, 하네스, 스캐폴드 같은 용어부터 맞추고 전체 흐름을 한 번 훑는다.

- 용어 정리: [Harness, Scaffold, and the AI Agent Terms Worth Getting Right](https://huggingface.co/blog/agent-glossary){:target="_blank"}
- 전체 개요 슬라이드: [Train the agent, not just the model](https://huggingface.co/spaces/sergiopaniego/train-the-agent-slides){:target="_blank"} (전 단계를 다룬 20장)
- 미세조정을 처음 해 본다면: [LLM Course](https://huggingface.co/learn/llm-course){:target="_blank"}

## 1단계: 지도 미세조정(SFT)

에이전트 궤적을 그대로 따라 하도록 학습시킨다. 궤적은 에이전트가 태스크를 푸는 과정을 도구 호출과 그 결과까지 빠짐없이 기록한 것으로, 대개 더 강한 모델이 만든다. 비용이 싸고 학습도 안정적이다. 다만 모델이 보는 것은 정답 궤적뿐이고 자기가 시도한 결과는 보지 못해서 문제를 풀다 실수하면 거기서 빠져나오는 법을 모른다. 원문은 이 빈틈을 모델 자신의 출력으로 학습하는 온폴리시 증류와 RL이 메운다고 설명한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The model learns to copy agent traces, recordings of an agent solving a task with every tool call and its result, usually produced by a stronger model. It's cheap and stable."</blockquote></details>

- 강의: [Training Agents 1: fine-tune a coding agent](https://www.youtube.com/watch?v=rNgUoH7Wbv8){:target="_blank"} ([코드](https://huggingface.co/burtenshaw/gemma4-pi-mono-youtube-livestream-1-scripts){:target="_blank"})
- 읽을거리: [Training Agents: Class 1, SFT from scratch](https://x.com/ben_burtenshaw/status/2067615361428545566){:target="_blank"}
- 문서: [SFT Trainer](https://huggingface.co/docs/trl/sft_trainer){:target="_blank"}

## 2단계: 증류

교사 모델이 학생을 가르친다. 교사가 직접 쓴 데이터로 가르치기도 하고 학생이 생성한 토큰에 교사가 매긴 확률로 가르치기도 한다. 교사는 보통 더 큰 모델이지만 MOPD처럼 전문 모델 여럿을 학생 하나로 합치기도 하고, 자기 증류라면 컨텍스트를 더 얹은 같은 모델이 교사가 된다. 토큰마다 값이 하나씩 붙는 촘촘한 신호라서 RL보다 적은 샘플로 학습된다. 대신 학생은 교사가 아는 범위를 넘지 못하고 온폴리시 증류는 학습 스텝마다 교사를 돌려야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The signal is dense, one value per token, so training needs fewer samples than RL. On the other hand, the student is limited by what the teacher knows, and on-policy distillation has to run the teacher at every training step."</blockquote></details>

- 강의: [Training Agents 2: distillation](https://www.youtube.com/watch?v=GZT_oNQnLV4){:target="_blank"} ([컬렉션](https://huggingface.co/collections/sergiopaniego/training-agents-class-2-distillation-6a47a62b9f09867dccf61e1e){:target="_blank"})
- 읽을거리: [A brief history of distillation in AI](https://huggingface.co/blog/sergiopaniego/brief-history-of-distillation-in-ai){:target="_blank"}, [Distillation in 2026 (so far)](https://huggingface.co/blog/sergiopaniego/distillation-2026){:target="_blank"}
- 문서: [Distillation Trainer](https://huggingface.co/docs/trl/distillation_trainer){:target="_blank"}

## 3단계: 강화학습(RL)

모델이 자기 출력에 매겨진 점수로 배운다. 답을 쓰면 보상 함수가 채점하고 학습은 점수가 높은 답이 더 자주 나오게 만든다. 교사는 필요 없고 답을 검사할 방법만 있으면 된다. 이 단계에서 쓰는 알고리즘은 GRPO(Group Relative Policy Optimization)로, 프롬프트 하나에 답을 여러 개 뽑아 모두 채점한 뒤 그룹 평균보다 높은 답의 확률을 올린다. 그룹 안에서 답끼리 비교하니 어느 시도가 나았는지 판단하는 모델을 따로 학습시킬 필요가 없다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The algorithm used in this step is GRPO (Group Relative Policy Optimization). For each prompt it samples several answers, scores them all, and raises the probability of the ones that scored above the group average."</blockquote></details>

신호는 증류보다 약하다. 토큰마다가 아니라 답 전체에 숫자 하나가 붙으므로 모델은 여러 번 시도해야 배운다. 평범한 보상 함수를 쓰면 답 하나가 곧 한 턴이고 채점도 한 번뿐이다. 여러 번 행동하고 행동마다 결과를 보게 하려면 다음 단계의 환경이 필요하다.

- 강의: [Training Agents 3: reinforcement learning](https://www.youtube.com/watch?v=ztdTed5egrM){:target="_blank"} ([컬렉션](https://huggingface.co/collections/sergiopaniego/training-agents-class-3-reinforcement-learning-6ad6aa71050342629dd0e3ec){:target="_blank"})
- 읽을거리: [How frontier models train on outcomes in 2026](https://huggingface.co/blog/sergiopaniego/agentic-rl-2026){:target="_blank"}
- 예제 프로젝트: [Fine-tuning a 350M model for structured outputs in 100 GRPO steps](https://huggingface.co/blog/grpo-with-trl-ifstruct){:target="_blank"}
- 문서: [GRPO Trainer](https://huggingface.co/docs/trl/grpo_trainer){:target="_blank"}

## 4단계: RL 환경

보상이 환경 안에서 한 행동에서 나온다. 환경은 상태를 유지하고 도구를 노출하며 여러 턴에 걸쳐 응답한다. 도구를 실행하고 무엇이 바뀌었는지 추적하고 최종 결과를 확인하는 것도 환경 몫인데, 코딩 태스크라면 테스트를 돌려 보는 식이다. 에피소드 하나가 답 하나보다 훨씬 길어서 롤아웃 생성이 학습에서 가장 느린 구간이 된다. 트레이너는 3단계와 같은 `GRPOTrainer`를 쓰고 환경은 OpenEnv로 붙인다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The environment runs the tools, keeps track of what changed and checks the final result, for example by running the tests of a coding task. Each episode is longer than a single answer, so generating rollouts becomes the slowest part of training."</blockquote></details>

- 강의: [Training Agents 4: from reward functions to environments](https://www.youtube.com/watch?v=nJV3yUuz6DU){:target="_blank"} ([컬렉션](https://huggingface.co/collections/sergiopaniego/training-agents-class-4-rl-environments-6aa27543918888a0f2b5312c){:target="_blank"}), [RL for Agents Workshop](https://www.youtube.com/watch?v=cixmqTsi2A4){:target="_blank"}
- 읽을거리: [Why RL environments work now (and could not in 2016)](https://huggingface.co/blog/sergiopaniego/rl-environments-story){:target="_blank"}
- 예제 프로젝트: [Paint watercolours with code](https://huggingface.co/blog/train-to-paint-with-code){:target="_blank"}, [Autonomous driving with CARLA](https://huggingface.co/blog/sergiopaniego/bringing-carla-to-openenv-trl){:target="_blank"}
- 문서: [TRL + OpenEnv](https://huggingface.co/docs/trl/openenv){:target="_blank"}, [OpenEnv docs](https://huggingface.co/docs/openenv){:target="_blank"}

## 5단계: 하네스와 확장

롤아웃 하나하나가 원격 샌드박스 안에서 기존 에이전트 하네스를 통째로 돌린 실행이다. 에이전트가 실제로 쓰는 하네스를 그대로 거쳐 학습하므로, 모델은 나중에 돌아갈 바로 그 환경에서 배운다. 생성과 학습은 서로 다른 GPU에서 돌아서 에피소드가 길어져도 트레이너가 놀지 않는다. 트레이너는 갱신한 가중치를 vLLM으로 다시 보낸다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Each rollout is a full agent run through an existing agent harness inside a remote sandbox. Training through the same harness the agent uses in practice means the model learns in the setting where it will run."</blockquote></details>

- 슬라이드: [Training a coding agent through a harness you did not write](https://huggingface.co/spaces/FineEnvs/multi-harness-rl-slides){:target="_blank"}
- 읽을거리: [The ultimate guide to multi-harness RL](https://huggingface.co/spaces/FineEnvs/multi-harness-rl){:target="_blank"}, [One sandbox per rollout](https://huggingface.co/blog/sergiopaniego/rl-environments-2026){:target="_blank"}, [Keep the Tokens Flowing: lessons from 16 open-source RL libraries](https://huggingface.co/blog/async-rl-training-landscape){:target="_blank"}
- 문서: [Async GRPO Trainer](https://huggingface.co/docs/trl/async_grpo_trainer){:target="_blank"}, [OpenEnv harnesses](https://huggingface.co/docs/openenv/tutorials/harnesses){:target="_blank"}

## 어디서 시작할까

원문은 순서대로 따라가는 커리큘럼이지만 이미 에이전트를 운영 중인 팀이라면 요약표의 '준비물' 열을 거꾸로 읽는 편이 빠르다. 지금 가진 것과 각 단계의 대가를 맞대 보면 시작점이 나온다.

- **강한 모델로 돌린 에이전트 로그가 쌓여 있다**: SFT로 시작한다. 싸고 안정적이지만 정답 궤적만 보고 배우므로 실수에서 복구하는 법은 익히지 못한다.
- **더 큰 모델을 교사로 쓸 수 있다**: 증류를 쓴다. 적은 샘플로 학습되지만 교사 수준이 상한이고 온폴리시 방식은 스텝마다 교사 추론 비용이 든다.
- **답을 자동으로 채점할 수 있다**(테스트 통과, 출력 형식 검사 등): GRPO로 RL을 건다. 교사가 필요 없는 대신 샘플이 많이 든다. 원문의 예제처럼 350M 모델에 구조화 출력을 100스텝 GRPO로 가르치는 작은 실험부터 해 볼 만하다.
- **도구를 여러 번 부르는 태스크다**: 단일 턴 보상 함수로는 행동마다 결과를 볼 수 없으니 OpenEnv 같은 환경이 필요하다. 이때부터는 롤아웃 생성이 병목이 된다는 점을 계산에 넣는다.
- **배포에 쓰는 하네스가 따로 있다**: 5단계 방식으로 같은 하네스 안에서 롤아웃을 돌리면 훈련 환경과 실행 환경이 일치한다. 생성용 GPU와 학습용 GPU를 나누는 비동기 구성(`AsyncGRPOTrainer` + vLLM)이 전제다.

## 참고

- 원문: [A curriculum for post-training agents](https://huggingface.co/blog/huggingface/train-agents){:target="_blank"} (2026-10-20)
- TRL 문서: [https://huggingface.co/docs/trl](https://huggingface.co/docs/trl){:target="_blank"}
- OpenEnv 문서: [https://huggingface.co/docs/openenv](https://huggingface.co/docs/openenv){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/-0xMiYQmk8g){:target="_blank"}

