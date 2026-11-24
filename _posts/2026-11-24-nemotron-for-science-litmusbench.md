---
layout: post
comments: true
title: "Nemotron for Science 정리 — LitmusBench v0.2 데이터셋과 NeMo Gym 평가 절차"
description: "NVIDIA가 Nemotron 3.5 Super VL의 과학 분야 사후 학습 사례와 화학 추론 오픈 데이터셋 LitmusBench v0.2를 묶어 Nemotron for Science로 내놨다. 발표 글이 말하지 않는 데이터셋 구성·라이선스·레코드 형식과 NeMo Gym의 평가 명령을 데이터셋 카드와 저장소 README 기준으로 정리했다."
img: nemotron-for-science-litmusbench_title.webp
date: 2026-11-24 19:30:00 +0900
last_modified_at: 2026-11-24 19:30:00 +0900
tags: [nvidia, nemotron, nemotron-for-science, litmusbench, nemo-gym, rlvr, post-training, chemistry, dgx-spark, llm-science, llm-eval, llm]
related: llm-science
categories: dev
source_url: https://blogs.nvidia.co.kr/blog/nvidia-introduces-nemotron-for-science-to-accelerate-discoveries-with-agents/
source_date: 2026-11-23
---
NVIDIA 한국 블로그에 11월 23일 [「AI 에이전트로 과학적 발견 가속화하는 NVIDIA Nemotron for Science」](https://blogs.nvidia.co.kr/blog/nvidia-introduces-nemotron-for-science-to-accelerate-discoveries-with-agents/){:target="_blank"}가 올라왔다. 글 자체는 발표문이라 파트너 사례와 인용이 대부분이다. 다만 글이 가리키는 모델과 데이터셋은 지금 Hugging Face에서 받을 수 있고 데이터셋을 채점하는 환경은 NeMo Gym 저장소에 공개돼 있다. 그래서 발표문은 짧게 요약하고 직접 써 볼 사람에게 필요한 내용은 [데이터셋 카드](https://huggingface.co/datasets/nvidia/Nemotron-RL-litmus-bench-v0.2){:target="_blank"}와 [NeMo Gym 저장소](https://github.com/NVIDIA-NeMo/Gym){:target="_blank"}의 README에서 가져와 출처를 구분해 적는다. (이 글의 초안은 Orca Bots 가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Nemotron for Science는 새 모델이 아니다. 10월에 공개된 Nemotron 3.5 Super VL에 과학 분야 사후 학습 사례, 화학 추론 데이터셋 LitmusBench v0.2, 사후 학습 레시피를 묶은 이름이다. 지금 받을 수 있는 것은 모델 가중치(OpenMDW 1.1)와 데이터셋(학습 47,118문항, 평가 2,056문항, 64 MB)이고 채점 환경은 NeMo Gym(Apache 2.0)에 `litmus_agent`라는 이름으로 들어 있다. 데이터셋은 문항마다 정답 추출 정규식과 정답을 갖고 있어 보상이 맞으면 1.0, 틀리면 0.0으로 떨어지는 RLVR 형식이다. 발표문이 내세운 파트너 수치(양자점 제조 정확도 61%, GROMACS 설정 복구 81% 등)는 재현 절차가 글에 없으므로 참고치로만 본다.

## 1. 발표 내용 요약

발표문에 따르면 NVIDIA가 Nemotron for Science를 선보인 날은 11월 16일(현지 시간)이고 한국 블로그 게시일은 11월 23일이다. 구성은 세 가지다.

- **Nemotron 3.5 Super VL의 과학 활용 사례**: 모델 자체는 이전에 다룬 [Nemotron 3.5 Super VL·Bolt 공개 글]({{site.baseurl}}/nvidia/llm-model/2026/10/23/nemotron-3-5-super-vl-bolt.html)의 그 모델이다.
- **LitmusBench v0.2**: 화학 분야 추론을 학습·평가하기 위한 오픈 데이터셋.
- **사후 학습 레시피**: 도메인 데이터로 모델을 조정하는 절차. 발표문은 레시피가 제공된다고만 쓰고 레시피로 가는 링크는 달지 않았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Nemotron for Science의 일부인 LitmusBench v0.2는 화학 분야 추론에 대한 언어 모델을 학습하고 평가하기 위한 약 5만 개의 질문으로 구성된 새로운 오픈 데이터셋입니다."</blockquote></details>

활용 기관으로는 생명과학 기업 Lila Sciences, Terray Therapeutics, Therna와 아르곤 국립연구소, 율리히 슈퍼컴퓨팅 센터, KTH, RIKEN이 언급된다.

## 2. 지금 받을 수 있는 것

발표문의 「이용 가능 여부」 절이 가리키는 것은 두 가지이고 나머지 행은 각 카드와 저장소에서 확인한 내용이다.

| 항목 | 위치 | 라이선스 | 출처 |
|---|---|---|---|
| Nemotron 3.5 Super VL (BF16) | `nvidia/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-BF16` | OpenMDW-1.1 | 발표문, 모델 카드 |
| 같은 모델의 NVFP4 양자화판 | `nvidia/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-NVFP4` | OpenMDW-1.1 | 모델 카드 |
| LitmusBench v0.2 | `nvidia/Nemotron-RL-litmus-bench-v0.2` | CC BY SA 4.0 + OpenMDW 1.1 | 발표문, 데이터셋 카드 |
| 채점 환경 `litmus_agent`, 벤치마크 설정 `litmus-bench` | `NVIDIA-NeMo/Gym` | Apache 2.0 | 저장소 README |

로컬 실행에 대해 발표문은 128GB 메모리의 DGX Spark를 지목한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Nemotron 3.5 Super VL 모델은 128GB 메모리를 탑재한 NVIDIA DGX Spark 시스템에서 로컬로 실행할 수 있도록 최적화됐습니다."</blockquote></details>

BF16 모델 카드의 빠른 시작 예시는 4× H100 기준이고 DGX Spark나 데이터센터 GPU에서 효율적으로 추론하려면 NVFP4 카드를 보라고 안내한다. NVFP4 카드의 배포 항목은 `1× DGX Spark, 1x NVIDIA RTX PRO 6000, 1× H200, 2× H100, or 1× GB200`이다. 로컬 런타임으로는 llama.cpp(GGUF)와 Unsloth가 카드에 링크돼 있다.

## 3. LitmusBench v0.2 구성

여기부터는 데이터셋 카드 기준이다. 발표문의 "약 5만 개"는 카드에서 학습 47,118문항과 평가 2,056문항으로 나뉜다.

<details class="evidence"><summary>원문 근거 (데이터셋 카드)</summary><blockquote>"It includes 47,118 training questions and 2,056 test questions, each in short-answer format"</blockquote></details>

- **원천 데이터**: ChEMBL, ExpansionRx, Biogen 데이터셋에서 만들었고 일부 문항은 RDKit 기술자(descriptor)를 쓴다.
- **범위**: 카드 표기로 126개 속성과 12개 분석(assay) 엔드포인트. 모든 문항이 단답형이다.
- **크기**: 64 MB. 언어는 영어.
- **용도**: 카드는 RL 학습용이라고 적고 NeMo Gym과 함께 LLM 사후 학습에 쓰라고 명시한다. 호환 프레임워크로 NVIDIA NeMo RL, TRL, Unsloth, veRL을 든다.
- **이전 판**: Litmus Bench v0.1이 따로 있다.

라이선스는 문항 출처에 따라 둘로 갈린다. 상업적 이용 가능 여부를 따질 때 한 덩어리로 보면 안 된다.

<details class="evidence"><summary>원문 근거 (데이터셋 카드)</summary><blockquote>"Governing terms: questions containing data from the ChEMBL database are licensed under CC BY SA 4.0. All other questions are licensed under OpenMDW 1.1."</blockquote></details>

카드는 평가 분할을 "test"라고 부르지만 저장소의 실제 파일 이름은 `train.jsonl`과 `validation.jsonl`이다(Hugging Face API의 파일 목록으로 확인). 데이터셋은 접근 승인 없이 받을 수 있는 상태(`"gated":false`)였다.

### 레코드 형식

`train.jsonl`의 첫 행을 그대로 옮긴다. 줄바꿈만 넣었다.

```json
{"dataset": "litmus_bench_v0.2",
 "agent_ref": {"name": "litmus_agent_agent"},
 "responses_create_params": {"input": [{"role": "user", "content": "Report the QED of this molecule.\n\nCOc1cc2c(cc1O)CN(C(=O)C(C)C)CC2\n\nProvide your final answer as '<Answer: floating_point_number>', where floating_point_number is the answer. The final result should be a single floating point number."}]},
 "expected_answer": 0.871091,
 "answer_type": "float",
 "output_regex": "<?Answer:\\s*\\*{0,2}([-+]?\\d*\\.?\\d+(?:[eE][-+]?\\d+)?)",
 "property": "qed",
 "method": "direct",
 "question_license": "CC-BY-SA-4.0",
 "metadata": {"uuid": "75b5d882057e192ad8c83c1b752fcbc3", "sources": [{"dataset": "chembl", "dataset_version": "36", "row_id": "CHEMBL488878"}]},
 "question_family": "tier2_rdkit"}
```

눈여겨볼 점은 세 가지다.

- 입력은 Responses API 형식(`responses_create_params.input`)의 메시지 배열이다.
- 문항마다 답 형식 지시가 다르다. 처음 세 행만 봐도 `<Answer: ...>`, `**Final answer is: ...**`, 이중 괄호로 요구가 바뀌고 그에 맞는 `output_regex`가 행에 들어 있다. 형식 준수 자체가 채점 대상이 된다.
- `question_license`가 행 단위로 붙어 있어 라이선스별로 걸러 쓸 수 있다.

## 4. NeMo Gym으로 평가하는 절차

저장소 README 두 개(`benchmarks/litmus-bench/README.md`, `resources_servers/litmus_agent/README.md`)를 기준으로 적는다. 아래 명령은 README에서 그대로 옮긴 것이고 이 글을 쓰면서 실행해 보지는 않았다.

채점기 `litmus_agent`는 도메인에 묶이지 않은 정답 검증기다. 모델 응답에서 값을 뽑아 `expected_answer`와 비교한다.

<details class="evidence"><summary>원문 근거 (litmus_agent README)</summary><blockquote>"Reward is `1.0` on match, `0.0` otherwise (including when no value can be extracted)."</blockquote></details>

`answer_type`은 `float`, `bool`, `string` 세 가지이고 기본 비교 규칙은 각각 `isclose`, `bool_eq`, `string_eq`다. 행에 `match` 필드를 두면 `abs_window`, `rel_window` 같은 허용 오차 규칙으로 바꿀 수 있다.

평가 데이터 준비와 실행은 다음과 같다.

```bash
gym eval prepare --benchmark litmus-bench
```

```bash
gym eval run \
  --benchmark litmus-bench \
  --model-type vllm_model \
  --split benchmark \
  --output results/litmus-bench.jsonl
```

준비 단계는 평가 파일을 내려받아 SHA-256, 행 수, 스키마, 문항군별 개수를 검증한다. 평가 2,056문항은 네 부분집합으로 나뉜다.

| 부분집합 | 문항군 | 문항 수 | 설정 파일 |
|---|---|---:|---|
| Tier 1/2 | `tier1_rdkit`, `tier2_rdkit` | 1,000 | `config_tier12.yaml` |
| ADME direct | `tier5_admet_direct` | 598 | `config_adme_direct.yaml` |
| ADME analogue | `tier5_admet_analog` | 358 | `config_adme_analogue.yaml` |
| ADME comparison | `tier5_admet_comparison` | 100 | `config_adme_comparison.yaml` |

실행 비용을 가늠할 때 필요한 설정값도 README에 있다.

<details class="evidence"><summary>원문 근거 (litmus-bench README)</summary><blockquote>"All benchmark configs use five rollouts per question."</blockquote></details>

- 문항당 롤아웃이 5회라 전체 벤치마크는 문항 수의 다섯 배만큼 호출한다. README는 이 분할에 `--num-repeats 5`를 또 주지 말라고 경고한다(곱해진다).
- 출력 토큰 상한은 통합·Tier 1/2 설정이 131,072, ADME 설정이 8,192이고 `--max-output-tokens`로 바꾼다.
- 전체를 돌리기 전에 저장소에 들어 있는 10문항 예제(`data/examples/`)로 먼저 확인할 수 있다.

```bash
gym eval run --no-serve \
  --agent litmus_agent_agent \
  --input benchmarks/litmus-bench/data/examples/litmus-tier12_example.jsonl \
  --output results/litmus-tier12-example.jsonl \
  --num-repeats 1 \
  --concurrency 8 \
  --max-output-tokens 8192
```

이 명령은 다른 터미널에서 `gym env start --resources-server litmus_agent ...`로 채점 서버와 모델 연결을 먼저 띄운 뒤 실행한다. README의 예시는 NVIDIA 추론 게이트웨이를 `--model-url`로 지정한다. 자체 vLLM 엔드포인트를 붙일 때의 인자 조합은 README에 따로 예시가 없어 여기 적지 않는다.

채점기 README에는 수치 답 추출의 함정도 적혀 있다. 줄 끝까지 잡는 열린 형식에서는 캡처된 텍스트의 마지막 숫자가 답으로 채택돼 `Final Answer: 42 (my 7th attempt)`가 `7`로 채점된다. 수치 답에는 구분자가 있는 형식을 쓰라는 권고다. 자체 평가 세트를 같은 채점기로 만들 때 그대로 적용되는 주의점이다.

## 5. 발표문이 제시한 사후 학습 결과

발표문의 수치는 다음과 같다. 모두 파트너와 NVIDIA가 밝힌 값이고 평가 세트·기준 모델 설정·재현 절차는 글에 없다.

- **Terray Therapeutics**: 과학 다이어그램에서 반응물·생성물·반응 조건을 추출하도록 사후 학습. RxnScribe 테스트에서 주석 처리된 반응의 약 76%를 복원.
- **Lila Sciences**: 실험실 피드백으로 사후 학습해 양자점 제조에 대한 정확도를 61%로 높임. 비교 대상으로 GPT-6 Astra 54%를 제시.
- **Therna**: mRNA 치료제 설계용 사후 학습으로 단백질 발현 범위 예측 정확도가 기존 최첨단 모델 대비 약 2배. 비교 평가에 참여한 과학자의 96%가 사후 학습 모델의 설명을 선호.
- **KTH**: 손상된 GROMACS 분자 동역학 설정을 복구하는 에이전트. DGX Spark 한 대에서 실행.

<details class="evidence"><summary>원문 근거</summary><blockquote>"실험실 피드백을 활용한 사후 학습을 통해 스크린과 태양광 패널에 사용되는 나노 크기의 결정체인 양자점(quantum dots) 제조에 대한 모델의 정확도를 61%로 약 2배 높였죠. 이는 GPT-6 Astra의 정확도 54%를 넘어서는 동시에, 10~20배 낮은 추론 비용으로 달성한 결과죠."</blockquote><blockquote>"단일 NVIDIA DGX Spark™ 시스템에서 하룻밤 동안 실행한 결과, 해당 에이전트는 추론 기능을 활성화한 상태에서 테스트 사례의 81%를 수정했으며, 모든 수정 사항은 GROMACS를 통해 독립적으로 검증됐습니다."</blockquote></details>

KTH 사례는 수치보다 구조가 참고할 만하다. 에이전트가 고친 결과를 모델이 아니라 GROMACS가 검증했다. LitmusBench의 정규식 채점과 같은 방향으로, 검증 가능한 보상이 있는 작업에 사후 학습과 에이전트를 붙이는 방식이다. 같은 계열의 RL 사후 학습 사례는 [DaisyRose-27B 글]({{site.baseurl}}/dev/2026/11/21/daisyrose-27b-rl-post-training.html)에서도 다뤘다.

## 6. 정리

- 발표문만 보면 파트너 사례 모음이지만 실제로 손에 잡히는 것은 데이터셋과 채점 환경이다. 화학을 다루지 않더라도 "문항마다 추출 정규식과 정답을 싣고 0/1 보상으로 채점한다"는 레코드 설계는 사내 도메인 평가 세트를 만들 때 그대로 참고할 수 있다.
- 사후 학습 레시피는 발표문에 링크가 없다. 데이터셋 카드가 NeMo Gym을 가리키고 저장소에 평가 절차가 있다는 것까지가 확인된 범위다. 발표문의 파트너 결과를 재현하는 학습 절차는 이 글에서 확인하지 못했다.
- 발표문이 로컬 실행 장비로 지목한 것은 128GB DGX Spark다. 벤치마크 자체는 Nemotron 전용이 아니다. README 예시가 `vllm_model`, `openai_model` 유형과 게이트웨이의 다른 모델을 대상으로 돌리고 있다.
- 데이터셋 라이선스는 문항별로 다르다. ChEMBL 유래 문항은 CC BY SA 4.0이라 동일조건변경허락 조건이 붙는다.

## 참고

- [AI 에이전트로 과학적 발견 가속화하는 NVIDIA Nemotron for Science — NVIDIA 한국 블로그](https://blogs.nvidia.co.kr/blog/nvidia-introduces-nemotron-for-science-to-accelerate-discoveries-with-agents/){:target="_blank"}
- [nvidia/Nemotron-RL-litmus-bench-v0.2 — Hugging Face 데이터셋 카드](https://huggingface.co/datasets/nvidia/Nemotron-RL-litmus-bench-v0.2){:target="_blank"}
- [nvidia/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-BF16 — 모델 카드](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-BF16){:target="_blank"}
- [nvidia/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-NVFP4 — 모델 카드](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Super-VL-120B-A12B-NVFP4){:target="_blank"}
- [NVIDIA-NeMo/Gym — GitHub 저장소](https://github.com/NVIDIA-NeMo/Gym){:target="_blank"}
- [Litmus-Bench 벤치마크 README](https://github.com/NVIDIA-NeMo/Gym/blob/main/benchmarks/litmus-bench/README.md){:target="_blank"}
- [litmus_agent 채점 서버 README](https://github.com/NVIDIA-NeMo/Gym/blob/main/resources_servers/litmus_agent/README.md){:target="_blank"}
