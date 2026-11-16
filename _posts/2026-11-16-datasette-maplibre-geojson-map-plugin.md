---
layout: post
comments: true
title: "datasette-maplibre 0.1a0: 위경도·GeoJSON 테이블을 벡터 지도로 띄우는 Datasette 플러그인"
description: "Simon Willison이 공개한 datasette-maplibre 0.1a0을 README 기준으로 정리한다. 설치 요건, 지도가 뜨는 컬럼 이름 규칙, GeoJSON 셀 지도, 설정 옵션 아홉 가지, Datasette Lite 데모 실행법을 다루고, 대부분을 Claude Fable 5.2 프롬프트 하나로 만들었다는 개발 과정도 함께 본다."
img: datasette-maplibre-geojson-map-plugin_title.webp
date: 2026-11-16 19:30:00 +0900
last_modified_at: 2026-11-16 20:20:00 +0900
tags: [llm-agent, datasette, maplibre, geospatial, openfreemap, geojson, simon-willison]
related: llm-agent
categories: tools
source_url: https://simonwillison.net/2026/Nov/15/datasette-maplibre/
source_date: 2026-11-15
---

Simon Willison이 Datasette용 지도 플러그인 [datasette-maplibre 0.1a0](https://github.com/datasette/datasette-maplibre/releases/tag/0.1a0){:target="_blank"}을 냈다. 테이블에 위도·경도 컬럼이나 GeoJSON 값이 있으면 그 위에 벡터 지도를 그려 준다. 지도 엔진은 MapLibre GL이고 타일은 무료 벡터 지도 서버 OpenFreeMap에서 받는다. Simon은 릴리스 글에서 이 플러그인의 대부분을 Claude Fable 5.2에 보낸 프롬프트 하나로 만들었다고 밝혔다. 이 글은 저장소 README에 적힌 설치·설정 방법을 정리하고 그 프롬프트가 어떤 것이었는지도 함께 본다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** `datasette install datasette-maplibre`로 설치하며 Datasette 1.0a43 이상이 필요하다. `latitude`/`longitude`, `lat`/`lng` 같은 이름의 컬럼 쌍이 있거나 GeoJSON 행이 여럿이면 테이블 위에 지도가 뜬다. GeoJSON 셀은 SVG 윤곽으로 먼저 그려지고 화면에 들어올 때 WebGL 지도로 바뀌는데, 동시에 살아 있는 지도는 기본 8개다. 뷰는 클러스터·포인트·히트맵 세 가지이고 설정은 인스턴스·데이터베이스·테이블 단위로 줄 수 있다. 설치 없이 Datasette Lite 데모로 브라우저에서 바로 볼 수 있다. 아직 알파(0.1a0)다.

## 무엇을 합친 플러그인인가

Datasette에는 이미 지도 플러그인이 둘 있었다. 좌표 점을 묶어 보여 주는 datasette-cluster-map과 GeoJSON 도형을 그리는 datasette-leaflet-geojson이다. datasette-maplibre는 이 둘의 기능을 벡터 지도 하나로 합치고 몇 가지를 더했다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"It combines the features of datasette-cluster-map and datasette-leaflet-geojson on a vector map, and adds some of its own."</blockquote></details>

지도 쪽 구성은 [MapLibre GL](https://maplibre.org/){:target="_blank"}과 [OpenFreeMap](https://openfreemap.org/){:target="_blank"}이다. Simon은 릴리스 글에서 OpenFreeMap을 Zsolt Ero가 운영하는 무료 벡터 지도 서버라고 소개한다. MapLibre GL JS는 플러그인에 들어 있어 따로 받을 필요가 없지만 스타일·타일·글꼴은 기본 설정에서 외부 서버에서 내려받는다. 외부 통신이 막힌 환경이라면 `style` 옵션으로 다른 스타일을 지정해야 한다는 뜻이다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"MapLibre GL JS 6.17.0 is bundled with the plugin. Map styles, tiles and fonts are loaded from tiles.openfreemap.org unless you configure a different style."</blockquote></details>

## 설치

Datasette가 깔린 환경에서 한 줄이면 된다.

```bash
datasette install datasette-maplibre
```

요구 버전은 Datasette 1.0a43 이상이다. PyPI 메타데이터의 의존성도 `datasette>=1.0a43` 하나뿐이다. 저장소 커밋 기록을 보면 0.x 지원과 우회 코드를 걷어내면서 이 버전을 요구하게 됐다. Datasette 0.x를 쓰고 있다면 이 플러그인은 설치 대상이 아니다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Install this plugin in the same environment as Datasette. It requires Datasette 1.0a43 or later."</blockquote></details>

## 지도가 뜨는 조건

README에 따르면 테이블·SQL 쿼리·행 페이지에 아래 컬럼 쌍 중 하나가 있으면 그 위에 지도가 나타난다.

| 위도 컬럼 | 경도 컬럼 |
|------|------|
| `latitude` | `longitude` |
| `lat` | `lng`, `lon`, `long` |
| `*_latitude` | `*_longitude` |
| `*_lat` | `*_lng`, `*_lon`, `*_long` |

컬럼 이름이 이 규칙에 맞지 않으면 `latitude_column`과 `longitude_column` 설정으로 직접 지정한다. GeoJSON 행이 여럿 있는 테이블과 쿼리에도 지도가 뜨고 이때는 모든 도형을 지도 하나에 그린다. 좌표와 GeoJSON이 둘 다 있으면 점과 도형이 같이 나온다.

행은 Datasette JSON API로 불러온다. 한 페이지에 다 들어가지 않으면 "load all" 버튼으로 나머지를 가져오고 "pause" 버튼으로 멈춘다. 테이블은 한 페이지만 보여 주지만 지도에는 불러온 행이 전부 올라간다.

지도 위 툴바에는 다섯 가지가 있다.

- **View**: 클러스터, 개별 포인트, 히트맵 가운데 고른다.
- **Colour by**: 아무 컬럼으로나 점과 도형을 색칠한다. 고윳값이 10개 이하인 컬럼은 개수가 붙은 범례가 생기고 그보다 많은 숫자 컬럼은 색상 램프로 표시된다.
- **Globe**: 지구본과 평면 지도를 오간다.
- **Filter to this area**: 테이블 페이지에서 지금 지도에 보이는 영역 안의 행만 남도록 `?latitude__gte=...` 같은 필터를 걸어 다시 불러온다.
- **Download GeoJSON**: 지금까지 지도에 올라온 것을 다른 컬럼을 속성으로 담은 GeoJSON 파일로 내려받는다.

알아 둘 동작이 몇 가지 더 있다. 좌표가 완전히 같은 행들은 어느 뷰에서든 숫자가 적힌 원반 하나로 그려지고 누르면 화살표로 행을 넘겨 볼 수 있는 팝업이 열린다. 지도 위치는 URL에 `#map=zoom/latitude/longitude` 형태로 남아서 페이지 링크가 곧 그 화면의 링크가 된다. 뷰·색상·지구본 상태도 설정값과 다르면 `#view=heatmap&color=fuel&globe=1`처럼 URL에 붙는다. 페이지를 스크롤해도 지도는 확대되지 않는다. 확대하려면 ctrl + 스크롤(macOS는 ⌘ + 스크롤)이나 줌 버튼을 쓴다.

## GeoJSON 셀마다 지도 하나

GeoJSON이 든 테이블 셀은 그 도형을 그린 지도로 바뀐다. geometry, `Feature`, `FeatureCollection` 모두 해당한다. 한 페이지에 도형이 수백 개 있어도 버티도록 처리 방식을 둘로 나눴다. 셀마다 SVG 윤곽을 먼저 그리고 화면에 들어오는 셀만 인터랙티브 지도로 바꾼다. 브라우저가 한 페이지의 WebGL 지도 수를 제한하기 때문에 동시에 살아 있는 지도는 8개까지다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Browsers limit how many WebGL maps a page can have, so only eight cells are live maps at once: when a ninth is needed, the oldest one that is off screen goes back to being an outline."</blockquote></details>

Simon이 릴리스 글에서 가장 마음에 든다고 꼽은 기능이 이것이다. 이 상한은 `max_cell_maps` 설정으로 바꿀 수 있다.

팝업 내용도 바꿀 수 있다. 기본은 행 전체를 보여 주지만 JSON이 든 `popup` 컬럼을 쿼리에 넣으면 그 내용으로 대체된다. 형식은 datasette-cluster-map과 같다.

```sql
select json_object(
  'image', photo_url,
  'alt', 'Photo of ' || name,
  'title', name,
  'description', substr(description, 0, 200),
  'link', '/browse/museums/' || id
) as popup, latitude, longitude from museums
```

키는 전부 선택 사항이고 `link`와 `image`는 `http://`·`https://` 또는 상대 URL이어야 한다.

## 설정

설정은 모두 선택 사항이다. 인스턴스 전체, 데이터베이스, 테이블 단위로 줄 수 있고 범위가 좁은 설정이 넓은 설정 위에 덮인다. README의 예시는 다음과 같다.

```json
{
    "plugins": {
        "datasette-maplibre": {
            "latitude_column": "xlat",
            "longitude_column": "xlng",
            "style": "https://example.com/style.json",
            "view": "points",
            "color_column": "category",
            "height": "70vh",
            "scroll_zoom": true,
            "max_cell_maps": 12,
            "cluster_radius": 50,
            "container": "#map-goes-here"
        }
    }
}
```

| 옵션 | 뜻 | 기본값 |
|------|------|------|
| `latitude_column`, `longitude_column` | 이름 규칙에 맞지 않을 때 쓸 컬럼 | 없음 |
| `style` | MapLibre 스타일 URL 또는 스타일 JSON 객체 | OpenFreeMap Liberty (`https://tiles.openfreemap.org/styles/liberty`) |
| `view` | `clusters`, `points`, `heatmap` | `clusters` |
| `color_column` | 처음에 색칠 기준으로 쓸 컬럼 | 없음 |
| `height` | 개요 지도의 CSS 높이 | `500px` |
| `scroll_zoom` | `true`면 스크롤 휠만으로 확대 | 꺼짐 |
| `max_cell_maps` | 동시에 인터랙티브 지도가 될 수 있는 GeoJSON 셀 수 | 8 |
| `cluster_radius` | 한 클러스터로 묶이는 점 사이 거리(픽셀) | 25 |
| `container` | 테이블 위 대신 지도를 넣을 요소의 CSS 선택자 | 없음 |

`cluster_radius`는 값을 줄이면 작은 클러스터가 더 많이 생긴다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"cluster_radius - how close points have to be, in pixels, to be merged into one cluster. Default 25. Smaller values give more, smaller clusters."</blockquote></details>

개요 지도의 MapLibre 지도 객체는 `datasette.maplibre.map`으로 다른 JavaScript에서 꺼내 쓸 수 있다.

## 데모로 먼저 보기

설치하기 전에 브라우저에서 볼 수 있다. Simon은 Claude에게 데모까지 만들게 했고 이 데모는 최신 Datasette Lite에서 돈다. [Datasette Lite 데모 링크](https://lite.datasette.io/?install=datasette-maplibre&url=https://gist.githubusercontent.com/simonw/39fa5d399465766039549912e2b3d291/raw/f6dfba99a75cbb8b5d9386223220eb52272767ee/demo.db&metadata=https://gist.githubusercontent.com/simonw/39fa5d399465766039549912e2b3d291/raw/f6dfba99a75cbb8b5d9386223220eb52272767ee/metadata.yml){:target="_blank"}를 열면 Pyodide와 WebAssembly로 플러그인과 데모 데이터베이스가 브라우저 안에 올라온다.

데모 첫 화면은 기능 하나씩을 보여 주는 링크 19개로 된 안내 투어다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Its home page is a guided tour of every feature: 19 links, each opening a map set up to show one thing. It uses 35,000 power plants, a month of earthquakes, country boundaries, rivers, capital cities and landmarks with photographs."</blockquote></details>

같은 데모를 로컬에서 돌리려면 저장소를 받아 `demo/` 디렉터리를 쓴다.

```bash
uv run python demo/build.py
uv run datasette demo/demo.db -m demo/metadata.yml -c demo/datasette.yml --setting max_returned_rows 5000
```

`build.py`는 처음 실행할 때 공개 데이터 약 14MB를 `demo/cache/`에 받아 `demo/demo.db`를 만든다. `--refresh`를 붙이면 데이터를 다시 받는다. 최근 지진 기록으로 갱신하고 싶을 때 쓴다.

## 프롬프트 하나로 만든 과정

에이전트로 도구를 만드는 사람에게는 이 플러그인이 만들어진 과정도 볼 만하다. Simon은 대부분을 Claude Fable 5.2에 보낸 프롬프트 하나로 만들었다고 썼다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"I built most of this with a single prompt to Claude Fable 5.2"</blockquote></details>

프롬프트 전문은 이렇다.

> Look at datasette-cluster-map and datasette-leaflet-geojson plugins, then look at ~/dev/tools/openfreemap-demo.html - then build a brand new Datasette plugin that uses maplibreGL and openfreemap and combines the features of those other two plugins - but be creative and think outside the box for other useful features the plugin might have

이 프롬프트는 세 부분으로 읽힌다. 먼저 참고할 기존 코드를 짚어 준다. 플러그인 두 개와 로컬에 있던 OpenFreeMap 데모 HTML 파일이다. 다음으로 만들 것을 한 문장으로 정한다. 두 플러그인의 기능을 합친 새 플러그인이다. 끝으로 여지를 준다. 쓸모 있을 다른 기능을 틀 밖에서 생각해 보라는 주문이다. Simon은 이 마지막 문구에서 괜찮은 아이디어가 몇 개 나왔다고 했고 앞에서 본 SVG 윤곽 후 WebGL 전환이 그중 하나다.

"대부분"이라는 표현은 그대로 받아들여야 한다. 프롬프트 한 번에 지금의 0.1a0이 나왔다는 말은 아니다. 저장소 커밋 기록에는 데모 구성, Datasette 1.0 알파 대응, 클러스터 표시 조정 같은 후속 커밋이 이어지고 그 다수에 `Co-Authored-By: Claude Fable 5.2` 꼬리표가 붙어 있다. 그 커밋들을 어떤 프롬프트로 만들었는지는 릴리스 글에 나오지 않는다.

같은 저자의 다른 릴리스를 다룬 [llm-gemini 0.37 글]({{site.baseurl}}/tools/2026/11/13/llm-gemini-037-gemini-4-hydrogen.html)도 있다.

## 쓰기 전에 확인할 것

- 버전이 0.1a0, 곧 알파다. PyPI에 올라온 릴리스도 이것 하나뿐이라 옵션 이름과 동작이 바뀔 수 있다.
- Datasette 1.0a43 이상이 필요하다. 1.0 자체가 아직 알파 계열이므로 0.x로 운영 중인 인스턴스에는 맞지 않는다.
- 기본 설정에서는 스타일·타일·글꼴을 `tiles.openfreemap.org`에서 받는다. 사내망처럼 외부 접속이 막힌 곳에서는 `style`에 접근 가능한 스타일을 지정해야 지도가 그려진다.
- 라이선스는 Apache-2.0이다.

## 참고

- 원문: [Simon Willison's Weblog - datasette-maplibre 0.1a0](https://simonwillison.net/2026/Nov/15/datasette-maplibre/){:target="_blank"} (2026-11-15)
- 저장소: [datasette/datasette-maplibre](https://github.com/datasette/datasette-maplibre){:target="_blank"}
- PyPI: [datasette-maplibre](https://pypi.org/project/datasette-maplibre/){:target="_blank"}
- [OpenFreeMap](https://openfreemap.org/){:target="_blank"}
- [MapLibre GL](https://maplibre.org/){:target="_blank"}
