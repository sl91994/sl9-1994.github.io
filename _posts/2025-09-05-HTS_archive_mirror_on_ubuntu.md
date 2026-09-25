---
title: WSL2 ubuntu 24.04のアーカイブミラーサーバーを変更
description: WSL2 の Ubuntu 24.04 で apt が公式サーバーに接続できなくなったときに，ubuntu.sources を書き換えて国内ミラーサーバーへ変更する方法
date: 2025-09-05 12:00:00 +0900
categories: [Environment]
tags: [linux, wsl]
---

## 変更理由

ネットワーク設定を変更していないにも拘らず，apt での`security.ubuntu.com, archives.ubuntu.com`等の公式サーバーへアクセスが出来なくなった．<br>
ipv6 の無効化やファイアウォールの確認などを行ったが，改善しなかった．

## [ミラーサーバー](https://launchpad.net/ubuntu/+archivemirrors)へ変更

`ubuntu 24.04`から apt リポジトリの設定は`/etc/apt/sources.list`ではなく，`/etc/apt/sources.list.d/ubuntu.sources`で行う．<br>
今回は最も速い[ICSCoE](https://launchpad.net/ubuntu/+mirror/ftp.udx.icscoe.jp-archive)に変更します．<br>

> 書き換える前に必ず，バックアップを取ってください．
{: .prompt-warning }

- /etc/apt/sources.list.d/ubuntu.sources

```conf
## See the sources.list(5) manual page for further settings.
Types: deb
URIs: [このURLを書き換える]
Suites: noble noble-updates noble-backports
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg

## Ubuntu security updates. Aside from URIs and Suites,
## this should mirror your choices in the previous section.
Types: deb
URIs: [このURLを書き換える]
Suites: noble-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

これを適用すると，正常に apt が使用できるようになります．
