---
layout: post
title: "ImageMagick CLI 명령어 사용하기"
description: "Windows Terminal 환경에서 imagemagick을 설치하고 CLI 명령어를 이용해서 이미지를 변환하는 방법을 알아본다."
img: imagemagick_title_crop.webp
date: 2021-04-22 18:00:00 +0900
last_modified_at: 2021-05-21 01:00:00 +0900
tags: [Windows10, imagemagick, convert, magick] # add tag
related: Windows10
categories: tools
---

Windows10 에서 [imagemagick](https://imagemagick.org/script/index.php)을 설치하고 CLI 명령어를 이용해서 이미지를 변환하는 방법을 알아본다.

블로그에 사용하기 위해 [unsplash](https://unsplash.com)에서 받은 고해상도 이미지의 크기를 줄이거나 위/아래를 기준으로 잘라내고 싶을 때 사용할 수 있다. 

imagemagick의 [Basic Usage](https://legacy.imagemagick.org/Usage/basics)를 참고한다. 

<!--more-->

## imagemagick 설치

"Chocolatey" 패키지관리자 또는 [공식 웹페이지](https://imagemagick.org/script/download.php#windows)에서 다운로드하여 설치할 수 있다. 

"Chocolatey" 에서 "imagemagick" 패키지를 검색하면 여러가지가 나오는데 "--version" 옵션으로 특정 버전을 설치한다. 

```powershell
❯ choco search imagemagick
Chocolatey v0.10.15
imagemagick 7.0.11.8 [Approved]
imagemagick.tool 7.0.11.8 [Approved] Downloads cached for licensed users
imagemagick.app 7.0.11.8 [Approved]
graphicsmagick-q16 1.3.35 [Approved] Downloads cached for licensed users
graphicsmagick 1.3.32 [Approved] Downloads cached for licensed users
amt 11.0 [Approved] Downloads cached for licensed users
6 packages found. 

❯ choco install imagemagick --version 7.0.11.8
```

설치 완료 후 magick.exe 버전을 확인한다. 

```powershell
❯ & 'C:\Program Files\ImageMagick-7.0.11-Q16-HDRI\magick.exe' -version
Version: ImageMagick 7.0.11-8 Q16 x64 2021-04-17 https://imagemagick.org
Copyright: (C) 1999-2021 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Visual C++: 192829913
Features: Cipher DPC HDRI Modules OpenCL OpenMP(2.0)
Delegates (built-in): bzlib cairo flif freetype gslib heic jng jp2 jpeg jxl lcms lqr lzma openexr pangocairo png ps raw rsvg tiff webp xml zip zlib
```

### 실행파일 alias 설정 

터미널 환경에서 "magick"만 사용할거라 환경변수에 "path"에 추가하는 대신 "PowerShell"에서 Alias를 설정하기로 한다. 

> $PROFILE을 편집기로 열고 Set-Alias로 magick.exe의 경로를 지정한다. 

```powershell
> code $PROFILE 

... (add alias)
Set-Alias magick -Value 'C:\Program Files\ImageMagick-7.0.11-Q16-HDRI\magick.exe' 
... (save)

> . $PROFILE 
```

## 이미지를 svg로 변환 

jpg 또는 png 이미지를 지정된 크기의 svg 이미지로 변환하고 변환된 파일의 정보를 확인해 본다. magick는 확장자를 보고 이미지의 포맷을 결정한다. 

```powershell
> magick .\me.jpg -resize 100x100 .\me.svg

❯ magick identify me.jpg
me.jpg JPEG 155x108 155x108+0+0 8-bit sRGB 6843B 0.000u 0:00.000

❯ magick identify .\me.svg
.\me.svg SVG 100x70 100x70+0+0 16-bit sRGB 20732B 0.016u 0:00.012
```

## 이미지 리사이즈 후 자르기

고해상도 이미지를 특정 사이즈로 변경 하고 이미지의 가운데를 기준으로 아래쪽으로 100픽셀을 잘라낸다. 이미지 사이즈를 줄이기위해 이미지 포맷은 webp로 변환한다. 

*이 페이지의 타이틀 이미지를 아래의 옵션으로 crop 해서 잘랐다.*

> -resize 900x600  
> -gravity Center  
> -crop 900x600-0-100  
> 변환 이미지 포맷은 확장자로 정해진다.         

```powershell
❯ magick .\almos-bechtold-AJ_Mou1FUS8-unsplash.jpg -resize 900x600 -gravity Center -crop 900x600-0-100 imagemagick_title_crop.webp
 
❯ magick identify .\almos-bechtold-AJ_Mou1FUS8-unsplash.jpg
.\almos-bechtold-AJ_Mou1FUS8-unsplash.jpg JPEG 1920x1280 1920x1280+0+0 8-bit sRGB 428917B 0.000u 0:00.000
 
❯ magick identify .\imagemagick_title_crop.webp
.\imagemagick_title_crop.webp WEBP 900x500 900x500+0+0 8-bit sRGB 25492B 0.000u 0:00.008
```

## 이미지 자르기 기준 

이미지를 왼쪽/오른쪽 또는 위/아래 기준으로 자르고 싶을 때 아래와 같이 "-crop" 옵션을 사용한다. 

> -crop -500-0 : 오른쪽 500픽셀 자름.   
> -crop +500-0 : 왼쪽 500픽셀 자름. 

```powershell
❯ magick identify .\main_screen.png
.\main_screen.png PNG 2400x1080 2400x1080+0+0 8-bit sRGB 2.00877MiB 0.000u 0:00.000

❯ magick .\main_screen.png -gravity center -crop -500-0 m5_main_screen.png
❯ magick .\main_screen.png -gravity center -crop +500-0 m6_main_screen.png
```

![magick-crop]({{site.baseurl}}/assets/img/m_imagemagick_crop.webp){:width="800px"}

## 명령어 결과 이미지로 저장 

윈도우즈 터미널에서 파워쉘 명령어의 결과를 이미지로 저장하고 싶을 때는 아래와 같이 명령어의 결과를 "Out-String"로 변환하여 magick로 보낸다. 

*magick -font D2Coding label:@- cmdlet-img.png*

```powershell
❯ Get-Process | Where-Object {$_.mainWindowTItle} |format-table id,name,mainwindowtitle | Out-String | magick -font D2Coding label:@- cmdlet-img.png
❯ magick identify .\cmdlet-img.png
.\cmdlet-img.png PNG 685x239 685x239+0+0 8-bit Gray 256c 12973B 0.000u 0:00.000
```

![magick-textimg]({{site.baseurl}}/assets/img/cmdlet-img.png){:width="800px"}

## WSL Ubuntu 20.04

WSL 환경에서는 apt 명령어로 설치하고 아래와 같이 **"convert"** 명령어를 사용한다. 

```bash
sudo apt update
sudo apt-get install build-essential
sudo apt install imagemagick
```

## Convert png to svg

```bash
❯ convert -resize 256x256 me_profile.png me_profile.svg

❯ file me_profile.svg
me_profile.svg: SVG Scalable Vector Graphics image
```

## Save command output to an image

명령어의 결과를 이미지로 만들 수 있는데 아래와 같은 오류가 생기면 **"/etc/ImageMagick-6/policy.xml"** 에서 아래의 정책을 주석처리 한다. 

*윈도우즈에서는 기본으로 주석처리 되어 있다*

```bash
❯ sudo vi /etc/ImageMagick-6/policy.xml
...(skip)
<!-- <policy domain="path" rights="none" pattern="@*"/> -->
...(skip)
```

```bash
❯ sudo ps aux | convert label:@- process.png
convert-im6.q16: attempt to perform an operation not allowed by the security policy `@-' @ error/property.c/InterpretImageProperties/3666.
convert-im6.q16: no images defined `process.png' @ error/convert.c/ConvertImageCommand/3258.
```

## 참고 URL

- [Install ImageMagick in Ubuntu 20.04 LTS](https://techpiezo.com/linux/install-imagemagick-in-ubuntu-20-04-lts/){:target="_blank"}
- [How to Install ImageMagick 7 on Debian and Ubuntu](https://www.tecmint.com/install-imagemagick-on-debian-ubuntu/){:target="_blank"}
- [How to convert a SVG to a PNG with ImageMagick?](https://stackoverflow.com/questions/9853325/how-to-convert-a-svg-to-a-png-with-imagemagick){:target="_blank"}
- [Anatomy of the Command-line](https://imagemagick.org/script/command-line-processing.php){:target="_blank"}
- [Save Linux Command Output To An Image Or A Text File](https://ostechnix.com/save-linux-command-output-image-file/){:target="_blank"}
