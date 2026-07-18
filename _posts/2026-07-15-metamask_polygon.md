---
layout: post
comments: true
title: "MetaMask에 Polygon 네트워크 추가하기"
description: "MetaMask 지갑에 Polygon PoS 네트워크를 수동으로 추가하는 방법을 정리한다."
img: crypto_title.jpg
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-16 02:40:00 +0900
tags: [MetaMask, Polygon, blockchain, wallet] # add tag
related: MetaMask
categories: tools
redirect_from:
  - /tools/2026/07/14/metamask_polygon.html
---

2022년에 작성해두었던 초안을 보완해서 발행한다. MetaMask 지갑에 Polygon PoS 네트워크를 수동으로 추가하는 방법을 정리해본다.

<!--more-->

## MetaMask 설치

[MetaMask](https://metamask.io/)는 가장 널리 쓰이는 이더리움 계열 지갑이다. [공식 다운로드 페이지](https://metamask.io/download/)에서 브라우저 확장(Chrome, Firefox, Edge 등)이나 모바일 앱을 설치하고 지갑을 생성한다. 시드 문구(Secret Recovery Phrase)는 절대 온라인에 저장하지 말고 안전한 곳에 보관한다.

MetaMask는 기본적으로 이더리움 메인넷만 설정되어 있어서 Polygon 같은 다른 네트워크는 직접 추가해야 한다. 최근 버전에서는 네트워크 선택 목록에 Polygon이 기본 제공 목록으로 떠서 클릭 한 번으로 추가되는 경우도 있지만, 없다면 아래처럼 수동으로 추가한다.

## Polygon PoS 네트워크 수동 추가

MetaMask에서 좌측 상단의 네트워크 선택 드롭다운을 열고 **Add network** → **Add a network manually**를 선택한 뒤 아래 값을 입력한다.

| 항목 | 값 |
| :--- | :--- |
| Network name | Polygon PoS |
| New RPC URL | https://polygon-rpc.com |
| Chain ID | 137 |
| Currency symbol | POL |
| Block explorer URL | https://polygonscan.com |

저장하면 네트워크 목록에 Polygon이 추가되고, 선택하면 바로 전환된다.

> 참고로 Polygon의 네이티브 토큰은 원래 **MATIC**이었는데 2024년에 **POL**로 리브랜딩되었다. 예전 가이드에는 통화 기호가 MATIC으로 나와 있지만 현재는 POL을 사용한다.

초안에 적어두었던 예전 문서 링크는 현재 [docs.polygon.technology의 MetaMask 가이드](https://docs.polygon.technology/tools/wallets/metamask/add-polygon-network/)로 옮겨졌으니 최신 값은 공식 문서를 확인하는 것이 좋다.

## 참고

- [Add Polygon network - Polygon Developer Docs](https://docs.polygon.technology/tools/wallets/metamask/add-polygon-network/){:target="_blank"}
- [MetaMask 다운로드](https://metamask.io/download/){:target="_blank"}
- [PolygonScan](https://polygonscan.com/){:target="_blank"}
