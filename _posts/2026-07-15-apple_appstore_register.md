---
layout: post
comments: true
title: "애플 앱스토어에 앱 등록하기"
description: "Apple Developer Program 가입부터 App Store Connect 앱 심사 제출까지의 절차 개요"
img: apple-title.jpg
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-15 00:00:00 +0900
tags: [apple, appstore, ios] # add tag
related: apple
categories: tools
---

2020년에 초안으로 남겨두었던 글을 보완해서 발행한다. 애플 앱스토어에 앱을 등록해서 출시하기까지의 전체 절차를 단계별로 정리해본다.

<!--more-->

## 1. Apple Developer Program 가입

앱스토어에 앱을 올리려면 [Apple Developer Program](https://developer.apple.com/programs/){:target="_blank"} 가입이 필수다. 연회비는 **연간 $99**(개인/조직 동일)이며 매년 갱신해야 한다. 갱신하지 않으면 스토어에 올려둔 앱이 내려간다. Apple ID로 [developer.apple.com](https://developer.apple.com){:target="_blank"} 또는 iOS의 Apple Developer 앱에서 가입할 수 있고, 조직 명의로 가입하려면 D-U-N-S 번호가 필요하다.

## 2. 번들 ID, 인증서, 프로비저닝 프로파일

개발자 계정의 **Certificates, Identifiers & Profiles(인증서, 식별자 및 프로파일)** 메뉴에서 앱의 신원을 준비한다.

- **번들 ID(Bundle ID)**: `com.회사명.앱이름` 형태의 역도메인 방식 고유 식별자. 한 번 앱에 연결하면 변경할 수 없으므로 신중하게 정한다.
- **배포용 인증서(Distribution Certificate)**: 앱스토어 제출용 코드 서명 인증서를 발급한다.
- **프로비저닝 프로파일(Provisioning Profile)**: 번들 ID와 인증서를 묶어 App Store 배포용 프로파일을 생성한다.

요즘은 Xcode의 **Automatically manage signing** 옵션을 켜두면 인증서와 프로파일을 자동으로 처리해줘서 직접 만들 일이 많이 줄었다.

## 3. App Store Connect에 앱 생성

[App Store Connect](https://appstoreconnect.apple.com){:target="_blank"}는 앱 등록, 심사 제출, 판매 현황, 리뷰 관리를 하는 곳이다. **나의 앱(My Apps) > + > 신규 앱**에서 앱 이름, 기본 언어, 번들 ID(2단계에서 만든 것 선택), SKU를 입력해 앱을 생성한다.

이후 스토어 페이지에 들어갈 메타데이터를 채운다.

- 앱 설명, 키워드, 지원 URL
- 스크린샷(기기 크기별), 앱 아이콘
- **개인정보 처리방침 URL** — 필수 항목이고 심사 거절의 단골 사유이므로 미리 웹에 게시해둔다
- 앱 개인정보 보호(수집하는 데이터 항목) 설문, 연령 등급, 가격 및 판매 지역

## 4. 빌드 업로드와 심사 제출

Xcode에서 **Product > Archive**로 배포용 빌드를 만들고 **Distribute App**으로 App Store Connect에 업로드한다. 업로드된 빌드는 처리(약 수십 분)가 끝나면 App Store Connect에서 선택할 수 있다. 실기기 테스트가 필요하면 TestFlight로 베타 배포를 먼저 해봐도 좋다.

앱 버전 페이지에서 빌드를 선택하고 **심사에 제출(Submit for Review)** 을 누르면 끝. 심사는 보통 **24~48시간, 길어야 1~3일** 안에 결과가 나온다. 거절되면 사유가 함께 오는데, 수정 후 재제출하면 된다. 승인되면 설정에 따라 자동 또는 수동으로 앱스토어에 출시된다.

## 정리

1. Apple Developer Program 가입 ($99/년)
2. 번들 ID·인증서·프로비저닝 프로파일 준비
3. App Store Connect에서 앱 생성 및 메타데이터 입력
4. 빌드 업로드 후 심사 제출

절차 자체는 정형화되어 있어서 어렵지 않지만, 개인정보 처리방침 URL과 스크린샷 규격 같은 메타데이터 준비에서 시간이 많이 든다. 심사 가이드라인을 미리 훑어보고 제출하면 거절 확률을 줄일 수 있다.

## 참고

- [Submitting - App Store - Apple Developer](https://developer.apple.com/app-store/submitting/){:target="_blank"}
- [App Store Connect - Apple Developer](https://developer.apple.com/app-store-connect/){:target="_blank"}
- [App Review Guidelines](https://developer.apple.com/app-store/review/guidelines/){:target="_blank"}
- [How to use App Store Connect in 2026 (MobileAction)](https://www.mobileaction.co/blog/how-to-use-app-store-connect/){:target="_blank"}
