---
title: 葬送のフリーレンの杖をモデリングしてみた
description: Blockbench と Shape Generator プラグインで「葬送のフリーレン」の杖をMinecraft用3Dモデルとして作成．1.21.4以降で必要になった items/ のjsonの注意点も解説
date: 2025-08-22 12:00:00 +0900
categories: [MinecraftModding]
tags: [modeling]
---

## Blockbench の導入

[Blockbench](https://www.blockbench.net)からアーキテクチャに合致するインストーラをダウンロードして実行します．

## Shape Generator プラグインの導入

`File>プラグイン>Shape Generator(dragonmaster95)`をインストールしてください．  
マイクラでの通常モデリングでは，ブロックを組み合わせてモデルを構築していきます．  
また，ブロックには**回転角の制限**がかかっており，**22.5 度**ずつしかブロックを回転させることができません．
この制限の上で球・円や曲面を作るためにはブロックを 22.5 度ずつ回転させて角を繋げることで，曲面を作ることができます．  
Shape Generator プラグインはこれを簡単に生成することを可能にする便利なユーティリティです．  
下記のフリーレンの杖でも，このプラグインを利用して曲面を作成しています．

`Tools>Generate Shape`から使用でき 16 角形と 8 角形が選択できます．
球を作る場合は，生成されたドーナツ形を複製しながら軸方向に回転させることで作ることができます．

## 葬送のフリーレンの杖を作ってみた

Blender 等で作るよりは遙かに簡単にモデリングできます．
作成したモデルの texture 画像と json を Mod に組み込むことで使用できます．

> **1.21.4 以降でのカスタム 3D モデル読み込みの注意点**  
  今までのモデル読み込みでは，`assets/<mod_id>/models/item/item_id.json`, `assets/<mod_id>/texture/item_texture_id.png`のみで動作しましたが，1.21.4 以降?ではこれらのファイルに加えて，`assets/<mod_id>/items/item_id.json`が必要になりました．  
  そこでおすすめな json ジェネレータが以下です．  
  Type を Model にし，Model に`mod_id:item/item_id`を指定することで json が生成されます．
  生成された json を`assets/<mod_id>/items/`に配置することでモデルが動作します．<br>
  [item json generator](https://misode.github.io/assets/item/)
{: .prompt-danger }

![frieren_wand_01](/assets/img/minecraft_modding/minecraft_modeling_01_01.png)
