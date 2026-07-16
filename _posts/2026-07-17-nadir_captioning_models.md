---
layout: post
comments: true
title: "나디르 위성영상 캡셔닝을 위한 오픈웨이트 멀티모달 모델 비교"
description: "수직(나디르) 시점 위성영상 캡셔닝에 쓸 수 있는 오픈웨이트 VLM들 — 범용 모델과 원격탐사 특화 모델 비교"
img: nadir_aerial_title.jpg
date: 2026-07-17 02:00:00 +0900
last_modified_at: 2026-07-17 02:00:00 +0900
tags: [llm, multimodal, VLM, remote sensing, captioning, open weight] # add tag
related: llm
categories: dev
---

나디르(nadir, 수직 하방) 시점 위성영상에 텍스트 캡션을 자동으로 붙이는 작업에 어떤 오픈웨이트 멀티모달 모델을 쓸 수 있는지 조사한 내용을 정리한다. [지난 글]({{site.baseurl}}/dev/2026/07/16/satellite_mllm_vs_embedding.html)에서 MLLM과 임베딩 모델의 역할 차이를 다뤘다면, 이번에는 캡셔닝(MLLM 쪽)에 초점을 맞춘다.

<!--more-->

> **TL;DR:** 범용 오픈웨이트 VLM(Qwen3-VL, InternVL3, Gemma 3 등)은 그대로 쓰면 위성영상 도메인 갭 때문에 캡션 품질이 떨어진다. 소규모로 시작한다면 원격탐사 특화 모델(RS-LLaVA, GeoChat, SkyEyeGPT)을 먼저 평가하고, 자체 데이터가 있다면 범용 최신 모델에 LoRA 파인튜닝하는 것이 현재 가장 실용적인 선택이다.

## 나디르 위성영상 캡셔닝은 무엇이 어려운가?

일반 사진과 달리 나디르 시점 영상에는 범용 VLM이 학습한 데이터와의 **도메인 갭**이 존재한다.

- **시점**: 학습 데이터 대부분이 지상 시점 사진이라, 수직 하방에서 본 객체(건물 지붕, 차량 상면)를 오인식하기 쉽다
- **스케일**: 한 장에 수 km² 영역이 담기고 관심 객체(차량, 항공기)는 수십 픽셀에 불과하다
- **회전 불변성**: 위성영상에는 "위쪽"이 없다 — 같은 장면이 임의 각도로 회전되어 들어온다
- **환각**: 지상 사진에서 흔한 맥락(사람, 간판 등)을 영상에 없는데도 지어내는 경향이 있다

## 어떤 오픈웨이트 모델을 쓸 수 있나?

### 범용 오픈웨이트 VLM (2026년 기준)

| 모델 | 규모 | 라이선스 | 특징 |
| :--- | :--- | :--- | :--- |
| Qwen3-VL | 다양한 크기 | Apache 2.0 | Qwen 시리즈 최신, 멀티모달 추론·긴 컨텍스트 강화 |
| Qwen2.5-VL-72B | 72B | Qwen | MMMU ~70%, OCR 강점, 오픈웨이트 최상위권 |
| InternVL3-78B | 78B | MIT | MIT 라이선스 중 최강급 (MMMU ~72%) |
| Gemma 3 | 4B~27B | Gemma | 128K 컨텍스트, 가벼운 배포 |
| Pixtral 12B | 12B | Apache 2.0 | 12B급에서 강력 |
| Phi-4-Multimodal | ~5B | MIT | 소형 클래스 최강, 엣지 배포 후보 |

벤치마크(MMMU, OCRBench 등) 기준으로는 상용 모델과 경쟁하는 수준이지만, 이 점수는 **일반 도메인** 기준이라 위성영상에 그대로 이어지지 않는다는 점을 유의해야 한다.

### 원격탐사 특화 모델

| 모델 | 베이스 | 특징 |
| :--- | :--- | :--- |
| RS-LLaVA | LLaVA + LoRA | RS-instructions 데이터셋으로 캡셔닝·VQA 공동 학습 |
| GeoChat | LLaVA-1.5 + LoRA | 영역 지정 질의와 공간 좌표 그라운딩 지원 |
| SkyEyeGPT | 자체 구조 | 8개 RS 데이터셋에서 이미지/영역 캡셔닝 검증, RS 비디오 캡셔닝 지원 |

셋 다 LLaVA 계열의 검증된 레시피(시각 인코더 + 어댑터 + LLM, LoRA 파인튜닝)를 원격탐사 데이터(RSICD 등)에 적용한 것이라, 나디르 영상 캡션의 도메인 어휘(토지 피복, 시설물 유형)를 범용 모델보다 잘 구사한다. 단점은 베이스 LLM이 최신 모델 대비 한 세대 뒤라는 점이다.

## 어떻게 선택해야 하나?

1. **빠른 검증이 목표** → RS-LLaVA/GeoChat을 그대로 돌려 자체 영상에서 캡션 품질을 먼저 확인한다. 특화 모델이 기대에 못 미치면 그 격차가 파인튜닝 필요성의 근거가 된다.
2. **자체 캡션 데이터가 있다** → 최신 범용 모델(Qwen3-VL, InternVL3 등)에 LoRA 파인튜닝하는 편이 특화 모델의 구식 베이스를 쓰는 것보다 상한선이 높다. GeoChat·RS-LLaVA가 이미 같은 방식(LoRA)으로 효과를 입증했다.
3. **라이선스 제약이 있다** → 상용 활용이라면 MIT(InternVL3, Phi-4)나 Apache 2.0(Qwen3-VL, Pixtral) 라이선스를 우선 검토한다.
4. **대량 처리 파이프라인** → 캡셔닝 전에 [임베딩 모델로 필터링]({{site.baseurl}}/dev/2026/07/16/satellite_mllm_vs_embedding.html)해서 MLLM 호출량을 줄이는 하이브리드 구성을 권장한다.

## 평가는 어떻게 하나?

원격탐사 캡셔닝의 표준 벤치마크로 RSICD, UCM-Captions, Sydney-Captions 등이 쓰이고, 지표는 BLEU/METEOR/CIDEr가 일반적이다. 다만 자동 지표는 도메인 어휘의 정확성(시설물 오인식 등)을 잘 잡지 못하므로, 실제 도입 시에는 **자체 영상 표본에 대한 전문가 검수**를 병행하는 것이 안전하다.

## 참고

- [Awesome Remote Sensing MLLM (서베이 모음)](https://github.com/ZhanYang-nwpu/Awesome-Remote-Sensing-Multimodal-Large-Language-Model){:target="_blank"}
- [RS-LLaVA 논문](https://www.mdpi.com/2072-4292/16/9/1477){:target="_blank"}
- [GeoChat (CVPR 2024)](https://mbzuai-oryx.github.io/GeoChat/){:target="_blank"}
- [SkyEyeGPT](https://www.sciencedirect.com/science/article/pii/S0924271625000206){:target="_blank"}
- [오픈웨이트 VLM 서베이 (2026)](https://blog.overshoot.ai/blog/vlm-survey-2026){:target="_blank"}
