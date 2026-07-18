---
layout: post
comments: true
title: "오픈소스 VOIP 구성요소 정리"
description: "오픈소스로 VOIP 시스템을 구성할 때 쓰는 시그널링/미디어 엔진들을 정리해본다."
img: protocol_title.png
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-15 00:00:00 +0900
tags: [voip, sip, opensips, rtpengine, kamailio] # add tag
related: opensource
categories: dev
redirect_from:
  - /dev/2026/07/14/survey_voip.html
---

2021년에 OpenSIPS 2.4와 RTPEngine mr7.4 조합을 메모해둔 초안을 보완해서 발행한다. 오픈소스로 VOIP 시스템을 만들 때 쓰이는 주요 구성요소를 현재(2026년) 기준으로 정리해본다.

<!--more-->

VOIP 시스템은 크게 두 축으로 나뉜다. 호 설정/해제를 담당하는 **시그널링(SIP)** 과, 실제 음성 패킷을 나르는 **미디어(RTP)** 다. 오픈소스 세계에서는 이 둘을 별도 컴포넌트로 분리해 조합하는 구성이 일반적이다.

## 시그널링 엔진: OpenSIPS

- GitHub: [OpenSIPS/opensips](https://github.com/OpenSIPS/opensips)
- 고성능 SIP 프록시/라우터. 초안 작성 당시 2.4를 썼는데 현재는 4.0이 최신 stable이고 3.6이 LTS다.
- SIP 라우팅, 로드밸런싱, NAT 트래버설, 과금 연동 등 통신사급 트래픽 처리에 쓰인다. 미디어는 직접 다루지 않고 RTPEngine 같은 미디어 프록시에 위임한다.

## 미디어 엔진: RTPEngine

- GitHub: [sipwise/rtpengine](https://github.com/sipwise/rtpengine)
- Sipwise가 만든 커널 레벨 RTP 프록시. 초안 당시 mr7.4였고 현재는 mr13.x대까지 릴리즈가 이어지고 있다(mr12.5, mr13.5 등이 LTS).
- 커널 모듈로 패킷을 포워딩해서 성능이 좋고, SRTP 암복호화, 트랜스코딩, 녹취, WebRTC(DTLS-SRTP) 브릿징까지 지원한다. OpenSIPS/Kamailio 양쪽 모두에서 표준처럼 쓰인다.

## 그 외 눈여겨볼 프로젝트

- **[Kamailio](https://github.com/kamailio/kamailio)** : OpenSIPS와 뿌리(OpenSER)가 같은 SIP 프록시. 기능과 용도가 거의 겹치며 커뮤니티/모듈 생태계가 크다. 현재 6.x대가 stable이다. OpenSIPS냐 Kamailio냐는 사실상 취향과 설정 스크립트 스타일 차이에 가깝다.
- **[FreeSWITCH](https://github.com/signalwire/freeswitch)** : 프록시가 아니라 B2BUA/소프트스위치. IVR, 컨퍼런스, 트랜스코딩 등 애플리케이션 레벨 기능이 필요할 때 쓴다. 대규모 구성에서는 앞단에 OpenSIPS/Kamailio를 두고 뒤에 FreeSWITCH를 배치하는 패턴이 흔하다.
- **[Asterisk](https://github.com/asterisk/asterisk)** : 가장 역사가 긴 오픈소스 PBX. 사내 전화 시스템처럼 PBX 기능이 중심이면 여전히 유효한 선택지다. 현재 22/23 버전대가 유지되고 있다.

정리하면, 대량 호 라우팅은 OpenSIPS 또는 Kamailio + RTPEngine 조합, 통화 기능(IVR/PBX)이 중심이면 FreeSWITCH나 Asterisk를 조합하는 것이 일반적인 오픈소스 VOIP 스택이다.

## 참고
- [OpenSIPS Available Versions](https://www.opensips.org/About/AvailableVersions){:target="_blank"}
- [rtpengine Releases](https://github.com/sipwise/rtpengine/releases){:target="_blank"}
- [Kamailio Releases](https://www.kamailio.org/w/category/releases/){:target="_blank"}
- [안심번호(050) 시스템 #1 - 시스템 구축 스토리 (우아한형제들 기술블로그)](https://techblog.woowahan.com/2705/){:target="_blank"}
