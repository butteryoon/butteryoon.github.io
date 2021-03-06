---
layout: post
title: "Android Studio 없이 adb 사용하기"
img: android-adb.jpg
date: 2019-01-30 18:38:00 +0900
last_modified_at: 2021-01-22 14:00:00 +0900
tags: [android, 안드로이드, adb] # add tag
related: android
categories: dev
---

윈도우즈 환경에서 adb를 사용하려면 그리 간단하지 않다. 

안드로이드 스튜디오 및 SDK를 설치해야 하고 특히 내 노트북과 같이 성능이 좋지 않은 환경에서는 안드로이드 스튜디오를 돌리는 것도 버벅대는 일도 있고 환경설정이 바뀌거나 하면 실행이 되지 않는다. 

Android Studio가 설치되어 있는 개발자 환경이 아니고 adb로 안드로이드 디바이스 모니터링만 하는 경우 안드로이드 스튜디오 및 SDK를 설치하지 않고 명령어라인에서 USB 연결로 ADB를 사용하기 위한 방법을 정리해본다. 


## 1. adb 명령어 라인 툴 설치 

[Minimal ADB and Fastboot](https://forum.xda-developers.com/showthread.php?t=2317790) 에서 [minimal_adb_fastboot_1.4.3_portable.zip](https://androidfilehost.com/?fid=962187416754459552) 다운로드 후 특정폴더에 압축을 풀면 아래와 같이 adb와 dll이 보인다. 

> 설치된 디렉토리로 이동하여 쓸 수도 있지만  "제어판\시스템 및 보안\시스템 > 고급 시스템 설정 > 환경변수 에 ADB_PATH를 추가하고 Path 사용자변수에 추가하면 편리하다.
> Windows 10의 경우 왼쪽 하단 검색툴에서 "시스템 환경 변수 편집"으로 검색한다. 

```powershell
$ ls
adb.exe*  AdbWinApi.dll*  AdbWinUsbApi.dll*  cmd-here.exe*  Disclaimer.txt  fastboot.exe*
```

## 1-1. 제조자 USB 드라이버 설치 

제조사별 드라이버를 설치하면 Android 기기를 연결했을 때 장치관리자에 "Android Device"가 표시되어야 한다. 

아래 제조사별 USB 드라이버룰 설치한다. 

 - [LG](https://www.mylgphones.com/lg-android-usb-device-drivers) 
 - [SAMSUNG](https://developer.samsung.com/mobile/file/68b2dc40-3833-4a8b-b58e-32f7aca25c00)
 - [GOOGLE](developer.android.com/studio/run/win-usb?hl=ko) 

## 2. adb 안드로이드 디바이스 연결. 

- [logcat](https://developer.android.com/studio/command-line/logcat?hl=ko) 사용자 가이드를 읽어두자. 

```powershell
$ adb devices
List of devices attached
LMX415Lc366ec0b offline
```

위와같이 adb devices 결과 offline 으로 보이면 안드로이드 개발자 옵션에서 USB 디버깅 항목을 다시 활성화 시킨 후 USB를 연결한다. 
USB를 연결할 때 USB 디버깅 확인 팝업이 보여야 한다. 

- 안드로이드 단말에서 "개발자 옵션"이 활성화되지 않으면 아래와 같이 offline 으로 표시된다. 
- 그러면 "설정 > 일반 > 휴대폰 정보 > 소프트웨어 정보"에서  "빌드 번호"를 7번정도 연속해서 눌러준다. 


```powershell
$ adb devices
List of devices attached
LMX415Lc366ec0b device
```

위와 같이 디바이스 시리얼번호 와 device 로 나와야 logcat을 실행 할 수 있다. 
준비 끝. 


## 3. logcat 사용. 

```powershell
$ adb -s SERIALNO logcat 
$ adb -s LMX415Lc366ec0b logcat 
```

> 연결된 디바이스가 1개이면 adb logcat 과 같이 단말의 시리얼번호를 명기하지 않아도 된다. 
> 디바이스가 여러개일 경우  "-s 시리얼번호"와 같이 특정 디바이스를 선택한다. 

## 4. adb Server 연결 실패

> 아래와 같이 ADB Server 연결이 안되는 경우 kill/start-server 명령어로 재시작 한다.  

```powershell
adb devices
List of devices attached
* daemon not running; starting now at tcp:5037
could not read ok from ADB Server
* failed to start daemon
error: cannot connect to daemon 

adb kill-Server
adb start-server
```

- 간혹 로컬 프로세스가 5037 포트를 점유하고 있는 경우가 있는데 이런 경우 해당 프로세스를 죽인 후 다시 시작한다. 
- adb 서버의 포트를 변경하는 방법은 확인 필요. 


## 5. mLogCat

> 윈도우즈 cmd 창이 익숙하지 않으면 [mLogCat](https://mlogcat.tistory.com)과 같은 GUI 툴을 사용할 수 있다. 


## 6. WiFi Connect 

> 동일 WiFi 환경에서 무선으로 logcat 사용하기 

### 1. 안드로이드 단말과 USB가 연결된 상태에서 디버깅할 안드로이드 단말의 TCP포트를 설정한다. 

```powershell
> adb -s R33M100M79 tcpip 8888
     restarting in TCP mode port: 8888
```
### 2. 안드로이드 단말에서 USB 연결 디버깅 허용 팝업이 표시되면 확인을 선택한다. 

### 3. 안드로이드 WiFi IP주소 확인 

```powershell 
❯ adb shell ip addr show wlan0
36: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 3000
    link/ether 7c:f3:1b:b6:99:67 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.97/24 brd 192.168.0.255 scope global wlan0
      valid_lft forever preferred_lft forever
    inet6 fe80::7ef3:1bff:feb6:9967/64 scope link
      valid_lft forever preferred_lft forever
```

### 4. USB 연결 제거 후 "adb connect" 명령어로 안드로이드 단말의 Adb서버에 연결한다.  

```powershell
> adb connect 192.168.0.47:8888
     connected to 192.168.0.47:8888
```

### 5. "adb devices" 명령어로 연결 정보를 확인한다.  

```powershell
❯ adb devices
List of devices attached
192.168.0.97:8803       device
``` 

## 7. APK Install 

"adb install/uninstall" 명령어로 패키지를 설치/삭제할 수 있다. 

```powershell
"adb uninstall com.samples.vdc"
"adb install COM.SAMPLES.VDC.apk"
```

세부 옵션은 아래와 같다. 

```powershell
app installation:
 install [-lrtsdg] PACKAGE
 install-multiple [-lrtsdpg] PACKAGE...
     push package(s) to the device and install them
     -l: forward lock application
     -r: replace existing application
     -t: allow test packages
     -s: install application on sdcard
     -d: allow version code downgrade (debuggable packages only)
     -p: partial application install (install-multiple only)
     -g: grant all runtime permissions
 uninstall [-k] PACKAGE
     remove this app package from the device
     '-k': keep the data and cache directories
```

### 7.1 APK 파일 버전 정보 확인하기 

"adb shell dumpsys" 명령어를 이용해 apk 파일의 정보를 확인하고 "Select-String" 명령어로 "version" 패턴을 검색한다. 

```powershell
❯ adb shell dumpsys package com.samples.vdc | Select-String -Pattern version

    versionCode=35 minSdk=24 targetSdk=24
    versionName=00.05.22
    apkSigningVersion=1
```

## 화면 캡쳐 

"adb shell screencap" 파라미터로 화면 캡쳐하여 지정된 위치에 이미지를 저장한다. 

Windows에서는 capture.cmd 파일을 만들고 아래의 명령어를 입력하면 "캡쳐 - 다운로드 - 삭제" 과정이 순서대로 진행된다. 

```powershell
adb shell screencap -p /sdcard/screen.png
adb pull /sdcard/screen.png
adb shell rm /sdcard/screen.png
```

## 파일 업로드 

"adb push 파일이름 경로"명령어로 파일을 옮길 수 있다. 

```powershell
❯ adb push samples.apk /sdcard/Download
samples.apk: 1 file pushed. 34.1 MB/s (31098455 bytes in 0.869s)
```

## 참고 URL 

> adb and mLogcat 설치 및 환경설정 참고.  

- [Minimal ADB and Fastboot](https://forum.xda-developers.com/showthread.php?t=2317790)
- [Minimal ADB and Fastboot Tool with mLogcat](https://www.utest.com/articles/android-log-capture-minimal-adb-and-fastboot-tool-with-mlogcat)  
- [Universal ADB Drivers](https://adb.clockworkmod.com)




