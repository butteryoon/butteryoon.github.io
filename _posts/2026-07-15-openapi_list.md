---
layout: post
comments: true
title: "사이드프로젝트용 무료 OPEN API 목록 정리"
description: "사이드프로젝트에 바로 쓸 수 있는 무료 OPEN API들을 무료 한도와 함께 정리"
img: api_code_title.jpg
date: 2026-07-15 00:00:00 +0900
last_modified_at: 2026-07-16 02:40:00 +0900
tags: [openapi, api]
related: openapi
categories: tools
redirect_from:
  - /tools/2026/07/14/openapi_list.html
---

2021년에 링크만 모아두고 방치했던 초안을 보완해서 발행한다. 사이드프로젝트를 시작할 때 데이터 소스로 바로 쓸 수 있는 무료 OPEN API들을 무료 한도와 함께 정리했다.
<!--more-->

## 목록

| API | 한줄 설명 | 무료 한도 |
|---|---|---|
| [OpenWeather](https://openweathermap.org/api){:target="_blank"} | 현재 날씨, 예보, 대기오염 등 날씨 데이터 API | 60 calls/min, 월 100만 콜 수준. 이메일만으로 API 키 발급 |
| [geo.ipify (IP Geolocation)](https://geo.ipify.org/){:target="_blank"} | IP 주소로 국가/도시/좌표/ISP를 조회하는 API | 월 1,000회 무료 lookup |
| [Unsplash API](https://unsplash.com/developers){:target="_blank"} | 고해상도 무료 사진 검색/랜덤 이미지 API | 데모 모드 시간당 50회, 프로덕션 승인 시 시간당 5,000회 |
| [Open-Meteo](https://open-meteo.com/){:target="_blank"} | API 키 없이 바로 쓰는 오픈소스 날씨 API | 비상업적 용도 무료, 키 발급조차 필요 없음 |
| [TMDB](https://developer.themoviedb.org/){:target="_blank"} | 영화/TV 프로그램 메타데이터, 포스터 이미지 API | 비상업적 용도 무료 (API 키 발급) |
| [ExchangeRate-API](https://www.exchangerate-api.com/){:target="_blank"} | 환율 데이터 API | Open access는 키 없이, 무료 플랜 월 1,500회 |

간단히 특징을 보면,

- **OpenWeather**: 날씨 앱 만들 때 가장 많이 쓰는 정석. 무료 티어로도 토이 프로젝트에는 충분하다.
- **geo.ipify**: 접속자 IP 기반 위치 표시 같은 기능에 유용. 무료 한도가 월 1,000회로 적은 편이라 캐싱 필수.
- **Unsplash**: 배경/썸네일 이미지가 필요할 때. 상업적 이용도 가능하지만 출처 표기(attribution) 규칙을 지켜야 한다.
- **Open-Meteo**: 회원가입/키 발급이 귀찮으면 이쪽. URL 호출만으로 바로 JSON 응답을 받는다.
- **TMDB**: 영화 검색/추천 서비스 연습용으로 인기. 문서가 잘 되어 있다.
- **ExchangeRate-API**: 환율 계산기, 해외직구 가격 비교 같은 프로젝트에 쓰기 좋다.

## 참고

- [OpenWeather Pricing](https://openweathermap.org/price){:target="_blank"}
- [geo.ipify Documentation](https://geo.ipify.org/docs){:target="_blank"}
- [Unsplash API Guidelines](https://unsplash.com/documentation){:target="_blank"}
- [public-apis (GitHub) - 무료 API 모음 저장소](https://github.com/public-apis/public-apis){:target="_blank"}
