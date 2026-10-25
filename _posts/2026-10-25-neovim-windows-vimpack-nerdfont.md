---
layout: post
comments: true
title: "Windows에서 Neovim 0.12 설정 다시 짜기 — Vundle에서 vim.pack으로, 한글 글꼴까지"
description: "2020년 이후 손대지 않은 Vundle 기반 .vimrc를 Neovim 0.12 내장 플러그인 관리자 vim.pack으로 옮기며 걸린 함정 여섯 가지, Neovide에서 GuiFont가 없는 이유, Nerd Font에 한글 폴백을 붙이는 네 가지 규칙을 정리한다."
img: neovim_vimpack_title.webp
date: 2026-10-25 14:30:00 +0900
last_modified_at: 2026-10-25 14:30:00 +0900
tags: [neovim, neovide, vim-pack, nerd-font, windows-terminal, markdown, editor, toolbox]
related: toolbox
categories: tools
---

Vundle로 굴러가던 `.vimrc`를 Neovim 0.12의 내장 플러그인 관리자 `vim.pack`으로 옮겼다. 설정 파일을 그대로 베껴 넣으면 시작부터 오류가 난다 — 없어진 옵션이 있고, 이사 간 저장소 주소가 남아 있고, 제어문자인 줄 알았던 글자가 그냥 글자인 경우도 있다. 여기에 터미널과 Neovide(GUI)에서 Nerd Font를 쓰면서 **한글만 다른 글꼴로 떨어뜨리는** 설정까지 붙였다. 이 글은 그 과정에서 실제로 걸린 것만 정리한다. (이 글은 Windows PC에서 수행한 설정 작업 기록을 Claude가 정리해 발행했다.)

<!--more-->

> **TL;DR:** Neovim 0.12에는 플러그인 관리자가 내장돼 있다(`vim.pack`). Vundle 설정을 옮길 때 `'pastetoggle'`·`:behave`·`cscopetag`는 Neovim에 아예 없어 시작 오류를 내고, `init.lua`와 `init.vim`을 같이 두면 E5422가 난다. Neovide에는 `GuiFont` 명령이 없다(그건 nvim-qt 전용) — `guifont` 옵션을 쓴다. 한글 폴백은 **크기를 마지막 항목에만** 쓰고 **DirectWrite 패밀리 이름**을 써야 한다. 틀린 이름은 오류 없이 조용히 무시된다.

## 왜 내장 vim.pack인가

플러그인이 열댓 개뿐인 구성이라 관리자를 하나 더 얹을 이유가 없었다. 후보들의 상태를 보면 선택이 쉬워진다 (GitHub API 조회, 2026-10-25 기준).

| 관리자 | ★ | 기본 브랜치 마지막 커밋 |
|---|---:|---|
| vim-plug | 35,781 | 2026-10-15 |
| Vundle.vim | 23,909 | **2024-07-30** |
| lazy.nvim | 21,672 | 2025-12-17 |

Vundle은 2년 넘게 멈춰 있다. 옮길 거라면 굳이 다른 외부 관리자로 갈아타기보다 **0.12에 들어온 `vim.pack`**이 낫다고 봤다. `add`/`del`/`get`/`update`로 끝나고, 락파일과 설치 경로를 Neovim이 관리한다.

## Vundle → vim.pack 이식 체크리스트

설정을 옮기며 실제로 걸린 여섯 가지다. 순서대로 밟으면 시작 오류 없이 넘어간다.

**① 없어진 옵션부터 걷어낸다.** `'pastetoggle'`, `:behave`, `cscopetag`/`cscopetagorder`, `:cscope`/`:lcscope`/`:scscope`는 Neovim에 **존재하지 않는다**. 공식 문서 `vim_diff.txt`의 "Vim에는 있지만 Neovim에는 없는 것" 목록에 올라 있다. `'pastetoggle'`은 붙임 처리가 자동이라 없앴고, cscope 계열은 통째로 빠졌다. 그대로 옮기면 첫 실행에서 오류가 쏟아진다.

<details class="evidence"><summary>원문 근거 (Neovim <code>vim_diff.txt</code>)</summary><blockquote>"- *'pastetoggle'* *'pt'* Just Paste It.™ |paste| is handled automatically when"</blockquote></details>

**② `mswin.vim`을 부르고 있었다면 경로를 확인한다.** `$VIMRUNTIME/mswin.vim`이 아니라 `scripts/mswin.vim`이고, 이 파일이 이미 `selection` 같은 mswin 동작을 설정한다. `:behave mswin`을 따로 부를 필요가 없다.

**③ 제어문자인 줄 알았던 것이 글자였을 수 있다.** F7/F8에 창 이동을 걸어 둔 매핑의 `^W`가 제어문자 0x17이 아니라 **캐럿(`^`)과 대문자 W 두 글자**였다. `grep -P '\x17'`로 확인하면 바로 드러난다. 처음부터 동작하지 않았는데 눈치채지 못한 채 그대로 들고 다닌 셈이다(이 `.vimrc`는 2020-02-07 이후 손대지 않은 파일이다). 옮기기 전에 한 번 훑어볼 값어치가 있다.

**④ 가드 변수를 설정값으로 착각하지 않는다.** `let g:tabular_loaded = 1`이 들어 있었는데, 이건 Tabular 플러그인의 **"이미 로드됨" 가드**다. 즉 Tabular가 한 번도 켜지지 않고 있었다. 지우니 `:Tabularize`가 생겼다. `*_loaded` 꼴 변수는 켜는 스위치가 아니라 **끄는 자물쇠**인 경우가 많다.

**⑤ `init.lua`와 `init.vim`을 함께 두면 안 된다.** 둘 다 있으면 `E5422: Conflicting configs`로 멈춘다. 옮기는 동안 참고하려면 `init.vim.bak`처럼 확장자를 바꿔 둔다.

**⑥ 저장소 주소를 현재 것으로 고친다.** 오래된 설정에는 이사 간 주소가 남아 있다.

```
scrooloose/nerdtree      → preservim/nerdtree
majutsushi/tagbar        → preservim/tagbar
plasticboy/vim-markdown  → preservim/vim-markdown
```

`vim-scripts/*` 미러를 쓰고 있었다면 원본 저장소(`preservim/nerdcommenter`, `jlanzarotta/bufexplorer`)로 바꾸는 편이 낫다.

한 가지 **확인이 더 필요한 지점**도 남겼다. `vim.pack.add`가 호출 시점에 플러그인을 로드한다고 보고, 플러그인이 읽는 `g:` 변수를 `add` 앞에 두는 구조로 작성했다. Vundle은 vimrc를 다 읽은 뒤 로드해서 순서 감각이 다르다. 이 부분은 문서로 한 번 더 확인하는 것을 권한다.

## Neovide에는 GuiFont가 없다

GUI로 Neovide를 붙이면서 가장 오래 헤맨 지점이다. 예전 `ginit.vim`에는 nvim-qt 시절에 쓰던 설정이 이렇게 남아 있었다.

```vim
if exists(':GuiFont')
    GuiFont! D2Coding:h12:l
endif
" GuiTabline 0 / GuiPopupmenu 0 / GuiScrollBar 1 도 같은 모양으로 가드가 씌워져 있었다

nnoremap <silent><RightMouse> :call GuiShowContextMenu()<CR>
inoremap <silent><RightMouse> <Esc>:call GuiShowContextMenu()<CR>
vnoremap <silent><RightMouse> :call GuiShowContextMenu()<CR>gv
```

**이 설정이 Neovide에서는 통째로 무용지물이다.** `GuiFont`·`GuiTabline`·`GuiPopupmenu`·`GuiScrollBar`는 **nvim-qt 전용 명령**이고 Neovide에는 없다. 실제로 `exists(':GuiFont')`을 물어보면 0이 돌아온다.

다행인 점과 아닌 점이 갈린다. 네 블록은 모두 `if exists(...)` 가드가 씌워져 있어 **조용히 건너뛰어진다** — 오류는 안 나지만 글꼴도 안 바뀐다(가드가 없었다면 `E492`가 났을 것이다). 반면 **우클릭 매핑 세 줄에는 가드가 없다.** 매핑 자체는 시작할 때 그대로 걸리는데 `GuiShowContextMenu` 함수가 Neovide에 없다(`exists('*GuiShowContextMenu')`이 0). 우클릭하면 `E117`(알 수 없는 함수)이 나게 된다 — 매핑과 함수 부재는 확인했고, 실제로 눌러 보는 시험까지는 하지 않았다.

Neovide는 평범한 **`guifont` 옵션**을 쓴다.

```vim
set guifont=MesloLGM\ Nerd\ Font,Paperlogy\ 4:h12
```

글꼴 이름에 공백이 있으면 `\ `로 이스케이프한다. nvim-qt의 `GuiFont!`도 같은 규칙이라 `GuiFont! MesloLGM\ Nerd\ Font:h12`처럼 써야 하는데, 어느 쪽이든 **Neovide에서는 `GuiFont` 자체가 없으므로** 이 줄을 고쳐 봐야 소용이 없다.

읽는 순서도 알아 둘 만하다. Neovide는 `ginit.vim`을 읽되 **`init.lua`보다 먼저** 읽는다(`:scriptnames`에서 ginit.vim이 1번, init.lua가 4번). 같은 옵션을 `init.lua`에서 다시 정하면 그쪽이 이긴다.

## 한글 폴백 네 가지 규칙

영문·숫자·아이콘은 Nerd Font로, **한글만 다른 글꼴로** 떨어뜨리고 싶었다. Meslo에 한글 글리프가 없으니 폴백이 필요하다. 규칙이 까다롭다.

**규칙 1 — 크기는 마지막 항목에만 쓴다.** 폴백은 쉼표로 잇는 목록이고 `:h12`는 끝에 한 번만 붙인다. 항목마다 쓰면 로그에 `Failed to parse guifont: Invalid size`가 찍히고 **목록 전체가 무효**가 되어 기본 글꼴로 돌아간다.

**규칙 2 — DirectWrite 패밀리 이름을 써야 한다.** 레지스트리나 GDI 기준 이름이 아니다. 페이퍼로지의 경우 `Paperlogy 4 Regular`로는 못 찾고 **`Paperlogy 4`**가 맞다. 이 글꼴은 굵기마다 패밀리가 따로라 `Paperlogy 1`~`9`로 나뉜다(4=Regular/Bold, 5=Medium/Bold).

**규칙 3 — 틀린 이름은 오류 없이 무시된다.** 주 글꼴을 못 찾으면 `Font can't be updated` 오류라도 나지만, **폴백 항목을 못 찾으면 조용히 넘어간다.** 설정은 멀쩡해 보이는데 한글만 그대로인 상태가 된다. 첫 시도가 정확히 이 경우였다.

**규칙 4 — 이름은 이렇게 확인한다.** Windows PowerShell 5.1에서,

```powershell
Add-Type -AssemblyName PresentationCore
[System.Windows.Media.Fonts]::SystemFontFamilies | ForEach-Object { $_.FamilyNames.Values }
```

한글이 든 `.ps1`을 만들 거라면 **BOM 포함 UTF-8**로 저장해야 5.1이 제대로 읽는다.

확인은 `neovide --log`로 파싱된 글꼴 목록을 직접 보는 게 가장 빠르다. 적용 전후로 창을 캡처해 픽셀 차이를 재 보면 영문 줄은 사실상 변화가 없고(평균 1.5) 한글 줄만 크게 바뀐다(59~64).

## 터미널 쪽도 같은 값으로

Windows Terminal은 설정에서 같은 조합을 넣는다.

```json
"face": "MesloLGM Nerd Font, Paperlogy 4"
```

`profiles.defaults`와 명시 프로필 **두 곳 모두** 넣어야 한다. airline의 파워라인 기호를 쓰려면 `init.lua`에 `vim.g.airline_powerline_fonts = 1`을 둔다.

**한계도 적어 둔다.** 격자(그리드) 기반 렌더링이 한글을 2칸으로 잡는데 Paperlogy는 비례 글꼴이라 자간이 넓어 보인다. 획이 굵고 둥글어지는 효과는 있지만 전후 차이가 극적이지는 않다. 고정폭 한글 글꼴을 쓰면 정렬은 깔끔해지는 대신 글꼴 선택지가 줄어든다.

## 마크다운 렌더링 — 컴파일러 없는 PC에서

편집 화면에서 마크다운을 그대로 렌더링하려고 후보를 추렸다 (2026-10-25 조회).

| 플러그인 | ★ | 기본 브랜치 마지막 커밋 |
|---|---:|---|
| markdown-preview.nvim | 8,021 | **2023-10-17** |
| render-markdown.nvim | 5,190 | 2026-10-10 |
| markview.nvim | 3,667 | 2026-10-18 |
| live-preview.nvim | 443 | 2026-10-22 |

인기만 보면 markdown-preview.nvim이지만 코드가 3년째 멈춰 있다. 편집 화면 안에서 렌더링하는 쪽으로 **render-markdown.nvim**을 골랐다.

여기서 환경 제약이 하나 걸렸다. **이 노트북에는 C 컴파일러가 없다.** 트리시터 파서를 새로 빌드할 수 없다는 뜻이다. 다행히 `markdown`·`markdown_inline` 파서는 Neovim 0.12에 내장돼 있어 핵심 기능은 그대로 돌아간다. 컴파일이 필요한 `html`·`latex`·`yaml` 기능만 끄면 된다.

```lua
require('render-markdown').setup({
  html  = { enabled = false },
  latex = { enabled = false },
  yaml  = { enabled = false },
})
```

이렇게 하니 `:checkhealth` 경고가 8개에서 1개로 줄었다(남은 하나는 아이콘 제공 플러그인 안내). 기존 vim-markdown과 함께 써도 충돌 목록에 없다. `:RenderMarkdown toggle`, `buf_toggle`, `preview`로 켜고 끈다.

## NERDTree와 CtrlP 북마크 잇기

디렉터리 북마크가 두 벌로 따로 논다는 걸 이번에 알았다.

| | 명령 | 저장 위치 |
|---|---|---|
| NERDTree | `:Bookmark`, 트리에서 `B` | `~/.NERDTreeBookmarks` |
| CtrlP | `:CtrlPBookmarkDir`, `:CtrlPBookmarkDirAdd` | `~/.cache/ctrlp/bkd/cache.txt` (이름`\t`경로) |

**서로 보이지 않는다.** NERDTree 북마크 표는 기본으로 숨겨져 있기도 하다(`g:NERDTreeShowBookmarks=0`).

쓰기 편하게 F9/F10에 CtrlP 북마크를 걸고, 작업 디렉터리가 바뀌면 **열려 있는** NERDTree만 따라가게 했다. 닫혀 있는데 새로 열지는 않는다.

```lua
vim.api.nvim_create_autocmd('DirChanged', {
  pattern = '*',
  callback = function()
    local ok, open = pcall(vim.fn.eval, 'exists("g:NERDTree") && g:NERDTree.IsOpen()')
    if not (ok and open == 1) then return end
    local win = vim.api.nvim_get_current_win()
    vim.cmd('NERDTreeCWD')
    if vim.api.nvim_win_is_valid(win) then vim.api.nvim_set_current_win(win) end
  end,
})
```

두 함정이 있다. NERDTree는 **존재하지 않는 경로가 섞인 북마크 파일을 읽으면 파일을 다시 써서 그 항목을 지운다** — CtrlP 쪽에서 옮겨 올 때 없는 폴더를 먼저 걸러야 한다. 그리고 `:CtrlPBookmarkDirAdd`는 `-n=?`로 정의돼 **디렉터리 인자 하나는 선택적으로 받지만 북마크 이름은 인자로 받지 못한다** — 이름은 늘 `input()` 프롬프트로 묻는다. 자동화 스크립트에서 부르면 그 지점에서 입력 대기로 멈춘다.

## 검증은 격리 환경에서

설정을 실제 환경에 바로 덮지 않고, `XDG_CONFIG_HOME`·`XDG_DATA_HOME`을 임시 폴더로 지정한 **격리 환경에서 먼저 설치·시작**했다. 이식 과정의 오류를 여기서 다 털고 실제 설정에 옮겼다. 수년 묵은 설정을 건드릴 때 특히 값어치가 있다.

GUI 쪽 상태를 확인할 때는 헤드리스 nvim에서 RPC로 붙어 조회했다.

```lua
local ch = vim.fn.sockconnect('pipe', '<Neovide 서버 주소>', { rpc = true })
-- exists(':GuiFont'), &guifont, :scriptnames 등을 조회
```

참고로 0.12에서 `nvim --server <주소> --remote-expr`는 값을 돌려주지 않고 TUI 클라이언트로 붙어 버렸다.

헤드리스 자동 시험에서 두 가지가 더 걸렸다. `VimEnter` 자동 명령 안의 `:edit`는 **filetype 감지가 되지 않는다** — 파일을 인자로 열어야 한다. 그리고 세션의 첫 `RenderMarkdown disable`이 반영되지 않는 현상을 봤는데, **원인은 확정하지 못했다.** 화면 갱신이 없는 환경 탓일 수 있다는 정도까지만 적어 둔다.

## 참고

- [Neovim `vim_diff.txt` — Vim과 다른 점](https://neovim.io/doc/user/vim_diff.html){:target="_blank"}
- [Neovide](https://github.com/neovide/neovide){:target="_blank"}
- [render-markdown.nvim](https://github.com/MeanderingProgrammer/render-markdown.nvim){:target="_blank"}
- [NERDTree](https://github.com/preservim/nerdtree){:target="_blank"} · [ctrlp.vim](https://github.com/ctrlpvim/ctrlp.vim){:target="_blank"} · [vim-markdown](https://github.com/preservim/vim-markdown){:target="_blank"}
- [Vundle.vim](https://github.com/VundleVim/Vundle.vim){:target="_blank"} · [vim-plug](https://github.com/junegunn/vim-plug){:target="_blank"} · [lazy.nvim](https://github.com/folke/lazy.nvim){:target="_blank"}
