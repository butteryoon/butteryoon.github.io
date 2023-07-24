---
layout: post
title: "Windwos10에서 gvim 컬러스킴 변경하기" 
description: "Windows 10 환경에서 gvim의 컬러스킴을 변경하는 방법을 기록해둔다."
img: vim-title.webp
date: 2021-01-14 17:00:00 +0900
last_modified_at: 2021-01-14 17:00:00 +0900
tags: [devtool, vim, plugin] # add tag
related: devtool
categories: tools
---

Windows 10 환경에서 vim의 컬러스킴을 변경하는 방법을 알아본다. 

<!--more-->

## 컬러스킴 설치.


[monokai-pro](https://github.com/phanviet/vim-monokai-pro) 컬러스킴을 추가하려고 한다.  

요즘은 "[Vundle](https://github.com/VundleVim/Vundle.vim)" 대신 "[vim-plug](https://github.com/junegunn/vim-plug)"를 추천하지만 귀차니즘으로 그냥 버티고 있다. (언젠간 바꿔야지.. )


vim의 플러그인 관리자로 "Vundle"을 쓰고 있어 아래와 같이 플러그인을 추가하고 ":VundleInstall" 명령어로 설치한다.   

```bash
Plugin 'phanviet/vim-monokai-pro' 

set termguicolors
colorscheme monokai_pro

let g:lightline = {
      \ 'colorscheme': 'monokai_pro',
      \ }
```

## 컬러스킴 파일 복사 


Windows 10 환경에서 Vundle로 컬러스킴을 추가하면 지정된 Vundle 홈 디렉토리 아래 플러그인이 설치되며 보통 colors 디렉토리 아래 colorschmem_name.vim 파일이 있다. 

chocolatey 패키지 관리자의 기본설정을 vim을 설치하면 아래의 디렉토리에 설치된다. 

```powershell
⚡ softr@YOON-IP700  C:\tools\vim\vim82\colors                                                          [17:28]
❯ pwd

Path
----
C:\tools\vim\vim82\colors
```

설치한 컬러스킴 'monokai_pro/colors/monokai_pro.vim" 파일을 vim이 설치된 디렉토리의 colors 디렉토리로 복사해준다. 


## 참고 URL
-  [How To Change And Use Vim Color Schemes](https://phoenixnap.com/kb/vim-color-schemes){:target="_blank"}
