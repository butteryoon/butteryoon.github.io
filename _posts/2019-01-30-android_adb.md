---
layout: post
title: "Adnroid Studio 없이 adb 사용하기"
img: android-adb.jpg
date: 2019-01-30 18:38:00 +0900
tags: [android, 안드로이드, adb] # add tag
categories: dev
---

윈도우즈 환경에서 adb를 사용하려면 그리 간단하지 않다. 

안드로이드 스튜디오 및 SDK를 설치해야 하고 특히 내 노트북과 같이 성능이 좋지 않은 환경에서는 안드로이드 스튜디오를 돌리는 것도 버벅대는 일도 있고 환경설정이 바뀌거나 하면 실행이 되지 않는다. 

Android Studio가 설치되어 있는 개발자 환경이 아니고 adb로 안드로이드 디바이스 모니터링만 하는 경우 안드로이드 스튜디오 및 SDK를 설치하지 않고 명령어라인에서 USB 연결로 ADB를 사용하기 위한 방법을 정리해본다. 


## 1. adb 명령어 라인 툴 설치 

-  [Minimal ADB and Fastboot](https://forum.xda-developers.com/showthread.php?t=2317790) 에서 minimal_adb_fastboot_1.4.3_portable.zip 다운로드 후 압축을 풀면 아래와 같이 adb와 dll이 보인다. 

> 설치된 디렉토리로 이동하여 쓸 수도 있지만  "제어판\시스템 및 보안\시스템 > 고급 시스템 설정 > 환경변수 에 ADB_PATH를 추가하고 Path 사용자변수에 추가하면 편리하다.

```
$ ls
adb.exe*  AdbWinApi.dll*  AdbWinUsbApi.dll*  cmd-here.exe*  Disclaimer.txt  fastboot.exe*
```

## 2. adb 안드로이드 디바이스 연결. 

- [logcat](https://developer.android.com/studio/command-line/logcat?hl=ko)  사용자 가이드를 읽어두자. 

```
$ adb devices
List of devices attached
LMX415Lc366ec0b offline
```

> 위와같이 adb devices 결과 offline 으로 보이면 안드로이드 개발자 옵션에서 USB 디버깅 항목을 다시 활성화 시킨 후 USB를 연결한다. 
> USB를 연결할 때 USB 디버깅 확인 팝업이 보여야 한다. 

```
$ adb devices
List of devices attached
LMX415Lc366ec0b device
```

> 위와 같이 디바이스 시리얼번호 와 device 로 나와야 logcat을 실행 할 수 있다. 


## 3. logcat 사용. 

```
$adb -s SERIALNO logcat 
```

> 연결된 디바이스가 1개이면 adb logcat 과 같이 단말의 시리얼번호를 명기하지 않아도 되다. 

## 4. mLogCat

> 윈도우즈 cmd 창이 익숙하지 않으면 mLogCat과 같은 GUI 툴을 사용할 수 있다. 

## 참고 URL 

> adb and mLogcat 설치 및 환경설정 참고. 

- [Minimal ADB and Fastboot](https://forum.xda-developers.com/showthread.php?t=2317790)
- [Minimal ADB and Fastboot Tool with mLogcat](https://www.utest.com/articles/android-log-capture-minimal-adb-and-fastboot-tool-with-mlogcat)  



