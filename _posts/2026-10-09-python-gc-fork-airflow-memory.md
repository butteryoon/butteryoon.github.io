---
layout: post
comments: true
title: "Python GC가 fork를 만나면 — Airflow 메모리 증가와 MySQL 커넥션 끊김의 공통 원인"
description: "NAVER D2 글 정리. fork로 만든 자식에서 순환 GC가 승계받은 객체의 PyGC_Head에 써서 COW를 일으키고, 자식의 GC가 부모의 MySQL 커넥션까지 닫는다. gc.freeze()로 두 문제를 함께 풀어 LocalExecutor PSS 58%를 줄인 과정."
img: python-gc-fork-airflow_title.webp
date: 2026-10-09 20:20:00 +0900
last_modified_at: 2026-10-09 20:20:00 +0900
tags: [naver-d2, python, gc, gc-freeze, multiprocessing, fork, copy-on-write, airflow, mysql, memray, sysmon]
related: sysmon
categories: dev
source_url: https://d2.naver.com/helloworld/4149925
source_date: 2026-10-08
---

Airflow 3를 PoC하던 NAVER WEBTOON 데이터 입수 플랫폼 팀이 증상 두 개를 만났다. worker 프로세스 메모리가 시간이 갈수록 늘었고 MySQL을 메타데이터 DB로 쓰는 환경에서는 dag-processor가 이따금 재시작됐다. 겉보기엔 상관없어 보이는 이 둘의 뿌리가 같았다. fork로 만든 자식 프로세스에서 Python 순환 GC가 부모에게 물려받은 객체를 건드린 것이다. 도정우 님이 NAVER D2에 올린 [「Python의 GC가 멀티프로세싱과 만나면 생기는 일」](https://d2.naver.com/helloworld/4149925){:target="_blank"}(2026-10-08)을 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** fork 직후 자식은 부모의 GC 추적 객체를 그대로 물려받는다. 자식에서 순환 GC가 돌면 그 객체들의 `PyGC_Head`에 쓰기가 일어나 페이지가 복사(COW)되고 PSS가 오른다. 같은 GC가 부모의 SQLAlchemy 커넥션 레코드를 수거하면 mysqlclient가 `COM_QUIT`을 보내 부모 커넥션이 끊긴다. fork 직전에 `gc.freeze()`를 부르면 두 문제가 함께 사라진다. LocalExecutor는 컨테이너 PSS 합계가 약 3.6GB에서 약 1.5GB로(58%), Celery worker 컨테이너는 2.25GB에서 1.07GB로 줄었다. 단, fork가 잦은 dag-processor에서는 freeze/unfreeze를 반복하면 오히려 부모 메모리가 샌다.

## 원문 정보

| 항목 | 값 |
|------|------|
| 원문 | [Python의 GC가 멀티프로세싱과 만나면 생기는 일 — Python과 Airflow, 그리고 관련된 문제 해결기 2편](https://d2.naver.com/helloworld/4149925){:target="_blank"} |
| 작성자 | 도정우 (NAVER WEBTOON, 데이터 입수 플랫폼 운영) |
| 발행일 | 2026-10-08 |
| 이전 편 | [1편: Python의 멀티프로세싱과 Airflow의 task 동작 방식](https://d2.naver.com/helloworld/4452165){:target="_blank"} |

1편은 Airflow가 task 실행용 프로세스를 모두 fork로 만든다는 점, 그리고 fork가 spawn보다 빠르고 가볍지만 부모 자원을 그대로 물려받고 Python에서는 COW가 기대만큼 동작하지 않는다는 한계를 짚었다. 2편은 그 한계가 실제 장애로 드러난 과정을 다룬다.

## 증상 1: worker 메모리가 계속 오른다

Airflow 3.1 계열 CeleryExecutor PoC에서 worker 컨테이너 메모리가 시간이 지날수록 늘었다. 구조가 같은 LocalExecutor부터 조사했다.

Memray로 worker 하나를 추적하자 힙 문제가 두 건 나왔다. secrets masker가 타입 확인용으로 `kubernetes.client` 전체를 import해 worker당 약 32MB(32개면 1GB 가까이)를 쓰고 있었고 Task SDK client는 SSL context를 반복 생성하며 새고 있었다. 둘 다 이슈로 올려 3.1.2에서 고쳐졌다. 팀은 이 김에 주요 컴포넌트를 코드 수정 없이 Memray로 띄우는 기능과 [Memory Profiling with Memray](https://airflow.apache.org/docs/apache-airflow/stable/howto/memory-profiling.html){:target="_blank"} 문서도 Airflow에 넣었다.

그래도 메모리는 계속 올랐고 이번 증가는 Memray에 잡히지 않았다. RSS는 거의 그대로인데 PSS와 USS만 올랐다. 프로세스가 건드리는 페이지 총량은 같고 부모와 공유하던 페이지가 자식 전용 복사본으로 바뀌고 있다는 뜻이다. `pmap`으로 영역별 PSS를 비교하니 부모와 같은 주소의 매핑에서 PSS가 늘고 있었다.

## 증상 2: MySQL 환경에서 dag-processor가 재시작된다

dag-processor 로그에는 `(2013, 'Lost connection to server during query')`가 남았다. 메인 루프가 우선 파싱 요청을 조회하는 지점은 재시도로 감싸여 있지 않아 프로세스가 그대로 죽었다. MySQL general log를 보니 그 시각에 해당 커넥션으로 `Quit` 명령이 들어와 있었다. 서버 타임아웃이 아니라 클라이언트가 정상 종료 절차를 밟은 것인데, 메인 프로세스는 그 커넥션을 풀에서 계속 쓰고 있었다.

원인을 좁히려고 SQLAlchemy Engine의 `connect` 이벤트에 훅을 걸어 커넥션 객체에 `weakref.finalize`를 붙였다. 소멸 로그는 파싱용으로 fork된 서브 프로세스(PID 417)에서 찍혔고 시각이 MySQL 로그의 `Quit`과 정확히 맞았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"mysqlclient 드라이버는 커넥션 객체가 해제될 때 서버에 COM_QUIT을 명시적으로 전송합니다. 자식이 보낸 Quit으로 서버는 세션을 종료하고, 부모는 다음 쿼리에서 2013 오류를 받습니다."</blockquote></details>

## 공통 원인: 순환 GC와 fork

CPython은 메모리를 두 단계로 회수한다. 참조 카운트가 0이 되면 객체를 즉시 해제하고 순환 참조처럼 카운트가 0이 되지 않는 경우는 순환 GC가 맡는다. 순환 GC는 컨테이너 객체에만 `PyGC_Head` 헤더를 붙여 추적하고 객체를 세대 0·1·2로 나눠 세대별 카운터가 임계값(기본 700, 10, 10)을 넘을 때 자동으로 수집한다. 언제 돌지는 프로그램이 정하지 않는다.

**COW가 생기는 이유.** 수집 때 GC는 대상 세대의 모든 추적 객체를 돌며 `gc_refs`라는 임시 값을 `PyGC_Head`에 쓰고 다시 돌며 그 값을 줄인다. fork 직후 자식은 부모가 만든 수십만 개의 추적 객체를 물려받으므로, 자식에서 GC가 한 번 돌면 그 객체들이 놓인 페이지에 쓰기가 일어난다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"수집이 일어날 때마다 GC는 부모로부터 승계받은 객체를 포함한 모든 추적 객체의 PyGC_Head에 쓰기를 수행할 수 있습니다. 자식이 그 객체를 쓰는지와는 무관합니다."</blockquote></details>

fork는 물리 프레임을 복사하지 않고 페이지 테이블만 복사한 뒤 양쪽 PTE를 쓰기 금지로 바꾼다. 어느 쪽이든 쓰기를 시도하면 page fault가 나고 그 프레임을 둘 이상이 참조하고 있으면 커널이 새 프레임으로 복사한다. 부모와 자식 중 누가 원본인지는 따로 없고 먼저 쓴 쪽이 복사본을 받는다.

저자는 `/proc/<pid>/pagemap`으로 이를 직접 확인했다. 부모가 GC 추적 객체 30만 개를 만들고 서로 다른 페이지에 있는 객체 20개의 주소를 기록한 뒤 fork한다. 자식은 객체에 전혀 접근하지 않고 `gc.collect()`만 한 번 부른다. 그 결과 자식의 PFN만 바뀌고 부모는 그대로였다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"자식은 이 객체들을 한 번도 건드리지 않았고 수거된 객체도 0개였지만, 순환 GC가 실행된 것만으로 20페이지 전부가 복사되었습니다."</blockquote></details>

**커넥션이 끊기는 이유.** Airflow는 `os.register_at_fork(after_in_child=...)`로 자식이 시작될 때 `engine.dispose(close=False)`를 호출해 새 커넥션 풀을 만든다. SQLAlchemy도 [fork와 함께 쓸 때 권장하는 방법](https://docs.sqlalchemy.org/en/20/core/pooling.html#using-connection-pools-with-multiprocessing-or-os-fork){:target="_blank"}이고 그 자체는 맞는 조치다. 문제는 버려진 옛 풀과 커넥션 레코드가 서로를 참조한다는 점이다. 참조 카운트로는 해제되지 않으니 자식에서 순환 GC가 돌아야 수거되고, 그 순간 mysqlclient 커넥션 객체가 해제되며 부모와 공유하는 소켓으로 `COM_QUIT`이 나간다. dynamic DAG generation처럼 파싱이 오래 걸리면 GC가 돌 확률이 높아지고 소규모 환경에서는 자식이 임계값에 닿기 전에 끝나 버린다. 재현 여부가 규모에 따라 갈린 이유다.

## 해법: gc.freeze()

처음엔 자식이 물려받는 객체 자체를 줄이려 했다. scheduler 루프 전에 worker를 먼저 만들고 무거운 import를 fork 뒤로 미뤘다. 하지만 `airflow` CLI 기동만으로 SQLAlchemy·FastAPI 같은 라이브러리가 올라와 100MB를 차지해 줄일 여지가 적었고, task마다 모듈을 다시 import하느라 처리량이 분당 100개 넘던 수준에서 60~70개로 떨어졌다. 자식에서 `gc.disable()`로 GC를 끄는 방법도 있지만 그러면 자식이 만드는 순환 참조가 수거되지 않아 다른 누수가 생긴다.

결국 택한 것은 Python 3.7부터 들어간 [`gc.freeze()`](https://docs.python.org/3/library/gc.html#gc.freeze){:target="_blank"}다. 호출 시점에 GC가 추적하는 모든 객체를 영구 세대로 옮기고 이후 수집은 이 세대를 건너뛴다. Instagram 엔지니어들이 fork 기반 웹 서버에서 같은 COW 문제를 겪고 CPython에 기여한 기능이다. fork 직전에 부르면 승계 객체의 `PyGC_Head`가 바뀌지 않으니 COW가 없고 부모 객체가 수거되지 않으니 커넥션도 닫히지 않는다. 영구 세대로 간 객체도 참조 카운트로는 여전히 해제될 수 있다. 같은 세대 객체는 이중 연결 리스트로 이어져 있어서 freeze는 리스트 세 개를 영구 세대 리스트 뒤에 붙이는 O(1) 작업이다.

### 컴포넌트별 적용

**LocalExecutor ([PR #58365](https://github.com/apache/airflow/pull/58365){:target="_blank"}, 3.1.4 백포트).** scheduler 자신은 계속 객체를 만들고 버리므로 freeze된 채로 두면 안 된다. 그래서 worker 생성 구간만 감싼다.

```python
def _spawn_workers(self, n: int):
    gc.freeze()
    try:
        for _ in range(n):
            self._spawn_worker()
    finally:
        gc.unfreeze()
```

자식은 물려받은 객체가 모두 영구 세대에 있는 상태로 시작하고 task 실행 중 만든 객체만 GC 대상이 된다. unfreeze한 부모 쪽에서는 COW가 생기지만 자식 수만큼 일어나던 복사가 부모에서 한 번으로 줄어든다. 실측한 freeze/unfreeze 비용은 각각 19μs, 10μs 정도였다.

**dag-processor ([PR #60505](https://github.com/apache/airflow/pull/60505){:target="_blank"}).** 같은 사이클을 적용했더니 오히려 부모 메모리가 계속 늘었다. `gc.freeze()`는 세대별 수집 카운터를 0으로 되돌리는데, dag-processor는 파일마다 fork하므로 카운터가 임계값에 닿기 전에 계속 리셋된다. 그 사이 부모가 만든 순환 참조 가비지는 unfreeze 때 세대 2로 옮겨지고 세대 2를 훑는 full collection은 트리거가 오지 않아 영영 돌지 않는다. freeze 직전에 `gc.collect()`를 강제하면 풀리지만 fork마다 full collection을 도는 비용이 커서 채택하지 않았다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"대신 파싱 루프에 진입하기 직전에 한 번만 freeze하고 unfreeze는 하지 않았습니다. 막아야 할 COW는 import airflow와 초기화 과정에서 만들어진 객체들의 것이고, 이들은 루프 전에 이미 존재하기 때문입니다."</blockquote></details>

**CeleryExecutor ([PR #62212](https://github.com/apache/airflow/pull/62212){:target="_blank"}).** Celery worker 메인 프로세스는 `worker_concurrency`(기본 16)만큼 ForkPoolWorker를 fork해 상주시킨다. 무거운 모듈 preload가 끝나는 `celery_import_modules` 시그널에서 `gc.freeze()`를 부르고 worker 초기화가 끝나는 `@worker_ready.connect`에서 `gc.unfreeze()`를 부른다.

## 측정 결과

| 컴포넌트 | 측정 조건 | 적용 전 | 적용 후 |
|----------|-----------|---------|---------|
| LocalExecutor | 분당 500 task, 12시간, worker 32개 | worker당 100MB 이상, 컨테이너 PSS 합계 약 3.6GB | worker당 약 40MB, 합계 약 1.5GB (58% 감소) |
| dag-processor | dynamic DAG 파일 5개, 파일당 평균 DAG 40개 | 메모리 스파이크, Pod 재시작 | 스파이크 감소, 재시작 해결 |
| CeleryExecutor | worker 컨테이너 1개 | 2.25GB | 1.07GB |

<details class="evidence"><summary>원문 근거</summary><blockquote>"worker 32개를 띄운 scheduler 컨테이너 전체로 보면 PSS 합계가 약 3.6GB에서 약 1.5GB로, 약 2.1GB(58%)가 줄었습니다."</blockquote></details>

세 변경은 각각 Airflow 3.1.4, 3.1.7, Celery provider 3.17.2에 들어 있다. 저자는 이 밖에도 pathlib `sys.intern` 메모리 증가(#65706), LocalExecutor 파일 디스크립터 락 누수(#65121) 등을 고쳤고, 이 변경이 모두 반영된 3.3.0에서는 자기 환경 기준으로 worker 내부 누수를 더 찾지 못했다고 적었다.

## 정리와 적용 포인트

원문도 강조하듯 이 문제는 Airflow 고유의 것이 아니다. gunicorn의 preload처럼 무거운 모듈을 먼저 올리고 fork로 자식을 만드는 Python 서비스라면 같은 패턴이 나올 수 있다. 진단 순서로 보면 쓸 만한 단서가 셋이다.

- **RSS는 그대로인데 PSS·USS만 오르면** 새 할당이 아니라 COW다. Memray 같은 힙 추적기에는 보이지 않으니 `pmap`이나 `/proc/<pid>/pagemap`으로 공유 페이지가 깨지는지 봐야 한다.
- **DB가 클라이언트 측 `Quit`을 기록했는데 부모는 커넥션을 쓰고 있었다면** fork된 자식의 GC가 승계 커넥션을 수거했을 수 있다. `engine.dispose(close=False)`만으로는 막히지 않는다.
- **`gc.freeze()`는 fork 빈도에 맞춰 쓴다.** 가끔 fork하는 프로세스는 freeze/unfreeze로 감싸고 fork가 GC 임계값 도달보다 잦은 프로세스는 루프 진입 전에 한 번만 freeze하고 unfreeze하지 않는다.

예전에 정리한 [sar를 이용한 리눅스 시스템 모니터링]({{site.baseurl}}/dev/2018/10/14/monitoring.html)의 top·sar는 RSS나 시스템 전체 사용량을 보여 준다. 공유 페이지가 하나씩 깨지며 늘어나는 이번 같은 증가는 프로세스별 PSS를 따로 봐야 잡힌다.

## 참고 자료

- [NAVER D2 원문 — Python의 GC가 멀티프로세싱과 만나면 생기는 일](https://d2.naver.com/helloworld/4149925){:target="_blank"} (2026-10-08)
- [1편: Python의 멀티프로세싱과 Airflow의 task 동작 방식](https://d2.naver.com/helloworld/4452165){:target="_blank"}
- [Instagram Engineering: Copy-on-Write Friendly Python Garbage Collection](https://medium.com/instagram-engineering/copy-on-write-friendly-python-garbage-collection-ad6ed5233ddf){:target="_blank"}
- [Python 공식 문서: gc.freeze()](https://docs.python.org/3/library/gc.html#gc.freeze){:target="_blank"}
- [SQLAlchemy: Using Connection Pools with Multiprocessing or os.fork()](https://docs.sqlalchemy.org/en/20/core/pooling.html#using-connection-pools-with-multiprocessing-or-os-fork){:target="_blank"}
- Airflow PR: [#58365](https://github.com/apache/airflow/pull/58365){:target="_blank"} · [#60505](https://github.com/apache/airflow/pull/60505){:target="_blank"} · [#62212](https://github.com/apache/airflow/pull/62212){:target="_blank"}
- [Memray](https://bloomberg.github.io/memray/){:target="_blank"} · [Airflow: Memory Profiling with Memray](https://airflow.apache.org/docs/apache-airflow/stable/howto/memory-profiling.html){:target="_blank"}
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/ZIPFteu-R8k){:target="_blank"}
