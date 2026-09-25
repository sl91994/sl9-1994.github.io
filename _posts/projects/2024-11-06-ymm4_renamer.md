---
title: 「Ymm4 Renamer」
description: ゆっくりムービーメーカー4専用 動くイラストファイル名自動変更ツール
date: 2024-11-06 12:00:00 +0900
categories: [Development]
tags: [rust, cli]
image:
  path: /assets/img/project_placeholder_ai.png
  alt: project_placeholder
---

# Ymm4 Renamer

- ymm4_renamer は，ゆっくりまんじゅうなどの動く立ち絵を ymm4 に読み込める形式に変換する CLI です．

## TechInfo

- 実装言語: Rust
- 開発理由: 趣味で ymm4 を使用したゆっくり動画を制作しており，動くイラストの，ファイル名の変更がとても面倒だったため

| Crate      | Version |
| :----------: | :-------: |
| thiserror  | 1.0.66  |
| regex      | 1.11.1  |
| log        | 0.4.22  |
| env-logger | 0.11.5  |
| clap       | 4       |
| serde      | 1       |

> 具体的な使用方法は[README](https://github.com/SL9-1994/ymm4_renamer/blob/main/README.md)に記載しています．

### Points

- 詳細なログ表示と簡単なログレベルの調整
- exe ファイルのみで完結する簡単な導入
- 130ms の高速な変換  
  下記は PowerShell の Measure-Command を使用した計測

```powershell
Days              : 0
Hours             : 0
Minutes           : 0
Seconds           : 0
Milliseconds      : 129
Ticks             : 1294819
TotalDays         : 1.49863310185185E-06
TotalHours        : 3.59671944444444E-05
TotalMinutes      : 0.00215803166666667
TotalSeconds      : 0.1294819
TotalMilliseconds : 129.4819
```

### 対応フォーマット

1. 旧きつねゆっくり

## Status

> completed
