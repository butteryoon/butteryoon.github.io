---
layout: post
comments: true
title: "OpenAI·Cisco Shirudo: 패치 전까지 실행 중인 Linux를 eBPF로 막는 오픈소스 런타임"
description: "OpenAI와 Cisco가 공개한 오픈소스 런타임 Shirudo를 정리한다. 서명된 eBPF 프로그램으로 취약한 코드 경로를 재시작 없이 막고, TPM 증거로 대상 호스트에 무엇이 올라갔는지 검증하는 구조다. 라이선스, 사전 요구 사항, 공식 퀵스타트의 VM 실습 순서와 pre-release 제약을 함께 적는다."
img: openai-cisco-shirudo_title.webp
date: 2026-11-24 19:00:00 +0900
last_modified_at: 2026-11-24 19:00:00 +0900
tags: [llm, llm-safety, openai, cisco, shirudo, ebpf, linux, tpm, runtime-security, codex]
related: llm-safety
categories: dev
source_url: https://openai.com/index/introducing-shirudo
source_date: 2026-11-23
---

OpenAI와 Cisco가 Shirudo라는 오픈소스 런타임을 공개했다. 취약점이 알려진 뒤 패치를 시험하고 배포하기까지 걸리는 기간 동안, 실행 중인 Linux 시스템에 임시 방어(원문 용어로 shield)를 재시작 없이 올리는 도구다. 방어 코드를 에이전트가 만들고 사람이 승인해 배포하는 흐름을 전제로 설계됐다는 점에서 에이전트 운영 쪽에서도 볼 만하다. 코드와 문서, VM 퀵스타트가 이미 열려 있어 지금 받아서 돌려 볼 수 있다. (이 글의 초안은 Orca Bots 가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** Shirudo는 eBPF 기반의 Linux 보안 런타임이다. 서명된 eBPF 프로그램만 로드되게 하고 로드·언로드 내역을 TPM으로 측정해 운영자가 오프라인에서 검증할 수 있게 한다. 라이선스는 BSD 2-Clause이고 BPF 프로그램은 BSD/GPL 이중 라이선스다. 다만 저장소는 pre-release라고 밝히고 있고 실제 대상 호스트에는 signed-loader ABI와 Shirudo 커널 패치가 들어간 커널, TPM 2.0이 필요하다. 지금 해 볼 수 있는 것은 소프트웨어 TPM을 쓰는 로컬 VM 퀵스타트와 Isovalent의 브라우저 실습이다.

원문은 2026년 11월 23일 OpenAI 사이트에 올라온 [OpenAI and Cisco introduce Shirudo](https://openai.com/index/introducing-shirudo){:target="_blank"}이고 저자 표기는 OpenAI다. 이 글은 그 발표문에 더해 [저장소 README](https://github.com/shirudo-dev/shirudo){:target="_blank"}, [공식 퀵스타트 문서](https://docs.shirudo.dev/start/quickstart.html){:target="_blank"}, [Isovalent 블로그의 같은 날 발표문](https://isovalent.com/blog/post/cisco-openai-shirudo/){:target="_blank"}을 함께 읽고 정리했다. 퀵스타트 절차는 문서에 적힌 내용을 옮긴 것이고 개발 환경에서 직접 실행해 확인한 것은 아니다.

## 무엇을 푸는 도구인가

원문이 짚는 문제는 패치 공백이다. 패치를 시험하고 배포하는 데는 며칠에서 몇 주가 걸릴 수 있다. 그 사이에 서비스를 보호하기가 어려운 이유는 익스플로잇과 정상 요청이 같은 Linux 동작에 의존할 수 있어서다. 그 동작을 통째로 막으면 서비스가 내려간다.

Shirudo는 이 절충을 다루려고 만든 도구다. 임시 방어가 취약점을 드러내는 동작은 막으면서 정상 요청은 통과시키는지 먼저 시험하고 운영자가 승인하면 서비스가 도는 채로 배포한다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Once an operator approves the shield, it can be deployed while the service continues running."</blockquote></details>

원문이 드는 사례는 Java4Shell이다. Codex로 만든 shield가 Java의 Base64 디코더에 입력 검사를 추가했고 이를 실행 중인 Elasticsearch 프로세스에 붙였다. 익스플로잇을 시도하자 shield가 악성 입력을 거부했고 Elasticsearch는 계속 돌았다. 원문은 이를 통제된 시험(controlled test)이라고 표현한다. Isovalent 글에 따르면 이 사례의 방어는 eBPF가 아니라 Java 바이트코드 형태로 만들어졌다. 즉 Shirudo가 다루는 방어는 커널 훅에 붙는 eBPF 프로그램과 JVM에 붙는 Java 에이전트 두 종류다.

## 네 가지 설계 축

원문은 Shirudo를 네 가지 기능 중심으로 설계하고 있다고 적는다.

- **Live protection**: eBPF shield가 시스템을 재시작하지 않고 취약한 코드 경로를 막는다. 적용하기 전에 제안된 shield가 애플리케이션에 어떤 영향을 줄지 모니터링할 수 있다.
- **Defense at machine speed**: 에이전트가 취약점을 조사하고, 대상을 좁힌 shield를 개발하고, 익스플로잇과 실제 워크로드에 대해 효과를 검증할 수 있도록 프로그래밍 가능한 제어 수단을 준다. 무엇을 배포할지는 보안팀이 정한다.
- **Trusted deployment**: shield는 암호학적으로 서명되고 로드 전에 검증된다. Linux의 eBPF verifier가 프로그램을 안전 규칙에 대조하고 커널 보안 통제가 허가되지 않은 변경을 제한한다.
- **Hardware-backed verification**: TPM으로 방어가 로드되거나 제거된 기록을 검증 가능하게 남긴다. 운영자는 이 증거로 시스템이 보안 요구 사항을 충족하는지 확인할 수 있다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Shields are cryptographically signed and verified before loading."</blockquote></details>

에이전트 관점에서 눈에 띄는 것은 두 번째 축이다. Isovalent 글은 Shirudo가 2022년에 오픈소스로 공개된 eBPF 보안 플랫폼 Tetragon에 뿌리를 두고 있고 에이전트용으로 다시 쓴 것이라고 설명한다. 사람을 위해 설계한 정책 언어의 범위에 에이전트를 가두지 않고 에이전트가 코드로 직접 방어를 작성하게 한다는 것이 그 글의 논지다. 대신 서명, eBPF verifier, 운영자 승인이 안전장치 역할을 한다.

<details class="evidence"><summary>원문 근거 (Isovalent)</summary><blockquote>"Shirudo has its roots in Tetragon, rewritten for the agentic era."</blockquote></details>

## 저장소가 말하는 실제 상태

발표문은 shield라는 말을 쓰지만 저장소와 문서의 용어는 upatch(서명된 마이크로패치)다. README가 밝히는 구조는 다음과 같다.

- 부팅 시 초기화가 끝나면 사용자 공간의 BPF 프로그램 로드를 서명된 로더로 제한한다.
- upatch의 로드와 언로드는 커널이 봉인된 `.bpf` 키링에 대조해 검증한 로더를 거친다.
- 로드된 upatch 집합이 바뀌면 부팅 단위의 TPM 레지스터가 확장되고 `shirudo attest`가 운영자가 오프라인에서 검증할 수 있는 증거를 돌려준다.
- 배포는 pull 방식이다. 대상 호스트가 OCI 레지스트리에서 패키지를 가져오고 텔레메트리는 설정된 OpenTelemetry 컬렉터로 내보낸다.
- Java upatch는 사용자 공간에서 검증하고 설정된 JVM이 시작할 때 에이전트를 로드한다.

README가 꼽는 용도는 런타임 보안·관측(`kernel-obs` 패키지), 라이브 완화, 시스템 하드닝, 임베디드·엣지 기기다. 하드닝 항목에는 컨테이너, Kubernetes Pod와 함께 LLM 에이전트에 대한 정책 강제가 들어 있다.

도입을 검토한다면 먼저 읽어야 할 제약은 이쪽이다.

- **pre-release다.** 실제 대상 호스트에는 signed-loader ABI와 Shirudo 커널 패치가 들어간 Linux 커널, 그리고 TPM 2.0이 필요하다. 배포판 기본 커널에 바로 올리는 도구가 아니다.
- **아키텍처**: CI는 x86-64에서 전체 테스트 단계를, arm64에서 호스트 테스트를 돌린다. 테스트 VM은 arm64에서도 로컬 부팅된다. riscv64 대상 검증은 보류 상태다.
- **릴리스**: `main`이 개발 브랜치이고 개발 패키지는 Build packages 워크플로의 실행 아티팩트로 받는다. 릴리스 태그는 GitHub Releases에 패키지를 올린다.

<details class="evidence"><summary>원문 근거 (README)</summary><blockquote>"shirudo is pre-release. Production targets require a Linux kernel with the signed-loader ABI and shirudo's kernel patches, plus a TPM 2.0."</blockquote></details>

라이선스는 BSD 2-Clause이고 BPF 프로그램은 BSD와 GPL 이중 라이선스다. README는 일부 서드파티 소스 파일이 호환되는 다른 라이선스로 포함돼 있고 파일마다 SPDX 식별자가 있다고 덧붙인다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"Shirudo is licensed under BSD 2-Clause, with its BPF programs dual-licensed under BSD and GPL."</blockquote></details>

## 퀵스타트로 확인할 수 있는 것

공식 퀵스타트는 체크아웃에서 Shirudo를 빌드하고 보호된 테스트 대상을 VM으로 띄운 뒤 upatch 하나를 서명, 로드, TPM 검증, 언로드까지 한 바퀴 돌리는 튜토리얼이다. 호스트가 운영자(operator) 역할을, 테스트 VM이 대상(target) 역할을 한다. VM은 소프트웨어 TPM(swtpm)을 쓴다.

사전 요구 사항 중 걸리기 쉬운 것만 추렸다.

- KVM이 되는 x86-64 또는 arm64 Linux 호스트. `/dev/kvm`을 읽고 쓸 수 있어야 한다. Mac에서는 Lima VM을 Linux 호스트로 쓰라고 안내한다.
- locked-memory 한도가 unlimited여야 한다(`ulimit -l`). Shirudo 바이너리는 메모리를 잠그며 이 조건이 안 맞으면 실행을 거부한다. root 권한은 필요 없다.
- 빌드에는 BPF 타깃을 포함한 clang·LLVM 23, OpenSSL 3.5 이상, libelf, zlib, libcurl 7.75 이상, tpm2-tss, LuaJIT 개발 파일이 필요하다.
- 테스트 VM에는 QEMU, swtpm, tpm2-tools, dracut 등이 필요하고 VM 스크립트는 빠진 명령이 있으면 거기서 멈추고 설치 명령을 출력한다.
- 검증 단계에서 `jq`와 `openssl` 명령을 쓴다.

순서는 문서 기준으로 아래와 같다. `operator$`는 호스트, `target#`은 VM 안의 root 셸이다.

```bash
# 1) CI 커널 아티팩트를 받아 준비한다 (SHA-256 확인 후 압축 해제)
operator$ testing/vm/kernel-artifact.sh fetch

# 2) 바이너리와 운영자용 헬퍼를 빌드하고 PATH에 넣는다
operator$ make
operator$ make shirudo-build
operator$ export PATH="$PWD:$PWD/tools/shirudo-build:$PATH"

# 3) 테스트 VM을 띄우고 대상 셸을 연다
operator$ make -C testing vm VM_NAME=quickstart
operator$ make -C testing vm-ssh VM_NAME=quickstart
target# cd /workspace/shirudo
target# shirudo status

# 4) 호스트에서 upatch를 빌드·서명한다
operator$ KEYS=/tmp/shirudo-test-vm-private-$(id -u)/operator-keys/keys
operator$ shirudo build testing/fixtures/basic.bpf.c --keys-dir "$KEYS" --vmlinux vendor/linux/vmlinux.h -o basic.upatch
operator$ PAYLOAD=$(shirudo upatch info --json basic.upatch | jq -r .payload_sha256)
operator$ shirudo build --unload basic --payload-sha256 "$PAYLOAD" --keys-dir "$KEYS" -o basic.unload.upatch

# 5) 대상에서 로드하고 확인한다
target# shirudo upatch load basic.upatch
target# shirudo upatch list

# 6) 끝나면 언로드하고 VM을 정리한다
target# shirudo upatch unload basic.unload.upatch
operator$ make -C testing vm-stop VM_NAME=quickstart
operator$ make -C testing vm-clean VM_NAME=quickstart
```

실습에 쓰는 `testing/fixtures/basic.bpf.c`는 커널 함수 `do_nanosleep()`에 붙어 아무 일도 하지 않는 프로그램이라 첫 로드로 안전하다고 문서는 설명한다. 서명 코드는 헬퍼(`tools/shirudo-build/shirudo-build`)에만 있어서 대상 호스트로 넘어가지 않는다.

이 튜토리얼에서 운영 관점으로 챙길 대목은 세 가지다.

1. **root도 못 건드린다.** 대상에서 root로 `cat /run/shirudo/bpffs/tamper`를 실행하면 Permission denied가 나온다. 런타임 이미지에서 실행된 신뢰된 바이너리만 Shirudo의 상태를 읽을 수 있고 거부된 시도는 `shirudo status`의 Tamper attempts에 유형별(`LoadNotLoader`, `MapCreate`, `BpfCommand`, `ObjGet`)로 집계된다.
2. **상태는 스크립트로 읽는다.** `shirudo status --json`이 판정을 내주고 명령은 판정과 무관하게 0으로 끝난다. `--check`를 주면 tampered일 때 1로 끝난다.
3. **언로드도 서명된 산출물이다.** 언로드 지시문은 upatch 이름과 제거할 페이로드를 지정하며 같은 이름으로 다른 페이로드가 올라가 있으면 대상이 `payload_mismatch`로 거부한다. 언로드해도 TPM 레지스터는 부팅 시점 값으로 돌아가지 않고 앞으로만 진행한다.

TPM 검증은 운영자가 nonce를 만들고(`openssl rand -hex 32`), 대상에서 `shirudo attest --nonce=<nonce>`로 증거를 받고, 호스트에서 `shirudo attest predict`로 레지스터 값을 예측한 뒤 `shirudo attest verify`로 대조하는 순서다. 검증 명령은 `verdict: ok`면 0, `verdict: fail`이면 1로 끝난다. 문서는 이 퀵스타트가 attestation 키 이름을 증거 자체에서 가져오므로 어느 TPM이 서명했는지는 증명하지 못한다고 스스로 밝히고 키를 한 번 고정하는 절차는 별도 문서(Attest a target)로 넘긴다.

프로덕션과 다른 점도 문서에 있다. QEMU가 커널과 initrd를 직접 시작하고 서명된 unified kernel image를 쓰지 않으므로 부팅 단계가 PCR 11에 측정되지 않는다.

## 지금 할 수 있는 것과 아닌 것

당장 할 수 있는 일은 평가다. KVM이 되는 Linux 호스트가 있으면 퀵스타트로 서명·로드·검증 흐름을 직접 볼 수 있고 환경이 없으면 [Isovalent의 브라우저 실습](https://labs.isovalent.com/#/lab/shirudo-getting-started){:target="_blank"}이 있다. 원문은 12월 15일 웨비나도 안내한다.

운영 중인 서버에 바로 올리는 것은 아직 아니다. 전용 커널 패치와 TPM 2.0이 전제이고 저장소가 pre-release라고 적고 있다. 원문도 Shirudo를 첫 단계라고 부르며 커널과 핵심 서비스 하드닝, 워크로드 권한 제한, 더 안전한 업데이트와 복구는 앞으로 강화할 영역으로 적는다. 취약점 공개와 함께 시험된 shield를 제공하겠다는 것도 목표로 제시된 내용이지 지금 제공되는 서비스가 아니다.

원문에는 성능 오버헤드, 지원 배포판 목록, 시험 규모에 대한 수치가 없다. Java4Shell 사례도 통제된 시험 한 건이다. 에이전트가 방어 코드를 만드는 부분은 Codex를 제어 평면으로 쓴 사례로 소개됐고 다른 에이전트로 같은 흐름을 구성하는 방법은 이번에 읽은 자료에는 나오지 않는다.

## 참고

- [OpenAI and Cisco introduce Shirudo](https://openai.com/index/introducing-shirudo){:target="_blank"} (OpenAI, 2026-11-23)
- [shirudo-dev/shirudo 저장소](https://github.com/shirudo-dev/shirudo){:target="_blank"}
- [Shirudo Quickstart](https://docs.shirudo.dev/start/quickstart.html){:target="_blank"}
- [Shirudo 문서](https://docs.shirudo.dev/index.html){:target="_blank"}
- [Isovalent 블로그 발표문](https://isovalent.com/blog/post/cisco-openai-shirudo/){:target="_blank"}
- [Isovalent Shirudo 실습](https://labs.isovalent.com/#/lab/shirudo-getting-started){:target="_blank"}
