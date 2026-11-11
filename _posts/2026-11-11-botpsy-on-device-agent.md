---
layout: post
comments: true
title: "BotPsy-0.8B: 노트북·폰에서 도는 0.8B 툴 콜링 에이전트 모델"
description: "Tether AI Research가 Qwen3.5-0.8B를 툴 콜링용으로 후속 학습한 BotPsy-0.8B를 Apache 2.0으로 공개했다. 1.2B 이하 비교 모델 중 툴 사용 벤치마크 최고 점수이며, GGUF Q8_0(0.81GB)으로 llama.cpp에서 바로 돌릴 수 있다. thinking 모드를 켜야 한다는 점과 약한 영역까지 정리한다."
img: botpsy_title.webp
date: 2026-11-11 20:00:00 +0900
last_modified_at: 2026-11-11 20:00:00 +0900
tags: [huggingface, llm-agent, llm-model, function-calling, on-device, gguf, llama.cpp, vllm, local-ai]
related: llm-agent
categories: [huggingface, llm-model]
source_url: https://huggingface.co/blog/qvac/botpsy
source_date: 2026-11-11
---

에이전트가 하는 일 중에는 프런티어 모델까지 필요 없는 것이 많다. 일정 하나 등록하고 알람을 맞추고 메시지를 보내는 정도라면 작은 모델이 기기 안에서 처리하는 편이 싸고 데이터도 밖으로 나가지 않는다. Tether AI Research(QVAC)가 11월 11일 Hugging Face 블로그에 공개한 **BotPsy-0.8B**가 이런 일을 겨냥한 모델이다. Qwen3.5-0.8B를 함수 호출과 에이전트 워크플로용으로 후속 학습했고 가중치는 Apache 2.0으로 올라와 있어 llama.cpp나 vLLM으로 오늘 바로 띄워 볼 수 있다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** BotPsy-0.8B는 Qwen3.5-0.8B 기반의 0.8B 툴 콜링 모델이다. 베이스 대비 BFCL v4 23.4→44.0, ACEBench 30.1→58.7, τ-bench 23.2→51.5로 툴 사용 점수가 대략 두 배가 됐고 여섯 개 툴·에이전트 벤치마크 중 다섯 개에서 2.5배 큰 Qwen3.5-2B-Think보다 높다. safetensors(`qvac/BotPsy-0.8B`)와 GGUF(`qvac/BotPsy-0.8B-GGUFs`, BF16·Q8_0·Q4_K_M)로 배포되며 권장 파일은 0.81GB짜리 Q8_0이다. 학습과 평가를 thinking 모드로 했는데 채팅 템플릿 기본값은 non-thinking이라 `enable_thinking`을 직접 켜야 한다. 영어 전용에 가깝고 웹 검색·메모리·딥리서치처럼 여러 단계를 거쳐 정보를 찾는 작업은 약하다.

## 무엇을 하는 모델인가

BotPsy는 도구를 불러 사용자의 간단한 작업을 처리하는 "개인 비서"형 모델이다. 원문이 꼽은 도메인은 웹 검색·리서치, 여행, 커머스·금융, 고객 응대, 기기·차량 제어, 파일·문서, 캘린더, 이메일·메시지다. 모델 카드는 구조를 "Qwen3.5 아키텍처(hybrid Gated DeltaNet + gated attention), 텍스트 전용으로 사용"이라고 적고 있다. 베이스인 Qwen3.5는 비전 입력을 받지만 BotPsy는 텍스트만 쓰도록 만든 모델이다.

제작진이 스스로 그은 선도 분명하다. 파라미터가 작아 **난이도 낮음~중간 작업**에 맞고 실수를 하므로 돈·개인정보·건강 같은 민감한 결정이나 안전이 걸린 자동화에는 쓰지 말라고 했다. 학습과 테스트는 주로 영어로 했고 권장 언어도 영어다.

## 벤치마크: 어디서 강하고 어디서 약한가

비교 대상은 베이스인 Qwen3.5-0.8B-Think, 2.5배 큰 Qwen3.5-2B-Think, 그리고 LFM2.5-1.2B-Think, MiniCPM5-1B-Think다. 모두 thinking 모드로 평가했다.

| 벤치마크 | BotPsy-0.8B | Qwen3.5-0.8B-Think (베이스) | Qwen3.5-2B-Think | LFM2.5-1.2B-Think | MiniCPM5-1B-Think |
|---|---|---|---|---|---|
| BFCL v4 | **44.0** | 23.4 | 37.7 | 28.1 | 22.8 |
| ACEBench | **58.7** | 30.1 | 55.4 | 39.4 | 24.5 |
| τ-bench | **51.5** | 23.2 | 46.5 | 6.5 | 28.8 |
| Claw-Eval | 46.5 | 39.3 | **48.9** | 31.6 | 31.0 |
| BrowseComp-Plus | **4.5** | 1.9 | 2.8 | 0.2 | 0.8 |
| ToolSandbox | **64.6** | 30.5 | 41.2 | 45.7 | 28.1 |

τ-bench는 τ²-bench(retail, telecom)와 τ³-bench(banking)의 평균이다. 여섯 개 중 Claw-Eval만 Qwen3.5-2B-Think가 앞선다(48.9 대 46.5).

<details class="evidence"><summary>원문 근거</summary><blockquote>"BotPsy roughly doubles the agentic capability of Qwen3.5-0.8B-Think on key benchmarks: BFCL v4 rises from 23.4 to 44.0, ACEBench from 30.1 to 58.7, and τ-bench from 23.2 to 51.5 (mean of the retail, telecom and banking subsets)."</blockquote></details>

에이전트를 직접 운영하는 입장에서 눈여겨볼 수치는 따로 있다.

- **맞는 도구가 없을 때 호출하지 않는다.** BFCL v4 hallucination 테스트에서 86.1로 비교 모델 중 가장 높았다. ACEBench의 special case(요청이 불완전하거나 파라미터가 틀렸거나 주어진 도구로 처리할 수 없는 경우)에서는 베이스 13.7에서 51.3으로 올랐지만 여기서는 Qwen3.5-2B-Think(58.0)가 여전히 앞선다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"On BFCL v4's hallucination tests, which check that the model declines when no appropriate tool exists, BotPsy scores 86.1, the highest of all models tested."</blockquote></details>
- **멀티턴에서 크게 좋아졌다.** BFCL v4 multi-turn은 7.5에서 46.1로 올라 Qwen3.5-2B-Think(23.8)의 두 배 가까이 된다. 긴 고객 대화에서 정책을 지키며 대신 처리해야 하는 τ²-bench telecom은 82.5(Qwen3.5-2B-Think 69.3)다.
- **추론 루프에 덜 빠진다.** IFBench 전체 시도에서 반복 추론 루프에 갇히는 비율이 22.9%에서 1.4%로, 32k 토큰 한도에 걸리는 비율이 28.3%에서 0.2%로 떨어졌다. 다섯 모델이 모두 맞힌 문제만 놓고 보면 정답까지 쓰는 추론 토큰도 베이스보다 GPQA Diamond에서 약 3배(3,690 대 11,563), MMLU-Pro에서 2.7배(1,912 대 5,105) 적다. 작은 모델을 기기에서 돌릴 때 지연을 좌우하는 부분이다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"Across all IFBench attempts, its rate of getting stuck in repetitive reasoning loops falls from 22.9% to 1.4%, and runs that hit the 32k-token limit fall from 28.3% to 0.2%."</blockquote></details>
- **안전성.** 여섯 가지 유해 범주로 자동 레드티밍을 했을 때 unsafe 응답 비율은 21%로 베이스(36%)보다 낮고 1.2B 이하 모델 중 가장 낮았다. 신뢰할 수 없는 도구 출력을 읽는 상황을 가정한 프롬프트 인젝션 공격에서는 46%에서 31%로 줄었다. 줄었다고는 해도 세 번에 한 번꼴이라 외부 웹 페이지나 메일 본문을 읽히는 구성이라면 별도 가드레일이 필요하다.
  <details class="evidence"><summary>원문 근거</summary><blockquote>"In automated red-teaming across six harm categories, BotPsy produces unsafe responses 21% of the time, compared with 36% for its base model."</blockquote></details>

약점도 원문에 정리돼 있다. BrowseComp-Plus 4.5는 다섯 모델 중 최고지만 절대값이 낮고 BFCL의 web search(14.0)와 memory(16.3)는 Qwen3.5-2B-Think(19.8, 26.1)보다 뒤진다. 앱 그룹별로 나눠 보면 단일 도구 호출 시나리오는 대부분 70~79점인데, 여러 단계에 걸친 정보 검색·리서치 시나리오는 15점에 그친다. τ³-bench banking은 모든 모델이 5점을 넘지 못했다(BotPsy 4.1). 일반 지식은 GPQA Diamond 38.1, MMLU-Pro 48.9로 1.2B 이하에서는 가장 높지만 Qwen3.5-2B-Think(59.9, 65.5)와 격차가 크다. 원문도 이 모델은 모르는 사실을 도구로 찾아오게 설계했지 지식 모델로 쓰라고 만든 게 아니라고 적었다. 다국어 지시 이행(Multi-IF)에서는 힌디어 24.5, 러시아어 57.6으로 영어·중국어·프랑스어 등의 72~80보다 많이 낮았다. 한국어는 평가 언어에 들어 있지 않다.

## 학습 방식: 4단계, 세 종류의 신호

Qwen3.5-0.8B에서 출발해 네 단계를 차례로 거쳤고 단계마다 직전 단계의 최고 체크포인트를 이어받았다.

1. **SFT**: 툴 사용 대화로 전체 파라미터를 미세조정했다. 시뮬레이션 환경의 멀티턴 에이전트 궤적, 단일·병렬·다단계 호출, 도구를 부르면 안 되는 사례가 들어갔다.
2. **MOPD(Multi-teacher On-Policy Distillation)**: 툴 사용, 일반 추론, 지시 이행에 각각 특화하도록 RL로 학습한 Qwen3.5-4B 교사 셋을 두고 프롬프트마다 한 교사에게 보내 토큰 단위 reverse KL을 학습 신호로 썼다. SFT에서 떨어진 지식과 지시 이행 능력을 되살리는 단계다.
3. **RL 1차**: DAPO로 규칙·검증기 보상만 써서 학습했다. 툴 콜링이 중심이고 지시 이행과 STEM 객관식이 함께 들어갔다. 보상은 먼저 추론·툴 호출 형식을 확인한 뒤, 맞다/틀리다 이진 보상 대신 도구 이름·인자 이름·인자 값을 정답과 비교해 부분 점수를 준다.
4. **RL 2차**: 수학, 코드, 지식 객관식, 지시 이행, 구조화 출력, 시뮬레이션 업무 환경의 다단계 툴 사용으로 범위를 넓혔다. 이 데이터에 멀티턴 대화는 없는데도 멀티턴 함수 호출 성능이 더 올랐다고 한다.

자체 소형 모델에 툴 콜링을 붙이려는 팀이라면 3단계의 부분 점수 보상 설계를 참고할 만하다.

## 직접 돌려 보기

### 파일 고르기

GGUF 저장소 `qvac/BotPsy-0.8B-GGUFs`에는 파일이 세 개 있다.

| 파일 | 크기 | 용도 |
|---|---|---|
| `botpsy-0.8b-bf16.gguf` | 1.52 GB | 양자화 없음, 원본 체크포인트와 같은 텐서 |
| `botpsy-0.8b-q8_0.gguf` | 0.81 GB | **권장**. BF16과 점수가 같은 수준, 크기는 절반 |
| `botpsy-0.8b-q4_k_m-imat.gguf` | 0.53 GB | 메모리가 빠듯할 때. 평균 약 1.5점 낮음 |

모델 카드의 llama.cpp 측정에서 Q8_0은 10개 벤치마크 평균 50.7로 BF16 GGUF(50.4)와 같은 수준이었다. Q4_K_M(imatrix)은 툴 사용 점수는 거의 유지했지만(BFCL v4 43.5, ACEBench 57.3) 지식 쪽에서 많이 깎였다(BF16 GGUF 대비 MMLU-Pro −3.5, GPQA Diamond −3.9). 같은 BF16 가중치라도 vLLM과 llama.cpp 사이에 τ-bench −4.0, Claw-Eval −3.1 차이가 났는데, 카드는 이를 양자화가 아니라 추론 엔진과 실행 간 편차 탓으로 본다. 벤치마크를 재현할 때는 엔진부터 맞춰야 한다는 뜻이다.


### 꼭 챙길 설정 두 가지

- **thinking 모드를 켠다.** 학습과 평가는 thinking 모드로 했는데 내장 채팅 템플릿의 기본값은 non-thinking이다. 켜지 않으면 위 점수를 기대하기 어렵다.
- **샘플링 값을 맞춘다.** 평가에 쓴 값은 `temperature=0.3`, `top_p=0.95`, `top_k=20`, `presence_penalty=1.5`다. 답 앞에 `<think>` 블록을 쓰기 때문에 생성 토큰 한도는 넉넉히 잡는다(모델 카드 예시는 `max_tokens` 8192).

<details class="evidence"><summary>원문 근거</summary><blockquote>"BotPsy was trained and evaluated in thinking mode. The embedded chat template defaults to non-thinking, so turn thinking on as shown below."</blockquote></details>

### llama.cpp

모델 카드의 예시를 그대로 옮긴다. OpenAI 호환 서버를 툴 콜링과 thinking을 켠 상태로 띄운다.

```bash
# 권장 파일 내려받기 (hf CLI가 없으면: pip install --user "huggingface_hub[cli,hf_xet]")
hf download qvac/BotPsy-0.8B-GGUFs botpsy-0.8b-q8_0.gguf --local-dir .

llama-server -m botpsy-0.8b-q8_0.gguf --jinja -c 32768 \
    --temp 0.3 --top-p 0.95 --top-k 20 --presence-penalty 1.5 \
    --chat-template-kwargs '{"enable_thinking": true}' --port 8080
```

```bash
curl -s http://localhost:8080/v1/chat/completions -H "Content-Type: application/json" -d '{
  "messages": [{"role": "user", "content": "Add the dentist on Friday at 3 pm."}],
  "tools": [{"type": "function", "function": {
    "name": "create_event",
    "description": "Add an event to the user'"'"'s calendar",
    "parameters": {"type": "object",
      "properties": {"title": {"type": "string"},
                     "day": {"type": "string", "description": "Day of the week"},
                     "time": {"type": "string", "description": "Start time, 24h HH:MM"}},
      "required": ["title", "day", "time"]}}}],
  "max_tokens": 8192
}'
```

응답의 `reasoning_content`에 추론이, `tool_calls`에 호출이 담겨 온다.

### vLLM

safetensors 저장소 `qvac/BotPsy-0.8B`를 쓴다. 텍스트 전용으로 띄우고 Qwen3 계열 파서를 지정한다.

```bash
vllm serve qvac/BotPsy-0.8B --language-model-only \
    --enable-auto-tool-choice --tool-call-parser qwen3_xml --reasoning-parser qwen3
```

```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:8000/v1", api_key="EMPTY")
resp = client.chat.completions.create(
    model="qvac/BotPsy-0.8B",
    messages=[{"role": "user", "content": "Add the dentist on Friday at 3 pm."}],
    tools=tools,  # 위 llama.cpp 예시와 같은 create_event 정의
    temperature=0.3, top_p=0.95, presence_penalty=1.5, max_tokens=8192,
    extra_body={"top_k": 20, "chat_template_kwargs": {"enable_thinking": True}},
)
print(resp.choices[0].message.tool_calls)
```

Transformers로 직접 부를 수도 있다. 모델 카드는 Qwen3.5를 지원하는 `transformers`(5.19.0에서 테스트)를 요구하고 `flash-linear-attention`과 `causal-conv1d`를 설치하지 않으면 Gated DeltaNet이 느린 참조 구현으로 돈다고 안내한다. `apply_chat_template`에 `tools=`와 `enable_thinking=True`를 넘기면 되고 출력은 Qwen3.5 XML 형식의 `<tool_call>` 블록으로 끝난다. 모바일·데스크톱 앱에 넣으려면 같은 GGUF를 QVAC SDK(`@qvac/sdk`, 0.23.0에서 테스트)로 불러 쓸 수 있다.

## 정리

모델 카드는 QVAC SDK로 폰·노트북·데스크톱에서 완전히 기기 안에서 돌릴 수 있다고 안내한다. 클라우드 모델이 맡던 일 중 "도구 하나 골라 인자 채우기"처럼 단순한 호출을 이 모델로 넘기고 성능을 재 보는 게 가장 현실적인 출발점이다. 한국어는 평가하지 않았으니 한국어 요청을 받는 구성이라면 자기 도구 스키마로 먼저 시험해 봐야 한다. 다단계 웹 리서치나 민감한 결정은 처음부터 큰 모델 쪽에 남겨 두는 게 맞다.

## 참고

- [BotPsy: On-Device Personal Assistant with a Sub-Billion-Parameter Agentic Language Model - Hugging Face Blog (2026-11-11)](https://huggingface.co/blog/qvac/botpsy){:target="_blank"}
- [qvac/BotPsy-0.8B - Hugging Face (safetensors 모델 카드)](https://huggingface.co/qvac/BotPsy-0.8B){:target="_blank"}
- [qvac/BotPsy-0.8B-GGUFs - Hugging Face (GGUF 파일·양자화별 벤치마크)](https://huggingface.co/qvac/BotPsy-0.8B-GGUFs){:target="_blank"}
- [QVAC SDK](https://qvac.tether.io/dev/sdk/){:target="_blank"}
