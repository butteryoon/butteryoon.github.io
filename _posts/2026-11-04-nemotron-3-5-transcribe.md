---
layout: post
comments: true
title: "NVIDIA Nemotron 3.5 Transcribe: 프롬프트로 제어하는 다국어 음성 인식·화자 분리·번역 통합 모델"
description: "NVIDIA Nemotron 3.5 Transcribe는 40개 언어 전사, 31개 언어 번역, 13개 언어 화자 분리를 체크포인트 하나로 처리한다. vLLM 서빙, FP8·NVFP4 양자화본, 프롬프트 제어, NeMo Speech 파인튜닝 레시피까지 정리했다."
img: nemotron-3-5-transcribe_title.webp
date: 2026-11-04 20:20:00 +0900
last_modified_at: 2026-11-04 20:20:00 +0900
tags: [nvidia, nemotron, asr, transcription, llm-model, huggingface]
related: llm-model
categories: dev
source_url: https://huggingface.co/blog/nvidia/nemotron-transcribe-granary
source_date: 2026-10-28
---

NVIDIA가 10월 28일 Hugging Face 블로그에 Nemotron 3.5 Transcribe와 학습 데이터 Granary v2를 공개했다. 전사, 화자 분리, 번역, 오디오 Q&A를 체크포인트 하나가 처리하고 어떤 출력을 낼지는 자연어 프롬프트로 정한다. 도메인 용어가 많으면 프롬프트로 먼저 다듬고 그래도 모자라면 공개된 레시피로 파인튜닝한다. 음성 에이전트나 회의록 파이프라인에 붙일 STT를 고르는 사람이라면 볼 만한 구성이다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Nemotron 3.5 Transcribe(30B-A4B)는 Hugging Face와 ModelScope에서 받을 수 있는 오픈웨이트 음성 모델이다. 40개 언어를 전사하고 그중 31개 언어는 영어와 상호 번역하며 13개 언어에서 화자를 구분한다. 녹음 한 개는 최대 1시간, 화자는 최대 8명까지 다룬다. 지금 vLLM으로 서빙할 수 있다. 실시간 스트리밍이 아니라 녹음 파일용이다. 의료 도메인 사례에서는 파인튜닝으로 처음 보는 화자의 WER을 최선 프롬프트 대비 45.8% 상대 감소시켰다.

## 모델 개요와 접근 경로

Nemotron 3.5 Transcribe는 음성 인코더에 Mamba–Transformer 하이브리드 MoE 언어 모델을 붙인 구조다. 전체 30B 파라미터 가운데 토큰마다 활성화되는 것은 약 4B다. 가중치는 Hugging Face 모델 카드(`nvidia/NVIDIA-Nemotron-3.5-Transcribe-30B-A4B`)와 ModelScope 컬렉션에서 받는다. 학습 데이터셋 `nvidia/Granary-v2`와 NeMo Speech 파인튜닝 튜토리얼도 함께 나왔다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"NVIDIA Nemotron 3.5 Transcribe is an open-weight, promptable model that turns recorded speech into useful text, now available on Hugging Face and ModelScope. It brings transcription, speaker attribution (diarization), translation, and audio question answering into one checkpoint: it transcribes 40 languages, translates 31 of them to or from English, and attributes speakers in 13."</blockquote></details>

입력 한도도 원문에 적혀 있다. 녹음은 최대 1시간, 화자는 최대 8명이다. 용도는 녹음된 오디오이고 라이브 스트리밍은 대상이 아니다. 실시간 대화형 음성 에이전트를 만든다면 [Nemotron Voice Agent로 말하는 할로윈 집 만들기]({{site.baseurl}}/dev/2026/11/03/nemotron-voice-agent-haunted-house.html)에서 쓴 스트리밍 ASR 쪽이 맞다. Transcribe는 회의 녹음이나 강의 다시보기처럼 끝난 녹음을 후처리할 때 쓰는 모델이다.

## 프롬프트로 작업 전환하기

체크포인트는 그대로 두고 지시문만 바꾸면 같은 회의 녹음에서 다른 결과물이 나온다. 원문이 든 예시는 이렇다.

- `Transcribe this recording and label each speaker.`
- `Translate this audio into English.`
- `What decisions were made in this meeting?`

생소한 용어가 많을 때 원문은 학습보다 컨텍스트를 먼저 써 보라고 권한다. 의료 녹음이라면 다음처럼 후보 용어와 표기 규칙을 프롬프트에 넣는다.

```
Transcribe this medical discussion in the original language. Possible terms: HbA1c, metformin. Use these terms only when supported by the audio. Preserve standard medical spelling and write numbers in conventional written form.
```

<details class="evidence"><summary>원문 근거</summary><blockquote>"For unfamiliar terms, try context before training. In a healthcare setting, for example: Transcribe this medical discussion in the original language. Possible terms: HbA1c, metformin. Use these terms only when supported by the audio. Preserve standard medical spelling and write numbers in conventional written form."</blockquote></details>

권장 프리셋(오디오 질문용 reasoning 프리셋 포함), 전체 프롬프트 가이드, 언어별 작업 지원 매트릭스는 모델 카드에 있다. 지원 언어는 작업마다 다르다. 오디오 Q&A는 영어로만 학습됐다. 단어 단위 타임스탬프도 나온다.

## 벤치마크 성능

원문이 제시한 수치를 항목별로 정리했다.

- **Hugging Face Open ASR Leaderboard 영어 트랙**: 오픈웨이트 모델 1위, 평균 WER 5.33%(그림 1 기준 RTFx 462.09)
- **같은 리더보드 다국어 트랙**: 7개 언어 평균 WER 4.62%, 모델 카드의 Open ASR 비교에 든 오픈웨이트 모델 중 최저
- **장문 트랙**: 평균 WER 10.98%, 같은 비교에서 최저
- **Artificial Analysis**: AA-WER 2.8%로 오픈웨이트 공동 1위. 공동 1위 모델의 활성 파라미터는 24B, Transcribe는 약 4B다
- **David AI 디아라이제이션 트랙(21개 언어)**: 14개 시스템 중 DER 최저(18.26%). NVIDIA 전용 화자 분리 모델 Nemotron 3 Diarization(18.38%)을 근소하게 앞섰다
- **VoiceArena Diarization-Bench(영어 전체 세션)**: 14개 시스템 중 cpWER 최저(32.4%), 대면·온라인 두 환경 모두 1위

<details class="evidence"><summary>원문 근거</summary><blockquote>"Among open-weight models, Transcribe leads on what was said; on who said it, it tops two speaker-attribution benchmarks outright. Hugging Face Open ASR Leaderboard. The leaderboard measures transcription accuracy as word error rate (WER) on public test sets, in three tracks: English: first among open-weight models at 5.33% average WER (Figure 1)."</blockquote></details>

<details class="evidence"><summary>원문 근거</summary><blockquote>"On the benchmark's diarization track, which scores who spoke when across 21 languages, it has the lowest diarization error rate of the 14 systems tested, slightly ahead of NVIDIA's dedicated Nemotron 3 Diarization model."</blockquote></details>

처리량 수치도 있다. 곧 나올 NVFP4 양자화본을 RTX PRO 6000 Blackwell Server Edition GPU 한 장에서 돌려 오디오 100시간을 3분 안에 처리했다. 30초 클립을 높은 동시성으로 넣어 잰 값이라 긴 녹음 한 개를 넣을 때의 체감 속도와는 다를 수 있다.

## 로컬 실행과 서빙 경로

원문에 따르면 양자화본이 지원돼 RTX Spark, DGX Spark, Jetson AGX Thor에서 로컬로 돌릴 수 있고 규모를 키우려면 Blackwell 데이터센터 GPU로 간다. 서빙 경로는 세 가지다.

- **vLLM**: 지금 공개된 체크포인트를 바로 서빙할 수 있다. NeMo Speech 저장소에 vLLM 서빙 문서가 있다
- **NeMo-Speech.cpp**: 양자화본과 함께 로컬 배포용으로 제공된다
- **SGLang-Omni**: 공개 첫날부터 셀프 호스팅을 지원한다

직접 띄우기 번거로우면 Baseten, DeepInfra, Simplismart 호스팅 엔드포인트를 쓰면 된다. 검수 시점(11월 4일)에 Hugging Face에는 NVIDIA 공식 `-FP8`, `-NVFP4` 저장소가 올라와 있었다. 원문은 VRAM 요구량을 밝히지 않았으니 GPU 사양은 모델 카드를 보고 정해야 한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Serve the available checkpoint with vLLM today. Local deployment with NeMo-Speech.cpp is available alongside the quantized versions. Transcribe is built for recorded audio, not live streaming."</blockquote></details>

## Granary v2와 파인튜닝으로 도메인 적응

Granary v2의 공개 매니페스트는 48개 언어, 16개 원천 코퍼스, 463,062시간 분량이다. Granary v1의 25개 언어보다 늘었다. 다만 이 수치는 데이터셋 공개본 기준이고 Transcribe를 학습할 때 쓴 전체 데이터 구성이나 샘플링 비중과는 다르다고 원문이 밝힌다. 큐레이션은 NVIDIA NeMo Curator로 했고 그 파이프라인도 공개돼 있어 자체 데이터에 그대로 쓸 수 있다.

의료 사례 연구는 Eka Care의 공개 벤치마크(3,619개 클립)를 튜닝용 855개와 평가용 2,764개로 나눠 진행했다. 결과는 이렇다.

- 전체 테스트셋 WER: 기본 30.8% → 프롬프트 개선(영어와 라틴 문자 표기 지정) 23.8% → 파인튜닝 11.8%
- 학습에 없던 화자(held-out) WER: 최선 프롬프트 22.44% → 파인튜닝 12.17%, 45.8% 상대 감소
- 같은 화자 기준으로 공개 의료 특화 모델(13.79%)과 범용 ASR 모델(14.92%)보다 낮다
- 일반 영어 LibriSpeech test-clean WER은 1.54%에서 1.69%로 소폭 올랐다
- 텍스트 생성부터 평가까지 전 과정에 NVIDIA GB300 한 장으로 약 14 GPU시간이 들었다

<details class="evidence"><summary>원문 근거</summary><blockquote>"Using Eka Care's KARMA scoring pipeline, fine-tuning cuts held-out WER from 22.44% for the best-prompt baseline to 12.17%, a 45.8% relative reduction and the clearest evidence that the gains generalize to new speakers."</blockquote></details>

학습 데이터는 합성 음성 약 153시간, Common Voice 억양 영어 29시간, MultiMed 의료 녹음 63시간이다. 파인튜닝 튜토리얼과 함께 Claude Code용 에이전트 스킬(`finetune-speech-llm`)도 저장소에 들어 있다. ModelScope 쪽에서는 ms-swift 프레임워크로도 파인튜닝할 수 있다.

## 실전 적용 체크리스트

자기 환경에서 바로 해 볼 수 있는 순서로 정리했다.

1. **데모로 먼저 확인**: Hugging Face Space(`nvidia/nemotron-transcribe`)에 자기 녹음을 넣어 품질을 본다
2. **모델 내려받기**: `hf download nvidia/NVIDIA-Nemotron-3.5-Transcribe-30B-A4B`, GPU 메모리가 빠듯하면 `-FP8`이나 `-NVFP4` 저장소를 받는다
3. **서빙**: NeMo Speech의 vLLM 서빙 문서를 따라 띄우거나 로컬 단말이면 NeMo-Speech.cpp를 쓴다
4. **프롬프트 실험**: 같은 녹음을 전사·번역·화자 분리·Q&A 프롬프트로 돌려 출력을 비교하고 도메인 용어는 `Possible terms:` 형식으로 넣어 본다
5. **도메인 적응**: 프롬프트로 부족하면 NeMo Speech 파인튜닝 튜토리얼과 의료 사례 문서를 따라 자체 데이터로 학습한다
6. **타임스탬프 활용**: Agora는 단어 단위 타임스탬프로 녹화된 수업 다시보기 옆에 전사문을 띄워 학생이 무슨 말이 언제 나왔는지 볼 수 있게 했다

## 참고 링크

- [Hugging Face 블로그 원문](https://huggingface.co/blog/nvidia/nemotron-transcribe-granary){:target="_blank"}
- [모델 카드](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Transcribe-30B-A4B){:target="_blank"} · [FP8](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Transcribe-30B-A4B-FP8){:target="_blank"} · [NVFP4](https://huggingface.co/nvidia/NVIDIA-Nemotron-3.5-Transcribe-30B-A4B-NVFP4){:target="_blank"}
- [Hugging Face 데모 Space](https://huggingface.co/spaces/nvidia/nemotron-transcribe){:target="_blank"}
- [ModelScope 컬렉션](https://modelscope.ai/collections/nv-community/Nemotron-35-Transcribe){:target="_blank"}
- [Granary v2 데이터셋](https://huggingface.co/datasets/nvidia/Granary-v2){:target="_blank"}
- [NeMo Speech vLLM 서빙 문서](https://github.com/NVIDIA-NeMo/Speech/blob/main/docs/source/speechlm2/vllm_serving.rst){:target="_blank"}
- [NeMo-Speech.cpp](https://github.com/NVIDIA/NeMo-Speech.cpp){:target="_blank"}
- [NeMo Speech 파인튜닝 튜토리얼](https://github.com/NVIDIA-NeMo/Speech/blob/main/tutorials/speechlm2/finetuning/README.md){:target="_blank"}
- [의료 ASR 합성 데이터 사례](https://github.com/NVIDIA-NeMo/Speech/blob/main/examples/speechlm2/finetuning/docs/medical-asr-synthetic-data.md){:target="_blank"}
- [Hugging Face Open ASR Leaderboard](https://huggingface.co/spaces/hf-audio/open_asr_leaderboard){:target="_blank"}
- [Artificial Analysis Speech-to-Text](https://artificialanalysis.ai/speech-to-text/non-streaming){:target="_blank"}
- [David AI 디아라이제이션 리더보드](https://research.withdavid.ai/blog/dai-asr-i18n#leaderboard){:target="_blank"}
- [VoiceArena Diarization-Bench 방법론](https://voicearena.com/diarization-bench/methodology){:target="_blank"}
- [Agora 블로그(수업 다시보기 전사 데모)](https://www.agora.io/en/blog/how-live-teaching-can-become-a-lasting-learning-resource/){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/fxHq67K11-E){:target="_blank"}
