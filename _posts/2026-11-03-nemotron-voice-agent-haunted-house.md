---
layout: post
comments: true
title: "Nemotron Voice Agent로 말하는 할로윈 집 만들기 — 로컬 음성 캐릭터 빌드 로그"
description: "NVIDIA 직원이 Nemotron Speech ASR, Bolt E4B 4GB, Magpie TTS Zeroshot, Audio2Face-3D를 워크스테이션 한 대에서 돌려 투덜대는 해적선 출신 집 캐릭터를 만든 과정. 모델 선택, LoRA로 성격 입히기, 응답 지연 1.70초를 줄인 캐시 전략, 직접 따라 만드는 경로를 정리했다."
img: nemotron_halloween_title.webp
date: 2026-11-03 20:30:00 +0900
last_modified_at: 2026-11-03 20:30:00 +0900
tags: [llm-agent, llm, nvidia, nemotron, voice-agent, tts, asr, audio2face, lora, local-llm]
related: llm-agent
categories: dev
source_url: https://huggingface.co/blog/nvidia/nemotron-halloween
source_date: 2026-10-30
---

Hugging Face 블로그에 10월 30일 올라온 [「Building a Haunted House with NVIDIA Nemotron Open Models」](https://huggingface.co/blog/nvidia/nemotron-halloween){:target="_blank"}는 할로윈용 인터랙티브 프로젝션 'MonsterHouse'의 제작기다. 차고 문에 나무판이 갈라진 입을 달고, 찾아온 아이들의 말을 알아들어 투덜대는 늙은 해적선 목소리로 대꾸하게 했다. 듣기·생각·말하기·입 모양까지 전부 NVIDIA 오픈 모델로 짰고 워크스테이션 한 대에서 돌아간다. 글쓴이가 NVIDIA 직원이라 모델 선택에 그 점은 감안하고 읽는 편이 좋지만, 로컬 음성 에이전트를 직접 만들 때 부딪히는 지연·성격·동기화 문제를 어떻게 풀었는지는 그대로 참고할 만하다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** [Nemotron Voice Agent](https://github.com/NVIDIA-AI-Blueprints/nemotron-voice-agent){:target="_blank"}(ASR → LLM → TTS 캐스케이드)를 뼈대로 대화 두뇌에 Bolt E4B 4GB(4비트 양자화)와 성격용 LoRA 어댑터를, 목소리에 Magpie TTS Zeroshot을, 입 모양에 Audio2Face-3D를 붙였다. 언어·음성 파이프라인은 RTX PRO 6000 Blackwell Max-Q, 언리얼 엔진 그래픽은 RTX PRO 4000 Blackwell로 GPU를 나눴다. 발화가 끝나고 소리가 나오기까지 중앙값은 생성 시 1.70초, 미리 만들어 둔 응답이면 0.72초다. MonsterHouse 코드는 공개되지 않았지만 쓰인 모델은 모두 지금 받아 쓸 수 있다.

## 파이프라인 구성

원문이 밝힌 구성은 여섯 갈래다.

- **듣기**: [Nemotron Speech ASR](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b){:target="_blank"}(스트리밍, 영어 전용 0.6B)이 방문객의 말을 텍스트로 옮긴다. 원문은 저장소에 있는 `nemotron-speech-streaming-en-0.6b.q8_0.gguf` 파일을 가리킨다.
- **생각**: [Nemotron 3.5 Bolt E4B 4GB](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Bolt-E4B-4GB){:target="_blank"}가 대답을 쓴다. 대화형 에이전트용 소형 오픈 모델이고 성격은 LoRA 어댑터가 입힌다.
- **말하기**: [Magpie TTS Zeroshot](https://build.nvidia.com/nvidia/magpie-tts-zeroshot){:target="_blank"}이 짧은 참조 음성 샘플 하나로 캐릭터 목소리를 낸다. 따로 학습한 목소리가 필요 없다.
- **입 모양**: [Audio2Face-3D](https://github.com/NVIDIA/Audio2Face-3D){:target="_blank"}가 깨끗한 음성을 언리얼 엔진 5 안의 입 애니메이션으로 바꾼다.
- **보기**: [Nemotron 3 Nano Omni](https://huggingface.co/nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-BF16){:target="_blank"}가 카메라 영상 속 코스튬을 묘사하고 사람 수 세기와 추적은 YOLO와 ByteTrack이 맡는다.
- **조율**: Nemotron Voice Agent가 이 모두를 워크스테이션 한 대 위에서 엮는다.

글쓴이는 캐릭터 릭을 만드는 데 공을 들이는 사이 Voice Agent가 나와서 가장 어려울 줄 알았던 듣기·대화·발화 부분을 "바로 가져다 쓰는" 수준으로 해결했다고 적었다. 캐릭터 릭은 Autodesk Maya에서 만들어 언리얼 엔진으로 옮겼고 눈·눈썹·깜빡임이 없는 얼굴이라 입 모양만 애니메이션하면 됐다.

## 모델 선택 — Bolt E4B를 고른 이유

글쓴이는 외부 연결 없이 혼자 돌아가는 시스템을 원했다. 그러니 메모리와 연산량 예산이 정해져 있었고 대화 모델은 ASR·TTS·애니메이션에 쓸 몫을 남겨야 했다. 캐릭터는 "투덜대라"는 프롬프트 한 줄로 끝나지 않는다. 인사에서 작별까지 정해진 대화 흐름을 따르고, 방문객이 실제로 한 말에 반응하고, 누가 "please"를 말했는지 같은 만남의 상태를 기억해야 했다.

모델 십여 개를 시험한 끝에 Bolt E4B를 골랐다. 기본 응답 모델로는 4비트 양자화판(이름의 '4GB')을 쓰고 비교용으로 [BF16 체크포인트](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Bolt-E4B-BF16){:target="_blank"}를 둔다. 경쟁 후보였던 Qwen3 30B-A3B, Qwen 3.6 35B-A3B보다 메모리를 덜 차지한다는 게 결정 이유였다. 다만 글쓴이 스스로 NVIDIA 직원이라 공개 전에 Bolt E4B를 써 볼 수 있었다고 밝혔다. Bolt 계열 출시 내용은 [Nemotron 3.5 Super VL·Bolt 공개]({{site.baseurl}}/nvidia/llm-model/2026/10/23/nemotron-3-5-super-vl-bolt.html) 글에 정리해 두었다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"I use the 4-bit quantized release (the “4GB” in its name) as the default response model, with the BF16 checkpoint available for comparison."</blockquote></details>

## LoRA로 성격 입히기

성격은 프롬프트, 애플리케이션 로직, 그리고 검토를 거친 예시 대화 수천 건으로 학습한 LoRA 어댑터가 함께 만든다. 처음엔 "이웃집이나 가 봐, 나 지금 투덜대느라 바빠!" 하고 쫓아내다가 계속 조르면 누그러지고, "please" 한마디면 곧바로 풀리고, "thank you"에는 따뜻하게 배웅한다. 한번 마음을 연 상대에게는 계속 그 태도를 유지한다.

- **데이터**: 멀티턴 대화 5,000건 이상. 로컬 에이전트와 아주 큰 LLM이 데이터를 만들고 심사했다. 밤이나 본업을 하는 동안 돌렸다.
- **학습**: 후보로 남긴 모델마다 LoRA를 따로 학습했고 모델에 따라 한 시간에서 두어 시간 걸렸다. 학습과 벤치마크도 로컬 에이전트가 돌렸다.
- **평가**: 응답의 연속성·타이밍·톤을 확인하고 모델과 어댑터 조합을 품질·반복·속도로 벤치마크한 뒤 실제로 들어 본다. 결국엔 넘어가되 쾌활한 안내원처럼 변하지는 않게 맞추는 게 목표였다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Creating a dataset of over 5000 multi-turn conversations meant I could train LoRAs for each model I felt good enough to be an option.  Some took about an hour to train, some took a couple."</blockquote></details>

## 응답 지연과 GPU 분리

집이 대답하는 데 오래 걸리면 무섭지 않다. 그래서 파이프라인은 답이 다 만들어지기 전에 말할 수 있는 텍스트부터 내보내, 문장 뒷부분이 오는 동안 이미 말을 시작한다. 자주 나오는 인사와 사탕 농담은 미리 준비해 둔다. 텍스트 뱅크가 언어 모델 호출을, 녹음해 둔 오디오가 음성 합성을 건너뛰게 한다.

```text
median end-of-speech-to-audio with Bolt Q4 & TTS = 1.70 seconds
median end-of-speech-to-audio cached responses     = 0.72 seconds
(note: speech-end detection currently waits for 300 ms of silence)
```

Magpie TTS Zeroshot의 목소리는 무드별로 다르다. 집에는 투덜거림·위협부터 재미·기쁨까지 일곱 가지 무드가 있고 프롬프트 템플릿이 정한 감정 태그를 LLM이 붙이면 그에 맞춰 대사를 읽는다. 감정마다 3~10초짜리 샘플이 따로 필요해서 글쓴이는 UI에 'Sample your Voice' 블록을 만들어 해적 목소리로 직접 읽고 녹음했다.

Audio2Face-3D 기본 가중치는 너무 낮지도 높지도 않은 중성적인 남성 또는 여성 목소리에서 가장 잘 동작한다. 원하는 건 울림이 깊은 악마 같은 목소리였지만 녹음 샘플이 원래 담백해서 오히려 유리했다. Audio2Face-3D에는 깨끗한 음성을 넣고 스피커로 나가는 두 번째 오디오 스트림에만 실시간 음성 효과를 건다. 오디오와 입 모양 동기화는 Audio2Face-3D 버퍼링을 보정하는 100ms 지연을 한 번 맞춰 두고 그대로 쓴다.

GPU는 두 장으로 나눴다. 병목은 메모리가 아니라 연산이었고 에이전트 파이프라인이 무슨 일을 하든 애니메이션 프레임레이트가 흔들리지 않게 하려는 목적이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"To keep everything running smoothly, I split the work to a pair of GPUs: the language and voice model pipeline runs on an NVIDIA RTX PRO 6000 Blackwell Max-Q, and Unreal Engine graphics with Audio2Face-3D runs on an NVIDIA RTX PRO 4000 Blackwell.  Memory isn’t the issue, compute is."</blockquote></details>

## 운영 콘솔과 남은 과제

MonsterHouse에는 별도 제어판이 있다. 캐시해 둔 대사 미리 듣기, 시작 무드 조정, 대화 흐름 추적, 음성 효과와 입 애니메이션 튜닝, 모델 로딩, 언리얼 엔진 연결 표시 등을 탭으로 나눴다. 이건 Voice Agent의 기능이 아니라 글쓴이가 직접 만든 화면이다. 모드는 셋이다. 라이브 모드는 방문객 응대, 데모 모드는 집이 자기소개를 하는 용도다. 마이크 모드는 인식·생성·합성을 모두 건너뛰고 글쓴이 목소리로 입을 움직이는 비상용이다.

Nemotron 3 Nano Omni는 선택 기능이다. 코스튬 차림 양 다섯 마리가 나오는 합성 테스트 영상에서 해적·뱀파이어·마녀·호박·유령 의상을 알아봤다. 글쓴이는 이게 실제 할로윈 밤에 통한다는 증거는 아니라고 선을 그었다. 실제 진입로의 무리, 밤 조명, 유행하는 코스튬 인식은 아직 시험해야 하고 재치 있는 응수에는 추가 학습이 필요할 수 있다고 봤다.

## 직접 만들어 보는 경로

원문 말미에 글쓴이가 권하는 순서를 옮긴다.

1. **음성 파이프라인부터**: [Nemotron Voice Agent](https://github.com/NVIDIA-AI-Blueprints/nemotron-voice-agent){:target="_blank"}를 클론해 Generic Assistant 예제를 돌린다. Nemotron Speech ASR, LLM, Magpie TTS로 이어지는 캐스케이드에 음성 구간 검출(VAD)과 발화 종료 처리까지 들어 있다.
2. **두뇌를 Bolt E4B로**: Voice Agent에 클라우드·서버·단일 GPU용 Bolt E4B 프로파일이 들어 있다. 먼저 [build.nvidia.com](https://build.nvidia.com/nvidia/nemotron-3.5-bolt-e4b){:target="_blank"}에서 써 볼 수 있다. 음성에 맞는 짧은 답에는 conversational 모드를 쓰고 단순한 대화에서는 reasoning을 끈다.
3. **성격 입히기**: 시스템 프롬프트와 이어지는 예시 대화 몇 개로 시작한다. 누가 이미 please를 말했는지 같은 대화 상태는 모델에게 기억시키지 말고 애플리케이션에서 관리한다. 프롬프트로 부족하면 LoRA 어댑터를 학습한다. [NeMo AutoModel](https://docs.nvidia.com/nemo/automodel/model-coverage/vision-language-models/nvidia/NVIDIA-Nemotron-3.5-Bolt-E4B-4GB-BF16){:target="_blank"}에 Bolt E4B용 LoRA 레시피가 있다.
4. **목소리**: Voice Agent 기본값은 Magpie TTS Multilingual이다. 캐릭터 목소리가 필요하면 Magpie TTS Zeroshot을 build.nvidia.com에서 써 보고 직접 돌리려면 접근 권한을 요청한다. 사용 허락을 받은 목소리만 쓴다.
5. **얼굴**: Audio2Face-3D에 깨끗한 음성을 넣고 효과는 나중에 더한다.
6. **(선택) 눈**: Voice Agent의 Nemotron Omni Assistant Subagents 예제로 웹캠 이해를 붙인다. 사람 수가 필요하면 가벼운 사람 추적기를 함께 쓴다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Keep conversation state, such as who has already said please, in your application instead of asking the model to remember it."</blockquote></details>

연산 자원이 모자라면 생성 자체를 쇼 밖으로 빼도 된다. 글쓴이 제안은 이렇다. Bolt E4B와 Magpie 파이프라인으로 응답 수백 개를 오프라인에서 만들어 두고 쇼에서는 Nemotron 3 ASR과 파이썬 대화 라우터로 캐시된 오디오만 고르는 라이브 버전을 돌린다. 글쓴이도 다음 작품인 인터랙티브 호박에 이 방식을 쓸 예정이라고 했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The MonsterHouse code itself is not packaged for release, but every NVIDIA model it uses is available today, and the NVIDIA Nemotron Voice Agent gets you most of the way."</blockquote></details>

## 참고

- 원문: [Building a Haunted House with NVIDIA Nemotron Open Models](https://huggingface.co/blog/nvidia/nemotron-halloween){:target="_blank"} (Hugging Face Blog, 2026-10-30)
- [NVIDIA Nemotron Voice Agent — GitHub](https://github.com/NVIDIA-AI-Blueprints/nemotron-voice-agent){:target="_blank"} / [build.nvidia.com](https://build.nvidia.com/nvidia/nemotron-voice-agent){:target="_blank"}
- [Nemotron 3.5 Bolt E4B 4GB — Hugging Face](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Bolt-E4B-4GB){:target="_blank"} / [기술 블로그](https://research.nvidia.com/labs/nemotron/NVIDIA-Nemotron-3.5-Bolt-E4B-4GB/){:target="_blank"}
- [Nemotron Speech ASR (streaming, English)](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b){:target="_blank"}
- [Magpie TTS Zeroshot](https://build.nvidia.com/nvidia/magpie-tts-zeroshot){:target="_blank"}
- [Audio2Face-3D](https://github.com/NVIDIA/Audio2Face-3D){:target="_blank"}
- [Nemotron 3 Nano Omni](https://huggingface.co/nvidia/Nemotron-3-Nano-Omni-30B-A3B-Reasoning-BF16){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/-FR8KNy5r1o){:target="_blank"}

