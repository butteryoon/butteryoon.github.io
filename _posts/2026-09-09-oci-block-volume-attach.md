---
layout: post
comments: true
title: "오라클 OCI 프리티어 인스턴스에 블록볼륨 추가하기"
description: "Oracle Cloud 인스턴스에 블록 볼륨을 붙이는 전체 과정을 실제로 해보고 정리했다. 볼륨 attach부터 파티션/GPT 생성, ext4 포맷, /data 마운트, UUID 기반 fstab 자동마운트까지."
img: oci_block_volume_title.webp
date: 2026-09-09 23:30:00 +0900
last_modified_at: 2026-09-10 11:30:00 +0900
tags: [oci, oracle-cloud, block-volume, storage, mount, fstab, ext4] # add tag
related: oci
categories: tools
---

오라클 클라우드의 [무료 인스턴스 시작하기]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)와 [OmniRoute 셀프호스팅]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)을 다룬 적 있다. 그 무료 인스턴스는 기본 부팅 볼륨(46.6GB)만으로도 뭐든 돌렸는데, LLM 모델이나 로그 데이터를 쌓다 보니 공간이 모자랐다. 이번엔 무료 인스턴스 두 대 — **oc01**(Rocky Linux 9, VM.Standard.E2.1.Micro, 서울 AD-1)과 **oc02**(Ubuntu) — 에 각각 **100GB 블록 볼륨을 붙여서** 확인·마운트·자동마운트까지 실제로 작업한 기록을 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다. 2026-09-10 실작업 기록으로 전면 갱신.)

<!--more-->

> **TL;DR:** OCI에서 블록 볼륨은 콘솔에서 "생성→인스턴스에 연결(attach)"까지만 하고, 나머지는 인스턴스 안의 리눅스에서 해줘야 한다. **Paravirtualized로 attach하면 iscsiadm 없이 재부팅도 없이 바로 `/dev/sdb`로 잡힌다.** 포맷 전에 반드시 기존 데이터를 확인하고(읽기 전용 마운트로 probe), 마운트는 `/data`, fstab은 **UUID 기준 + `nofail`**로 등록한다. 덤으로 얻은 교훈: **clone한 볼륨은 파일시스템 UUID까지 복제되므로** 원본과 같이 붙일 거면 `tune2fs -U random`으로 바꿔야 한다.

## 1. OCI 블록 볼륨의 구조 — "붙이는 것"과 "쓰는 것"은 다르다

OCI 블록 볼륨을 쓰는 과정은 **두 레이어**로 나뉜다.

| 레이어 | 하는 일 | 어디서 | 작업 주체 |
|--------|---------|--------|-----------|
| 제어 평면 (Control Plane) | 볼륨 생성, 인스턴스에 **연결(attach)** | OCI 콘솔 / `oci` CLI | 오라클 관리자 |
| 데이터 평면 (OS) | 디바이스 인식, 파티션, 포맷, 마운트, fstab | 인스턴스 안 리눅스 | 서버 관리자 |

처음 하는 사람은 콘솔에서 "연결"만 하고 끝내기 쉬운데, 거기까지는 USB를 꽂아둔 것과 같다. OS가 그 드라이브를 읽으려면 **파티션 → 포맷 → 마운트 → 자동마운트**까지 해줘야 한다. 2절이 콘솔 단계, 3~5절이 "OS에서 쓰기" 단계다.

## 2. 관리 콘솔에서 볼륨 생성·연결

OCI 관리 콘솔에서 햄버거 메뉴 → **스토리지(Storage)** → **블록 볼륨(Block Volumes)**으로 들어가면 볼륨 목록이 나온다. 여기서 **블록 볼륨 생성**으로 원하는 크기(프리티어는 부팅 볼륨 포함 총 200GB까지 무료)의 볼륨을 만들고, 생성된 볼륨의 상세 화면에서 **연결된 인스턴스 → 인스턴스에 연결(attach)**을 누르면 콘솔에서 할 일은 끝이다.

![OCI 콘솔 블록 볼륨 화면]({{site.baseurl}}/assets/img/oci_block_volume_console.webp)

연결 유형은 **Paravirtualized(반가상화)를 권장**한다. iSCSI로 붙이면 콘솔이 알려주는 `iscsiadm` 명령 3줄을 인스턴스에서 실행해야 하지만, Paravirtualized는 재부팅도 iSCSI 세션도 없이 즉시 `/dev/sdb`로 잡힌다 — E2.1.Micro 같은 소형 shape에는 이쪽이 간단하다. (`iscsiadm -m session`이 "No active sessions"로 나오는 게 Paravirtualized에선 정상이다. 반대로 iSCSI attach를 했는데 `iscsiadm -m discovery -t sendtargets -p 169.254.2.2:3260`이 Connection refused면 볼륨이 아직 안 붙었다는 진단 신호다.) 이제부터는 인스턴스 안 리눅스에서의 작업이다.

## 3. 연결된 볼륨 확인 — `lsblk`

블록 볼륨을 인스턴스에 연결하면 리눅스에서 `sdb`, `sdc` 같은 디바이스로 잡힌다. 이 인스턴스에는 100GB 볼륨이 이미 붙어 있었다.

```text
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
NAME    SIZE TYPE FSTYPE MOUNTPOINT MODEL
sdb     100G disk                   BlockVolume
└─sdb1  100G part ext4   /data
sda    46.6G disk                   BlockVolume
├─sda2    8G part swap   [SWAP]
├─sda3 38.4G part xfs    /          # ← 부팅 볼륨
└─sda1  200M part vfat   /boot/efi
```

여기서 볼 건 두 줄이다.
- **`sda` = 부팅 볼륨** (`/`, `/boot/efi`, swap). xfs로 포맷해 마운트해둔 상태.
- **`sdb` = 외부 블록 볼륨** (MODEL `BlockVolume`). 처음엔 FSTYPE/MOUNTPOINT가 비어 있었고, 아래 절차를 거쳐 `/dev/sdb1` ext4 → `/data`까지 채운 **최종 상태**다.

## 4. 포맷 전에 반드시: 기존 데이터 확인

이번에 붙인 볼륨은 **이미 GPT + ext4 파티션(sdb1)이 있는 상태**였다. 무턱대고 포맷하면 데이터가 날아가므로, 포맷 전 확인이 먼저다.

```bash
$ sudo blkid /dev/sdb1                      # 파일시스템 존재 여부 확인
$ sudo mkdir -p /mnt/probe
$ sudo mount -o ro /dev/sdb1 /mnt/probe    # 읽기 전용으로 열어보기
$ ls /mnt/probe                             # lost+found 만 있으면 빈 볼륨
$ sudo umount /mnt/probe
```

`lost+found`만 있으면 빈 ext4 볼륨이니 **포맷을 생략하고 그대로 쓰면 된다** — 이번 작업이 그 경우였다.

완전히 새 볼륨(파티션 테이블 없음)이라면 GPT 파티션을 만들고 포맷한다. 대화형 fdisk보다 `parted` 한 줄이 스크립트화하기 좋다.

```bash
$ sudo parted -s /dev/sdb mklabel gpt mkpart data ext4 1MiB 100%
$ sudo mkfs.ext4 /dev/sdb1
```

여기서 GPT 레이블은 "이 디스크를 어떤 파티션 구조로 나눠 쓸지"를 기록하는 파티션 테이블 표준이다. 옛 MBR 방식의 2TB·4파티션 한계가 없어서, 요즘 새 디스크는 그냥 GPT로 만들면 된다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"The GUID Partition Table (GPT) is a standard for the layout of partition tables of a physical computer storage device, such as a hard disk drive or solid-state drive. It is part of the Unified Extensible Firmware Interface (UEFI) standard." — Wikipedia, GUID Partition Table</blockquote></details>

## 5. 마운트 + 자동마운트(fstab)

### 5-1. 마운트

마운트 포인트는 `/data`로 잡았다. fstab을 건드리기 전에 백업부터 뜨는 습관도 함께.

```bash
$ sudo mkdir -p /data
$ sudo cp /etc/fstab /etc/fstab.bak.$(date +%Y%m%d_%H%M%S)
$ sudo mount /dev/sdb1 /data
$ df -hT /data
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb1      ext4   98G   60M   93G   1% /data
```

`/data`에 잘 붙었다. 100GB 볼륨에서 ext4 메타데이터·예약 블록 몫을 빼고 93GB를 쓸 수 있다. oc01(Rocky)·oc02(Ubuntu) 모두 같은 절차로 동일한 결과.

### 5-2. UUID 기반 자동마운트 (fstab)

여기가 **가장 큰 함정**이다. OCI 인스턴스의 `/etc/fstab` 주석에 오라클이 아예 경고를 박아놨다.

```text
## If you are adding an iSCSI remote block volume to this file you MUST
## include the '_netdev' mount option or your instance will become
## unavailable after the next reboot.
## SCSI device names are not stable across reboots; please use the device UUID instead of /dev path.
```

**재부팅하면 `/dev/sdb`가 `/dev/sdc`로 바뀔 수 있다는 얘기다.** 경로 대신 고정된 UUID로 등록해야 한다. 공식 문서도 attach만으로는 부팅 때 마운트되지 않으니 설정을 더 해야 한다고 못 박는다.

<details class="evidence"><summary>원문 근거</summary><blockquote>"If you want to mount these volumes when the instance starts, you need to perform additional configuration steps: If you specified a device path when you attached the volume, see /etc/fstab Options for Block Volumes Using Consistent Device Paths. If you didn't specify a device path ..., see Traditional fstab Options." — OCI 공식 문서 (Connecting to a Volume)</blockquote></details>

```text
$ sudo blkid /dev/sdb1
/dev/sdb1: UUID="806ae008-6cad-48a4-82a1-e7099696469f" TYPE="ext4" PARTUUID="bccc114a-..."

$ echo 'UUID=806ae008-6cad-48a4-82a1-e7099696469f /data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
UUID=806ae008-6cad-48a4-82a1-e7099696469f /data ext4 defaults,nofail 0 2
```

- **`defaults,nofail`은 필수에 가깝다** — 나중에 볼륨을 detach하면 fstab 항목 때문에 부팅이 멈출 수 있는데, `nofail`이 그걸 막는다. (오라클이 권하는 `_netdev`를 같이 넣어도 좋다)
- 등록한 뒤 `sudo mount -a`로 실제 마운트를, `sudo findmnt --verify --fstab`으로 문법을 검증하면 재부팅 전에 fstab이 멀쩡한지 알 수 있다. `findmnt /data`로 최종 상태 확인까지.

### 5-3. 권한 설정

마지막으로 인스턴스 기본 사용자에게 소유권을 넘겼다 (Rocky는 `rocky`, Ubuntu는 `ubuntu` — 이미지에 따라 다르다).

```bash
$ sudo chown rocky:rocky /data     # oc01 (Rocky Linux 9)
$ sudo chown ubuntu:ubuntu /data   # oc02 (Ubuntu)
```

## 6. 트러블슈팅 — clone 볼륨의 UUID 충돌

이번 작업에서 얻은 가장 값진 발견이다. **볼륨을 clone하면 파일시스템 UUID까지 그대로 복제된다.** 실제로 oc01의 볼륨이 oc02 볼륨의 clone이라 두 볼륨의 UUID가 완전히 같았다(위의 `806ae008-...`). 인스턴스가 다르면 당장은 문제없지만, 한 인스턴스에 원본과 클론을 동시에 붙이면 fstab의 UUID 매칭이 꼬인다. 그럴 땐 클론 쪽 UUID를 새로 발급하면 된다.

```bash
$ sudo tune2fs -U random /dev/sdb1   # 새 UUID 발급 (fstab도 갱신할 것)
```

그리고 용량 계획 참고: Always Free 한도는 **부트 볼륨 + 블록 볼륨 합산 200GB**다. 46.6GB 부트 2개 + 100GB 블록 1개면 거의 꽉 찬다.

## 마무리

OCI 블록 볼륨을 처음 만지면 가장 헷갈리는 게 "콘솔에서 attach한 다음 뭘 해야 하나"인데, 그 부분을 실제 명령 출력과 함께 정리했다. 다시 짚으면:

1. OCI 콘솔에서 **연결(attach)**까지만 하면 OS엔 그냥 `sdb`로 보인다.
2. **기존 데이터 확인 → (새 볼륨이면 파티션·포맷) → 마운트** 순서를 인스턴스 안에서 끝내야 볼륨을 쓸 수 있다. 포맷 전 read-only probe는 습관으로.
3. **fstab엔 반드시 UUID로 등록**하자 — 디바이스명은 재부팅마다 바뀐다. `nofail`(혹은 `_netdev`) 옵션을 넣어 부팅이 막히는 일을 피하자.

부팅 볼륨(46.6GB)에 LLM 모델을 담기엔 빠듯했는데, 이제 `/data` 100GB가 생겼다. 다만 한 가지 짚어둘 것 — **OCI 프리티어 인스턴스에는 GPU가 없어서 vLLM으로 모델을 서빙할 수는 없다.** 이 공간의 용도는 모델 파일 보관, 로그·백업 저장, 그리고 Ollama로 소형 모델을 CPU 추론으로 굴려보는 정도까지다. 본격적인 서빙은 GPU 있는 환경의 몫이고, 여기서는 앞서 만든 [OmniRoute AI 게이트웨이]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)의 저장 공간으로 활용할 생각이다.

## 참고

- [OCI 블록 볼륨 연결 공식 문서](https://docs.oracle.com/en-us/iaas/Content/Block/Tasks/connectingtoavolume.htm)
- [오라클 무료 인스턴스 시작하기]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)
- [OmniRoute 셀프호스팅 — RAM 1GB로 AI 게이트웨이 돌리기]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)
- 타이틀 사진: [Unsplash](https://unsplash.com/photos/VYfxkePredI){:target="_blank"}