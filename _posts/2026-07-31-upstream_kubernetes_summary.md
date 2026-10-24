---
layout: post
comments: true
title: "Upstream Kubernetes 요약 — 아키텍처·버전 차이 정책·릴리즈, 그리고 배포판과의 차이"
description: "upstream(바닐라) Kubernetes의 아키텍처·버전 차이(skew) 정책·릴리즈 단계를 공식 문서 기준으로 정리하고, GPU 워크로드를 올릴 때 추가로 걸리는 지점(Device Plugin·GPU Operator·배치 스케줄러)과 OpenShift·K3s·RKE2 배포판 선택 기준을 비교한다."
img: cloud-title.webp
date: 2026-07-31 01:21:00 +0900
last_modified_at: 2026-10-24 20:00:00 +0900
tags: [kubernetes, architecture, control-plane, k3s, openshift, devops, gpu, llm-infra, toolbox]
related: toolbox
categories: dev
---

[NVIDIA AI Factory 해설]({{site.baseurl}}/dev/2026/07/30/nvidia_ai_factory_design_guide.html)에서 인프라 요구사항으로 "CNCF 바닐라 K8s"라는 표현이 나온다. 그런데 실제 구축 현장에서는 upstream(바닐라) Kubernetes와 OpenShift·K3s 같은 배포판 중 무엇을 쓸지가 늘 논점이다. 이 글은 upstream Kubernetes의 핵심을 요약하고 다른 패키지들과의 차이를 비교한다. **2026-10-24 보완**: GPU 워크로드를 올릴 때 추가로 걸리는 지점(Device Plugin·GPU Operator·버전 조합)과 기본 스케줄러의 한계(큐·갱 스케줄링) 두 절을 더했다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검토·보완해 발행했다.)

<!--more-->

> **TL;DR:** upstream Kubernetes는 Control Plane(`kube-apiserver`·`etcd`·`controller-manager`·`scheduler`)과 워커(`kubelet`·`kube-proxy`)로 구성되며, 버전 차이(skew)는 "kubelet은 apiserver보다 최대 3개 마이너 낮게, 절대 높지 않게"가 핵심 규칙이다. 기능은 Alpha → Beta → GA 단계로 성숙한다. 배포판 선택은 "전체 제어(upstream) vs 엔터프라이즈 레이어(OpenShift) vs 경량(K3s/RKE2) vs 비-K8s(Swarm/Nomad)"의 트레이드오프다. **GPU 워크로드를 올릴 생각이라면 여기에 두 가지가 더 붙는다** — GPU는 Device Plugin 경로로만 보이고(GPU Operator가 그 묶음을 깔아 준다), 기본 스케줄러는 파드를 하나씩 배치해서 분산 학습 같은 다중 파드 작업에는 Kueue·Volcano 같은 큐·갱 스케줄링 계층이 필요하다.

## Control Plane 구성 요소

| 컴포넌트 | 역할 |
|----------|------|
| `kube-apiserver` | API 진입점, 인증·인가·감사 |
| `etcd` | 클러스터 상태 저장소 (분산 KV) |
| `kube-controller-manager` | ReplicaSet·Deployment 등 컨트롤러 루프 실행 |
| `kube-scheduler` | 자원·제약조건 기반 파드-노드 매칭 |

kubeadm으로 구축한 클러스터라면 네 요소가 **static pod**로 `/etc/kubernetes/manifests/`에 놓이며, `kubectl get pods -n kube-system`으로 상태를 확인할 수 있다 (관리형 서비스나 일부 배포판은 구성이 다르다).

워커 노드에는 컨테이너를 실제로 돌리고 보고하는 `kubelet`(+containerd 등 CRI 런타임)과 Service 라우팅을 담당하는 `kube-proxy`(iptables/ipvs 모드)가 있다.

## Upstream과 다른 패키지의 차이

| 패키지 | 배포 형태 | API 호환성 | 운영 복잡도 | 주요 사용처 |
|--------|-----------|-----------|-------------|-------------|
| **Kubernetes (upstream)** | OSS 소스, kubeadm/kops 등으로 직접 설치 | 표준 API 전체 | 설치·업그레이드 직접 관리 (학습 곡선) | 전체 제어가 필요한 온프레미스·하이브리드 |
| **Docker Swarm** | Docker Engine 내장 | Docker API (K8s와 비호환) | `docker swarm init` 한 줄 | 소규모 Docker 중심 서비스 |
| **HashiCorp Nomad** | 단일 바이너리 | 자체 job DSL (K8s API 아님) | 가벼운 설치, 유연한 워크로드 | 컨테이너+비컨테이너 혼합, 배치 작업 |
| **Red Hat OpenShift** | K8s 위에 엔터프라이즈 레이어 | K8s 호환 + OpenShift 전용 API | 설치·구성 무겁지만 상용 지원 | 규제·보안 중심 엔터프라이즈 |
| **K3s** | 단일 바이너리 경량 배포판 | K8s 인증(conformant) 배포판 | `k3s server` 수준으로 간단 | 엣지·IoT·리소스 제약 환경 |
| **RKE2** | K3s 계열 보안 강화판 (FIPS) | 표준 K8s API | 설치·업그레이드 자동화 | 보안·규정 준수가 중요한 환경 |

정리하면 네 가지 축의 트레이드오프다.

- **배포·운영 모델**: upstream은 kubeadm 등으로 직접 설치·업그레이드하는 것이 기본. 배포판은 그 부담을 자동화(K3s/RKE2)하거나 상용 지원(OpenShift)으로 흡수한다.
- **API 범위**: upstream·K3s·RKE2·OpenShift는 표준 K8s API를 공유하므로 워크로드 이식이 쉽다. Swarm·Nomad는 API 체계 자체가 달라 이전 비용이 크다.
- **생태계**: K8s 계열은 CNI/CSI/CRI 플러그인 생태계를 공유한다. OpenShift는 여기에 이미지 레지스트리·CI/CD·보안 레이어를 얹는다.
- **유스케이스**: 전체 제어·멀티클라우드는 upstream, 엣지·IoT는 K3s, 규제 환경은 OpenShift/RKE2, Docker 중심 소규모는 Swarm, 혼합 워크로드는 Nomad.

[AI Factory 해설]({{site.baseurl}}/dev/2026/07/30/nvidia_ai_factory_design_guide.html)의 파트너 플랫폼 목록(OpenShift, Rancher, Charmed K8s 등)이 정확히 이 배포판 계층이고, NVIDIA GPU Operator·Kueue 같은 도구는 표준 K8s API 위에서 동작하므로 upstream이든 인증 배포판이든 올릴 수 있다.

**GPU 노드를 붙일 계획이라면 기준이 하나 더 생긴다.** 어느 쪽을 골라도 GPU Operator는 올라가지만, 손이 가는 양이 다르다. 드라이버를 노드 OS에 직접 맞춰야 하는 upstream은 조합을 전부 사람이 책임지는 대신 버전 선택이 자유롭다. OpenShift·RKE2처럼 OS·커널까지 함께 관리하는 쪽은 검증된 조합을 받는 대신 그 벤더가 올려 주는 속도를 기다려야 한다. K3s는 엣지 추론에는 가볍지만 다중 GPU 노드 학습 클러스터로 키울 그림이라면 애초에 대상이 아니다. **"지금 돌릴 수 있는가"보다 "드라이버를 누가 책임지는가"로 가르는 편이 실제 운영과 맞는다.**

## 버전 차이(skew) 정책 — 자주 틀리는 부분

[공식 정책](https://kubernetes.io/releases/version-skew-policy/){:target="_blank"} 기준으로 규칙이 컴포넌트 쌍마다 다르다. "±1"로 뭉뚱그리면 틀린다 (공식 한국어 문서 용어는 "버전 차이(skew) 정책"이다).

| 컴포넌트 쌍 | 허용되는 버전 차이 |
|-------------|-----------|
| kubelet ↔ kube-apiserver | kubelet은 **최대 3개 마이너 낮게** (1.25 미만은 2개), **절대 apiserver보다 높으면 안 됨** |
| kube-apiserver 인스턴스 간 (HA) | 최신·최구 인스턴스가 **1개 마이너 이내** |
| kubectl ↔ kube-apiserver | **±1 마이너** (높아도, 낮아도 1까지) |

예: apiserver가 v1.36이면 kubelet은 v1.33~1.36, kubectl은 v1.35~1.37이 지원 범위다. 업그레이드 순서가 "Control Plane 먼저, 워커 나중"인 이유가 이 규칙(kubelet은 apiserver보다 높으면 안 됨)에서 나온다.

## GPU 워크로드를 올릴 때 추가로 걸리는 것

AI 팩토리든 사내 LLM 서빙이든, 위 구성 그대로는 GPU가 보이지 않는다. 쿠버네티스는 GPU를 일반 자원처럼 다루지 않고 **[Device Plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/){:target="_blank"} 경로**로만 노출한다. 노드에 드라이버·컨테이너 런타임 훅·디바이스 플러그인이 모두 갖춰져야 `nvidia.com/gpu` 자원이 스케줄러 눈에 들어온다.

그 묶음을 수작업으로 맞추지 않으려고 쓰는 것이 [NVIDIA GPU Operator](https://github.com/NVIDIA/gpu-operator){:target="_blank"}다. 드라이버·container-toolkit·device-plugin·DCGM exporter를 오퍼레이터가 한꺼번에 설치·갱신한다. **표준 K8s API 위에서 도므로 upstream이든 인증 배포판이든 올라간다.**

실무에서 걸리는 지점은 셋이다.

- **GPU 자원은 쪼개지지 않는다.** `nvidia.com/gpu: 1`은 정수 단위이고 CPU·메모리처럼 오버커밋되지 않는다. 한 GPU를 여러 파드가 나눠 쓰려면 MIG 분할이나 타임 슬라이싱을 따로 설정해야 한다 ([공식 문서](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/){:target="_blank"}).
- **버전 조합이 하나 더 늘어난다.** 아래 skew 정책은 K8s 컴포넌트 사이의 규칙이고, 여기에 **드라이버·커널·GPU Operator·K8s 버전**의 벤더 호환 조합이 겹친다. 클러스터 업그레이드 전에 [GPU Operator 플랫폼 지원 표](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/platform-support.html){:target="_blank"}를 먼저 보는 편이 안전하다 — K8s 쪽 skew를 지켜도 드라이버 쪽에서 막히는 경우가 있다.
- **노드가 동질적이지 않다.** GPU 노드와 CPU 노드가 섞이면 taint/toleration과 nodeSelector를 쓰지 않는 한 일반 파드가 비싼 GPU 노드를 차지한다.

## 기본 스케줄러로 부족한 순간 — 큐와 갱 스케줄링

`kube-scheduler`는 **파드를 하나씩** 배치한다. 단일 추론 파드에는 문제가 없지만, 분산 학습처럼 **N개 파드가 모두 떠야 의미가 있는 작업**에서는 이 방식이 함정이 된다. 8개 중 5개만 자리를 잡으면 그 5개가 GPU를 점유한 채 나머지를 기다리고, 다른 작업도 못 들어온다.

그래서 얹는 계층이 둘이다.

- **[Kueue](https://kueue.sigs.k8s.io/){:target="_blank"}** — 잡 단위 **큐잉과 쿼터**. 자원이 날 때까지 잡을 대기열에 세워 두고, 팀·네임스페이스별 할당량을 지킨다.
- **[Volcano](https://volcano.sh/){:target="_blank"}** — **갱(gang) 스케줄링**. 필요한 파드 수가 한꺼번에 확보될 때만 배치해 부분 점유로 인한 교착을 막는다.

[NVIDIA AI Factory 오픈소스 구성]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)에서 Cloud Native Platform 계층을 "K8s + GPU Operator + Kueue/Volcano"로 잡는 이유가 여기 있다. 기본 쿠버네티스에 빠져 있는 조각을 메우는 것이지, 다른 종류의 쿠버네티스를 쓰는 게 아니다.

## 릴리즈 단계 (Alpha → Beta → GA)

| 단계 | 특징 |
|------|------|
| **Alpha** | 실험적 기능. 기본 비활성이며 `--feature-gates`로 켠다. 예고 없이 제거될 수 있음 |
| **Beta** | 안정화 단계. 대부분 기본 활성화되고 문서화 완료. API가 바뀔 수 있음 |
| **GA (Stable)** | 프로덕션 준비 완료. 정식 API에 편입되어 하위 호환 보장 |

새 마이너 버전은 대략 4개월 주기로 릴리즈되고, 최근 3개 마이너 버전이 패치 지원을 받는다. 클러스터 버전 확인은 `kubectl version`으로 client/server 버전을 비교하면 된다.

## 실전 체크리스트

1. **Control Plane 상태**: `kubectl get pods -n kube-system`에서 apiserver·etcd·controller-manager·scheduler Running 확인
2. **버전 차이 확인**: `kubectl version`으로 client/server 차이가 ±1 이내인지, 노드는 `kubectl get nodes`로 kubelet 버전이 apiserver 이하인지 점검
3. **업그레이드**: kubeadm 기준 `kubeadm upgrade plan` → `kubeadm upgrade apply vX.Y.Z` → 노드별 kubelet 업그레이드 순서
4. **로그 진단**: static pod 구성이면 `kubectl logs -n kube-system kube-apiserver-<노드명>`이 첫 진단 지점

## 참고

- [Kubernetes Components (공식 문서)](https://kubernetes.io/docs/concepts/overview/components/){:target="_blank"}
- [Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/){:target="_blank"}
- [Kubernetes Release Process](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/README.md){:target="_blank"}
- [NVIDIA GPU Operator (저장소)](https://github.com/NVIDIA/gpu-operator){:target="_blank"} · [Kueue](https://kueue.sigs.k8s.io/){:target="_blank"} · [Volcano](https://volcano.sh/){:target="_blank"}
- [NVIDIA AI Factory 해설 (관련글)]({{site.baseurl}}/dev/2026/07/30/nvidia_ai_factory_design_guide.html)
- [AI Factory 오픈소스로 구성하기 (관련글)]({{site.baseurl}}/dev/2026/07/24/ai_factory_opensource.html)
- [블로그 정리하기 (관련글)]({{site.baseurl}}/tools/2026/07/14/blog_maintenance.html)
