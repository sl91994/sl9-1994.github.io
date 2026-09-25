---
title: WSL2上のNeovimで，コピー&ペーストができない問題
description: WSL2 上の Neovim で Windows とクリップボードを共有できない問題を，win32yank を導入して解決する方法
date: 2025-10-06 12:00:00 +0900
categories: [Environment]
tags: [neovim, wsl]
---

## [解決策](https://github.com/neovide/neovide/issues/544#issuecomment-820519937)

```shell
$ curl -sLo /tmp/win32yank.zip https://github.com/equalsraf/win32yank/releases/download/v0.1.1/win32yank-x64.zip
$ unzip /tmp/win32yank.zip -d /tmp/
$ sudo cp /tmp/win32yank.exe /usr/local/bin/
$ sudo chmod +x /usr/local/bin/win32yank.exe
$ which win32yank.exe
/usr/local/bin/win32yank.exe
```

**~/.config/nvim/lua/config/wsl_clipboard.lua**
```lua
vim.opt.clipboard = "unnamedplus"
```