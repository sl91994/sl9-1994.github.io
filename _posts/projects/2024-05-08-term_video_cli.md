---
title: 「Term Video CLI」
description: Asciiアニメーション ターミナル描画ツール
date: 2024-05-08 12:00:00 +0900
last_modified_at: 2026-08-07 22:06:00 +0900
categories: [Development]
tags: [rust, ffmpeg, cli]
image:
  path: /assets/img/project_icon/term_video_cli_icon.png
  alt: term_video_cli_icon
---

<iframe width="560" height="315" src="https://www.youtube.com/embed/-fAEu1D2woU?si=zQWcM0CPVWXB0W5G" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

# TermVideoCli

- TermVideoCli は任意の動画ファイルを読み込み，ターミナルへ Ascii アニメーション表示することのできる CLI です．

#### [Youtube demo](https://youtu.be/-fAEu1D2woU)

## Tech Info

- 開発期間: 4 日
- 言語: Rust

### 機能

1. Youtube から動画をダウンロードし，ファイルを読み込む．
2. FFmpeg を使用し，動画を画像フレーム&&グレースケール変換・ターミナルサイズに合わせ画像をリサイズする．
3. フレームの明度ごとに文字を割り当て，繰り返し処理を使用してターミナルに表示する．

### 実装にあたってつまずいた部分

FFmpeg の Rust 用 Wrapper Crate を使用しようと思ったのですが，Build にそもそも失敗し，
結局，うまくいかなかったため Command::new で子プロセスとして FFmpeg を実行する方法に切り替えました．

再生中に，ノイズのようなものが表示されるのを解決できませんでした．

### 別の実装方法を考慮すべき点

24FPS 処理のため，3 分 40 秒の動画で 220s\*24f で 5200 枚程度の画像が生成される事になり，
ターミナルサイズが大きいほど解像度が高くなるため，綺麗に描画したい場合は，より容量が必要になります．

## ステータス

> completed

## History

- 2025/05/07: rusty_ytdl を依存関係から削除し，Youtube のダウンロード機能を廃止(-u オプション)
