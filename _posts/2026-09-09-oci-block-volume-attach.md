---
layout: post
comments: true
title: "OCI 블록 볼륨 연결 · 포맷 · 마운트 · 자동마운트 — 실전 기록"
description: "Oracle Cloud 인스턴스에 블록 볼륨을 붙이는 전체 과정을 실제로 해보고 정리했다. 볼륨 attach부터 파티션/GPT 생성, ext4 포맷, /mnt/data 마운트, UUID 기반 fstab 자동마운트까지."
img: oci_block_volume_title.webp
date: 2026-09-09 23:30:00 +0900
last_modified_at: 2026-09-09 23:30:00 +0900
tags: [oci, oracle-cloud, block-volume, storage, mount, fstab, ext4] # add tag
related: oci
categories: tools
---

오라클 클라우드의 [무료 인스턴스 시작하기]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)와 [OmniRoute 셀프호스팅]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)을 다룬 적 있다. 그 무료 인스턴스는 기본 부팅 볼륨(46.6GB)만으로도 뭐든 돌렸는데, LLM 모델이나 로그 데이터를 쌓다 보니 공간이 모자랐다. 이번엔 그 인스턴스(oc01)에 **100GB 블록 볼륨을 붙여서** 포맷·마운트·자동마운트까지 해본 기록을 정리한다. (이 글의 초안은 설치된 Hermes 에이전트가 작성했고, Claude가 검수 후 발행했다.)

<!--more-->

> **TL;DR:** OCI에서 블록 볼륨은 콘솔에서 "생성→인스턴스에 연결(attach)"까지만 하고, 나머지 파티션(GPT)·포맷(ext4)·마운트·부팅시 자동마운트(fstab)는 인스턴스 안의 리눅스에서 해줘야 한다. 여기서는 붙여둔 100GB 볼륨을 `/dev/sdb`로 확인해 → `/dev/sdb1` 파티션 생성 → ext4 포맷 → `/mnt/data`에 마운트 → **UUID 기준 fstab 등록**(재부팅 직후에도 그대로)까지 실제 명령 출력으로 보여준다. 남는 교훈은 두 가지다. SCSI 디바이스명은 재부팅마다 바뀌니 fstab엔 UUID를 쓸 것, 그리고 OCI 외부 블록 볼륨엔 `_netdev`/`nofail` 옵션을 붙여두는 편이 안전하다는 것.

## 1. OCI 블록 볼륨의 구조 — "붙이는 것"과 "쓰는 것"은 다르다

OCI 블록 볼륨을 쓰는 과정은 **두 레이어**로 나뉜다.

| 레이어 | 하는 일 | 어디서 | 작업 주체 |
|--------|---------|--------|-----------|
| 제어 평면 (Control Plane) | 볼륨 생성, 인스턴스에 **연결(attach)** | OCI 콘솔 / `oci` CLI | 오라클 관리자 |
| 데이터 평면 (OS) | 디바이스 인식, 파티션, 포맷, 마운트, fstab | 인스턴스 안 리눅스 | 서버 관리자 |

처음 하는 사람은 콘솔에서 "연결"만 하고 끝내기 쉬운데, 거기까지는 USB를 꽂아둔 것과 같다. OS가 그 드라이브를 읽으려면 **파티션 → 포맷 → 마운트 → 자동마운트**까지 해줘야 한다. 2~5절이 바로 그 "OS에서 쓰기" 단계다.

## 2. 연결된 볼륨 확인 — `lsblk`

블록 볼륨을 인스턴스에 연결하면 리눅스에서 `sdb`, `sdc` 같은 디바이스로 잡힌다. 이 인스턴스에는 100GB 볼륨이 이미 붙어 있었다.

```text
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT,MODEL
NAME    SIZE TYPE FSTYPE MOUNTPOINT MODEL
sdb     100G disk                   BlockVolume
└─sdb1  100G part ext4   /mnt/data
sda    46.6G disk                   BlockVolume
├─sda2    8G part swap   [SWAP]
├─sda3 38.4G part xfs    /          # ← 부팅 볼륨
└─sda1  200M part vfat   /boot/efi
```

여기서 볼 건 두 줄이다.
- **`sda` = 부팅 볼륨** (`/`, `/boot/efi`, swap). xfs로 포맷해 마운트해둔 상태.
- **`sdb` = 외부 블록 볼륨** (MODEL `BlockVolume`). 처음엔 FSTYPE/MOUNTPOINT가 비어 있었고, 아래 절차를 거쳐 `/dev/sdb1` ext4 → `/mnt/data`까지 채운 **최종 상태**다.

## 3. 파티션 생성 (GPT)

`/dev/sdb`엔 파티션 테이블이 아예 없었다. `fdisk`로 GPT 파티션을 하나 만든다. (4KB 섹터 정렬 경고가 떠도 파티션 번호·섹터를 기본값으로 두면 알아서 정렬된다.)

```text
$ sudo fdisk /dev/sdb <<EOF
> g
> n
> 1
>
>
> w
> EOF
Welcome to fdisk (util-linux 2.23.2).
Building a new GPT disklabel (GUID: 4B9F7297-...)
...
Created partition 1
The partition table has been altered!
Syncing disks.
```

`g` = GPT 디스크 레이블 생성, `n` = 새 파티션(1번, 전체 용량), `w` = 쓰기. 이렇게 `/dev/sdb1`이 생겼다.

## 4. ext4 포맷

빈 파티션은 바로 쓸 수 없고 파일시스템을 얹어야 한다. 데이터가 든 볼륨이면 절대 포맷하지 말 것 — 이 100GB는 새로 쓸 볼륨이라 마음 놓고 밀었다.

```text
$ sudo mkfs.ext4 /dev/sdb1
...
Creating journal (131072 blocks): done
Writing superblocks and filesystem accounting information: done
```

## 5. 마운트 + 자동마운트(fstab)

### 5-1. 마운트

```text
$ sudo mkdir -p /mnt/data && sudo mount /dev/sdb1 /mnt/data
$ df -hT /mnt/data
Filesystem     Type  Size  Used Avail Use% Mounted on
/dev/sdb1      ext4   98G   60M   93G   1% /mnt/data
```

`/mnt/data`에 잘 붙었다. 100GB 볼륨에서 ext4 메타데이터·예약 블록 몫을 빼고 93GB를 쓸 수 있다.

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

$ echo 'UUID=806ae008-6cad-48a4-82a1-e7099696469f /mnt/data ext4 defaults,nofail 0 2' | sudo tee -a /etc/fstab
UUID=806ae008-6cad-48a4-82a1-e7099696469f /mnt/data ext4 defaults,nofail 0 2
```

- **`defaults,nofail`**: 볼륨이 없어도 부팅을 막지 않는다. (오라클이 권하는 `_netdev`를 같이 넣어도 좋다)
- 등록한 뒤 `sudo mount -a`로 문법을 확인하면 재부팅 전에 fstab이 멀쩡한지 알 수 있다.

### 5-3. 권한 설정

마지막으로 `opc` 사용자(무료 인스턴스 기본 계정)에게 소유권을 넘겼다.

```text
$ sudo chown -R opc:opc /mnt/data
```

## 마무리

OCI 블록 볼륨을 처음 만지면 가장 헷갈리는 게 "콘솔에서 attach한 다음 뭘 해야 하나"인데, 그 부분을 실제 명령 출력과 함께 정리했다. 다시 짚으면:

1. OCI 콘솔에서 **연결(attach)**까지만 하면 OS엔 그냥 `sdb`로 보인다.
2. **파티션(GPT) → 포맷(ext4) → 마운트** 이 3단계를 인스턴스 안에서 끝내야 볼륨을 쓸 수 있다.
3. **fstab엔 반드시 UUID로 등록**하자 — 디바이스명은 재부팅마다 바뀐다. `nofail`(혹은 `_netdev`) 옵션을 넣어 부팅이 막히는 일을 피하자.

부팅 볼륨(46.6GB)에 LLM 모델을 담기엔 빠듯했는데, 이제 `/mnt/data` 100GB가 생겼다. 다음 편에선 이 공간에 Ollama나 vLLM을 올려 로컬 모델을 서빙하고, 앞서 만든 [OmniRoute AI 게이트웨이]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)에 붙여볼 생각이다.

## 참고

- [OCI 블록 볼륨 연결 공식 문서](https://docs.oracle.com/en-us/iaas/Content/Block/Tasks/connectingtoavolume.htm)
- [먹는 서버: 오라클 무료 인스턴스 시작하기]({{site.baseurl}}/tools/2021/01/18/oracle_cloud_start.html)
- [OmniRoute 셀프호스팅 — RAM 1GB로 AI 게이트웨이 돌리기]({{site.baseurl}}/tools/2026/09/06/omniroute-selfhosting-oci.html)