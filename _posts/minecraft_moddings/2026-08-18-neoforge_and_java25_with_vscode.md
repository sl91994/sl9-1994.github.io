---
title: VScodeを用いたMinecraft v26.1.2 NeoForge 開発環境構築
date: 2026-08-18 13:15:00 +0900
description: Minecraft v26.1.2 NeoForge
categories: [MinecraftModding]
tags: [neoforge_26.1.2]
---

# ENV-neoforge_mdk_and_java25_with_vscode

Windows環境下で，VScodeを用いた `v26.1.2` のマインクラフトMOD開発環境を構築します．

## Step by Step

### `Scoop` のインストール

`Scoop`[^1] と呼ばれるWindowsのコマンドラインインストーラを導入します．

```powershell
PS C:\> Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
PS C:\> Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
Initializing...
Downloading...
Creating shim...
Adding ~\scoop\shims to your path.
Scoop was installed successfully!
Type 'scoop help' for instructions.
```

### VScodeの拡張機能の導入

`Extention Pack for Java`[^2] をインストールしておきます．

### MDKの導入

`NeoForge Mod Generator`[^3] を用いて，MDKとひな型をセットアップします．  
このとき，`Gradle Plugin` は，`ModDevGradle` にしておきます．

### 動作チェック

生成されたzipファイルを任意の階層で解凍し，VScodeで開きます．

```powershell
PS C:\> .\gradlew build
#...
BUILD SUCCESSFUL in 2m 8s
5 actionable tasks: 5 executed
Configuration cache entry stored.

PS C:\> .\gradlew runClient
```

無事にスタート画面が表示され，設定したMODのステータスが `Loaded` になっていれば環境構築の完了です．

## References

[^1]: [Scoop](https://scoop.sh)

[^2]: [Extension Pack for Java](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-java-pack)

[^3]: [Mod Generator - The NeoForged project](https://neoforged.net/mod-generator/)
