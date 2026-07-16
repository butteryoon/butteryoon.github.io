---
layout: post
comments: true
title: "OpenRouter 무료 모델을 API로 사용하기"
description: "OpenRouter의 프리(:free) 모델을 API로 호출하는 방법과 사용 가능한 모델 목록"
img: llm_api_title.jpg
date: 2026-07-15 01:00:00 +0900
last_modified_at: 2026-07-16 02:40:00 +0900
tags: [llm, openrouter, api, free] # add tag
related: llm
categories: tools
---
온프레미스 서비스로 오픈웨이트 모델을 사용하려 할 때 GPU 인프라가 충분하지 않으면 여러가지 모델을 하기 어렵다. 이렇게 LLM 모델별 API로 테스트해보고 싶은데 비용이 부담될 때 [OpenRouter](https://openrouter.ai)의 무료 모델을 쓸 수 있다. 여러 프로바이더의 모델을 하나의 OpenAI 호환 API로 호출할 수 있고, 모델 ID에 `:free`가 붙은 모델은 과금 없이 사용할 수 있다.

<!--more-->

## OpenRouter란

OpenRouter는 OpenAI, Anthropic, Google, Meta 등 여러 프로바이더의 모델을 **하나의 API 엔드포인트**로 중계해주는 서비스다. OpenAI SDK와 호환되기 때문에 base URL과 API 키만 바꾸면 기존 코드를 그대로 쓸 수 있다.

## API 키 발급

1. [openrouter.ai](https://openrouter.ai) 가입
2. 우측 상단 프로필 &gt; **Keys** 메뉴에서 **Create Key**
3. 발급된 `sk-or-v1-...` 키를 환경변수로 저장

```powershell
# PowerShell
[System.Environment]::SetEnvironmentVariable("OPENROUTER_API_KEY", "<발급받은 키>", "User")
```

```bash
# bash
export OPENROUTER_API_KEY="<발급받은 키>"
```

## 무료 모델 찾기

[모델 목록 페이지](https://openrouter.ai/models?max_price=0)에서 가격 필터를 FREE로 설정하면 무료 모델만 볼 수 있다. 아래처럼 API로도 확인할 수 있는데, 모델 ID가 `:free`로 끝나는 것들이다.

```bash
curl -s https://openrouter.ai/api/v1/models | \
  jq -r '.data[].id | select(endswith(":free"))'
```

이 글을 쓰는 시점(2026-07-15) 기준으로 사용할 수 있는 무료 모델은 20개다. Llama 3.3 70B, Qwen3 Coder 480B, NVIDIA Nemotron 3 Ultra 550B 같은 대형 모델도 무료로 테스트할 수 있다.

![OpenRouter Free Models]({{site.baseurl}}/assets/img/openrouter_free_models.png)

## API 호출



API의 모델명에 "openrouter/free"로 적으면 프리 모델중에 인풋에 알맞는 모델을 선택해서 응답을 해준다. ( 어떤 모델이 선택됐는지 알 수는 없다 )

### curl

```bash
curl https://openrouter.ai/api/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -d '{
    "model": "meta-llama/llama-3.3-70b-instruct:free",
    "messages": [
      {"role": "user", "content": "안녕하세요. 자기소개 해주세요."}
    ]
  }'
```

### Python (OpenAI SDK)

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)

resp = client.chat.completions.create(
    model="qwen/qwen3-coder:free",
    messages=[{"role": "user", "content": "파이썬으로 퀵정렬 구현해줘"}],
)
print(resp.choices[0].message.content)
```

## 무료 모델 사용 한도

무료 모델에는 요청 횟수 제한이 있다 (2026년 7월 기준).


| 구분               | 분당 요청 | 일일 요청  |
| ---------------- | ----- | ------ |
| 기본               | 20회   | 50회    |
| 크레딧 $10 이상 구매 이력 | 20회   | 1,000회 |


$10만 한 번 충전해두면 하루 1,000회까지 늘어나므로, 본격적으로 테스트할 계획이라면 최소 크레딧을 넣어두는 것이 편하다. 무료 모델은 프로바이더 사정에 따라 목록이 자주 바뀌고, 입력한 데이터가 학습에 사용될 수 있다는 점도 유의해야 한다.

## 마무리

VSCode [AI TOOLS]({{site.baseurl}}/tools/2025/04/13/aitools.html) 플러그인에 커스텀 모델로 오픈라우터 프리모델을 등록하고 플레이그라운에서 여러모델을 비교하며 가장 적합한 모델을 찾을 수 있다.

## 참고

- [OpenRouter Quickstart](https://openrouter.ai/docs/quickstart){:target="_blank"}
- [OpenRouter API Rate Limits](https://openrouter.ai/docs/api-reference/limits){:target="_blank"}
- [무료 모델 목록](https://openrouter.ai/models?max_price=0){:target="_blank"}

