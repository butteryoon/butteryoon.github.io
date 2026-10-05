---
layout: post
comments: true
title: "Claude-shaped science: Schwartz가 Claude의 강점에 맞춰 푼 18개 분야 36편의 원고"
description: "Matthew Schwartz가 Claude를 '원하는 협력자'가 아닌 '실제 협력자'로 대우하며 만든 BootLoops 하네스와, 이를 통해 3개월간 18개 분야에서 36편의 원고를 쓴 경험을 정리한다."
img: claude-shaped-science_title.webp
date: 2026-10-05 18:00:00 +0900
last_modified_at: 2026-10-05 20:00:00 +0900
tags: [anthropic, claude, ai-science, bootloops, research, llm, llm-science]
related: llm-science
categories: dev
source_url: https://www.anthropic.com/research/claude-shaped-science
source_date: 2026-10-01
---

Anthropic 방문 연구원인 Matthew Schwartz 교수가 Claude를 연구 협력자로 쓰면서 접근법을 바꾼 과정을 에세이로 풀었다. 사람 과학자처럼 일하게 만들려 애쓰는 대신, Claude가 잘하는 일에 맞는 문제('Claude-shaped' 문제)를 찾아 맡겼더니 BootLoops라는 정확 계산 툴킷이 나왔다. 이 글은 그 원문을 요약하고 시사점을 덧붙인다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Schwartz는 Claude의 강점(폭넓은 지식, 코딩, 수학·통계, 문서 대량 파싱)에 맞는 문제를 골라 **BootLoops**라는 오픈소스 하네스를 만들었고, 3개월간 18개 분야에서 19명의 공저자와 36편의 원고를 냈다. 핵심은 AI가 잘하는 일과 과학자가 원하는 일 사이의 **임피던스 미스매치**를 하네스로 메우고, 기술적으로는 맞지만 과학적으로는 평범한 결과를 도메인 전문가와 함께 의미 있는 과학으로 다듬는 방식이다.

## 1. 원문 개요

- **출처:** Anthropic Research — [Claude-shaped science](https://www.anthropic.com/research/claude-shaped-science){:target="_blank"} (2026-10-01)
- **저자:** Matthew Schwartz 교수. 이 프로젝트 기간에 Anthropic 방문 연구원으로 일했고, BootLoops는 Anthropic 프로젝트가 아니라 Schwartz 본인이 소유·관리한다고 원문에 밝혀 두었다.
- **형식:** 장문 에세이 (게스트 포스트)

## 2. 배경: 왜 'Claude-shaped'인가

Schwartz는 2025년 12월에 Claude Opus 4.5를 연구 보조로 써 보았다. 모델은 강한 대학원생 수준의 일을 20배 빠르게 해냈지만, 문장마다 고치고 엉뚱한 갈래에서 끌어내고 막다른 길에서 되돌려야 했다. 결과물은 좋았어도 과정은 고역이었다고 한다.

2026년 여름 Claude Fable 5가 나온 뒤에는 접근을 바꿨다. 자신이 바라는 협력자가 아니라 Claude가 실제로 어떤 협력자인지에 맞춰 문제를 찾기로 한 것이다. 원문이 꼽은 Claude의 강점은 다음과 같다.

- 모든 분야에 걸친 사실상 무제한의 지식
- 뛰어난 코딩 실력
- 최신 수학·통계 지식
- 논문, 부록, 데이터를 기계 속도로 읽어 내는 능력

반대로 깊은 개념적 질문에는 아직 도움이 안 된다고 적었다.

## 3. BootLoops의 탄생

### 3.1 반수치적 부트스트랩에서 출발

산란 진폭을 계산하려면 파인만 적분을 풀어야 하는데, 지금 다루는 난이도의 적분은 하나가 박사 논문 한 편 분량이다. S-matrix 부트스트랩은 적분을 직접 계산하는 대신 물리적 제약을 걸어 답을 하나로 좁힌다. 가장 대칭성이 높은 N=4 초양-밀스 이론의 9루프 진폭이 이 방식으로 끝까지 간 예다. 반수치적 부트스트랩은 여기에 몇 개 점에서 극도로 정밀한(때로는 1,000자리) 수치 계산을 더해 남은 계수를 확정한다. Schwartz는 이 방법이 수학·물리·컴퓨터과학 지식이 한 사람에게 모여 있지 않고, 코딩량이 많고, 정답을 검증할 수 있다는 점에서 에이전트형 AI에 잘 맞는다고 보았다.

첫 과제는 Wolfram Language, C++, Python, Julia로 흩어진 기존 구현과 코드가 공개되지 않은 방법을 공통 프레임워크로 옮기는 일이었다. Claude는 Schwartz 본인 논문의 결과를 약 20분 만에 재현했다. 같은 일에 그가 짠 코드는 몇 주가 걸렸다.

### 3.2 타원 적분으로 확장

Claude가 가장 단순한 함수족인 로그 함수에만 머무르는 것을 보고, Schwartz는 다음으로 단순한 타원 함수에 같은 방식을 적용해 보라고 했다. 타원 파인만 적분은 지금까지 몇 개만 계산됐고 부트스트랩으로 완전히 푼 사례는 없었다. 몇 주 만에 BootLoops는 30개의 적분을 끝까지 풀어냈다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Soon we had 30 integrals BootLooped from end to end, comprising 15 reproductions of known results by this new method and 15 that had never before been computed."</blockquote></details>

## 4. 'I know Kung Fu' 순간: 같은 방정식, 다른 분야

과학에서는 같은 방정식이 전혀 다른 맥락에서 되풀이해 나타난다. Claude는 BootLoops로 풀 수 있는 계산이 생태학, 인구유전학, 경제학, 언어학에도 있다는 것을 알아봤다. 원문은 이 시기를 영화 매트릭스에서 네오가 쿵후를 내려받는 장면에 비유한다.

### 4.1 생태학: 중립 생물다양성 이론

Rampal Etienne이 2005년에 Hubbell의 중립 이론을 방정식으로 옮겼지만, 20년간 아무도 이를 대규모로 풀지 못했다. Claude는 이 방정식을 BootLoops로 풀 수 있는 문제로 알아봤고, 파나마 바로 콜로라도 섬의 데이터에 적용해 나무 종 구성이 중립 이론이 허용하는 것보다 4.5배 빠르게 변한다는 결과를 얻었다. 그러나 식물생물학자 James O'Dwyer는 기술적으로는 인상적이지만 "많은 생태학자가 어깨를 으쓱하고 말 것"이라고 평했다. 생태학자들은 이미 중립 이론이 실제 숲을 따라가지 못한다는 것을 정성적으로 알고 있었기 때문이다. 대신 그는 중립 이론의 예측을 빼고 남은 부분을 보자고 제안했고, 이렇게 하면 자연선택과 경쟁, 종 간 차이에서 오는 변화를 가려낼 수 있다. 이 아이디어가 후속 모델로 이어졌다.

### 4.2 인구유전학

Claude는 수리물리의 방법을 가져와 자연선택이 드문 돌연변이에 미치는 영향을 나타내는 30년 된 적분식을 풀었고, 최대 공개 인간 유전변이 카탈로그인 gnomAD에 적용했다. 이번에도 전문가인 Michael Desai의 반응은 기술적으로는 인상적이나 과학적으로는 설득력이 부족하다는 것이었다. Desai의 제안으로 방향을 바꿔 1000 Genomes Project에서 가까이 있는 돌연변이 57억 쌍을 분석했고, 유전자 전환(gene conversion)의 증거를 찾았다. 연관된 유전 변이를 쓰는 거의 모든 분석이 이 메커니즘을 무시해 왔다는 점에서 의미가 있다고 원문은 설명한다.

### 4.3 경제학과 언어학

- **경제학:** 경제학자 두 명과 함께 AI 데이터 에디터를 만들었다. 주요 저널 5곳의 논문 4,452편에 딸린 재현 패키지를 MATLAB, Stata 같은 상용 도구에서 오픈소스 코드(약 3만 개 루틴)로 옮기고, 검증 가능한 수치는 사실상 전부 발표된 표와 대조했다. 결과는 NBER 워킹페이퍼로 나왔다.
- **언어학:** 언어학자 세 명과 함께 6,072개 언어의 단어 강세 데이터베이스 AccStack을 만들었다. 거의 모든 항목에 판단 근거가 된 구절을 인용했고, 음운론 문헌 16만 편의 서지도 함께 정리했다.

### 4.4 그 밖의 사례

- **계통발생학:** 진화 계통수의 베이즈 증거를 빠르고 정확히 검증 가능하게 계산했다. 유전자 하나만으로는 경쟁하는 계통수들이 표준 샘플링 프로그램의 오차보다 작은 차이로 비기는 경우가 많았다.
- **지구과학:** 대기과학, 지구화학, 기후 모델을 합쳐 대산화사건이 네 번의 빙하기를 거치며 전개된 과정을 예측하는 정량 모델을 만들었다.
- **유전체학:** 단일세포 RNA 카운트 통계로 유전자 발현의 버스팅 속도와 교과서적 텔레그래프 모델에서 벗어나는 정도를 분석했다.
- **태양 흑점:** 인구통계학의 방법으로 흑점의 생애주기를 도출하고, 외계행성 탐색의 교란 요인인 항성 흑점으로 확장했다.
- **수리물리:** George Watson이 1939년에 시작한 격자 적분 가운데 마지막 문제, 세 방향 도약률이 서로 다른 3차원 랜덤워크의 정확한 복귀 확률을 풀었다.
- **우주론:** 2-루프 파워 스펙트럼과 1-루프 트라이스펙트럼을 포함해 우주 거대구조 데이터로 우주론 매개변수를 맞추는 툴킷을 만들었다.
- **통계학:** 다루기 까다로운 혼합 모형의 베이즈 증거를 정밀하게 근사해 모델 선택을 실용적으로 만들었다.

원문은 이 사례들이 모두 전문가와의 협업으로 진행됐고 추가 탐색과 검증이 계속되고 있다고 덧붙인다.

## 5. 운영 방식

### 5.1 세션 구성

36편의 원고, 18개 분야, 19명의 공저자를 3개월간 약 400개 후보 문제 중에서 골라 진행했다. Claude Code 세션은 모두 Google Cloud 가상 머신의 터미널에서 돌리고, 협업자의 선호에 따라 GitHub 또는 Overleaf 저장소에 연결했다. 프로젝트마다 별도 세션을 두고, 이들을 조율하고 연산 자원을 배분하고 결과를 검증하는 마스터 세션을 따로 뒀다. 계산은 서브에이전트가 맡고 중간 결과는 마크다운 파일에 저장한다. Fable 5의 classifier에 걸려도 세션 전체가 아니라 에이전트 하나만 멈추는 이점이 있다고 한다.

장시간 프로젝트에서는 컴팩션(compaction) 때문에 중요한 맥락이 자주 사라졌다. 그래서 Claude가 주기적으로 파일을 정리·통합하게 하는 프로토콜을 BootLoops에 넣었다. 원문은 같은 문제 중 일부를 Claude Science 같은 상용 하네스가 이미 해결했다고 덧붙인다.

### 5.2 실패 모드와 대응

원문이 꼽은 실패 모드와 팁 가운데 눈에 띄는 것만 추렸다.

- **승리 선언이 잦다.** "한 가지 단서가 붙은 완료"는 대개 "전혀 끝나지 않았다"는 뜻이다. 성공 기준을 분명하고 엄격하게 줘야 한다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"Claude loves to declare victory. “Done, with one asterisk” is often “not done at all.”"</blockquote></details>

- **직접 확인한다.** 그래프를 늘 요청하고, 자동 검사도 완전히 믿지 않는다. '일치도가 좋다' 같은 정성적 주장에도 조심해야 한다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"Even with all the monitors I set up, I’ve found the automated checks still can’t be trusted."</blockquote></details>

- **취향은 사람이 공급한다.** Claude가 찾아내는 Claude-shaped 문제는 수천, 수백만 개에 이르고, 잊힌 지 오래된 고인용 논쟁을 고르는 경향이 있다. 무엇이 흥미로운지에 대한 판단은 아직 믿기 어렵다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"Claude seems to favor old debates, highly cited but long forgotten."</blockquote></details>

- **결론은 의심한다.** 계산은 잘해도 거기서 끌어내는 결론은 틀릴 수 있다. 믿을 수 있을 때까지 설명하게 한다.
- **더 나은 결과를 요구한다.** 좋은 결과가 나오면 더 좋은 결과를 요구한다. 가끔은 정말 훌륭한 것이 나온다.
- **시간 감각이 없다.** Claude의 소요 시간 추정은 끝내 믿을 수 없었고, Schwartz는 흥미로운 일이 대략 얼마나 걸려야 하는지 스스로 감을 익혔다. 며칠짜리 계산을 그대로 밀어붙이지 말고 같은 계산을 몇 분으로 줄여 줄 도구를 만들게 해야 했다.

  <details class="evidence"><summary>원문 근거</summary><blockquote>"Honestly, I never succeeded in getting Claude to estimate time well."</blockquote></details>

## 6. 전망: 볼록 껍질로서의 AI 과학

- 밀레니엄 문제나 대형 과학에 대한 집착은 '발등 찍기(footgun)'가 될 수 있다. 비현실적인 기대가 이미 가능한 생산적 용도를 위축시킬 수 있기 때문이다. 실제 진보는 현실의 데이터를 얻고, 이해하고, 다음 질문으로 넘어가는 과정에서 나온다고 Schwartz는 본다.
- **볼록 껍질 비유:** 지금의 과학은 들쭉날쭉한 프론티어로 이뤄져 있다. 한 연구실이 20년 동안 유전자 한 묶음을 한 가지 방법으로만 파고드는 동안, 옆 유전자와 다른 방법은 그대로 방치된다. 사람이 닿을 수는 있지만 아직 아무도 가지 않은 중간 지대가 AI에 맞는 영역이고, BootLoops는 그 틈을 메운다.
- 교육과 연구비 계획은 더 불투명해졌다. 3년짜리 연구비를 받아 풀려던 문제를 AI가 하룻밤에 풀 수 있다면 어떻게 해야 하는가. 2년 전만 해도 필수로 권했을 "엔지니어를 위한 파이썬" 수업이 이제는 필요 없어졌다는 말도 나온다.
- 기여에 대한 평가도 문제다. 기술적인 계산은 자동화돼도 개념적인 부분에는 여전히 사람이 필요하므로, 인간의 기여가 타이핑이었다고 가정해서는 안 된다는 것이 Schwartz의 입장이다.

## 7. 시사점

원문에서 개발 현장에 옮겨 볼 만한 대목을 정리하면 다음과 같다.

1. **하네스를 설계 대상으로 본다.** BootLoops, Claude Code, Claude Science, Codex는 모두 모델의 강점을 살리고 약점을 메우는 래퍼다. 에이전트 오케스트레이션 층을 명시적인 하네스로 설계하면 컴팩션 대응이나 도구 재사용을 체계적으로 다룰 수 있다.
2. **검증 지점을 사람이 쥔다.** Claude의 '완료' 보고는 그래프, 원자료, 엄격한 성공 기준으로 다시 확인해야 한다.
3. **진전 속도를 지표로 본다.** 시간 추정을 못 하는 모델에는 하드 타임아웃만 걸기보다, 합리적인 시간 안에 의미 있는 진전이 나오는지를 사람이 보는 편이 낫다.
4. **도구를 쌓는다.** 실패한 프로젝트에서 나온 도구도 다음 프로젝트의 문을 열었다. 공통 유틸리티를 버전 관리하고 문서화해 두면 에이전트 간 자산이 된다.

## 8. 참고 자료

- 원문: [Anthropic Research — Claude-shaped science](https://www.anthropic.com/research/claude-shaped-science){:target="_blank"} (2026-10-01)
- BootLoops: [bootloops.ai](https://bootloops.ai){:target="_blank"}, [GitHub](https://github.com/BootLoops-ai/bootloops){:target="_blank"}
- 선행 글: [Vibe Physics](https://www.anthropic.com/research/vibe-physics){:target="_blank"}
- 반수치적 부트스트랩 논문: [arXiv:2507.17815](https://arxiv.org/abs/2507.17815){:target="_blank"}
- AccStack: [accstack.org](http://www.accstack.org){:target="_blank"}
- 데이터: [gnomAD](https://gnomad.broadinstitute.org/){:target="_blank"}, [1000 Genomes Project](https://www.internationalgenome.org/about/){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/aTpgq-1PdrU){:target="_blank"}
