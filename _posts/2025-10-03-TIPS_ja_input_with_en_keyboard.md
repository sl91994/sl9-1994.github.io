---
title: 英語配列でIME日本語入力を動作させる
description: Windows11 で日本語配列キーボードを英語配列 (101/102キー) として使い，Ctrl+Space で Microsoft IME の日本語入力を切り替える設定方法
date: 2025-10-03 12:00:00 +0900
categories: [Environment]
tags: [windows]
---


## 導入環境

- 物理キーボード: テンキーなし日本語配列(105, 106)
- OS: Windows11 home

## キーマッピングの変更

`時刻と言語>言語と地域`から日本語パッケージの右側にある三点リーダーから，`言語のオプション`を選択します．<br>

![ja_to_en_01](/assets/img/2025/how_to_setup_ja_input_with_en_Keyboard_03.png)

下にあるキーボードレイアウトから，`英語キーボード(101/102キー)`を選択し，デバイスを再起動します．<br>

![ja_to_en_02](/assets/img/2025/how_to_setup_ja_input_with_en_Keyboard_04.png)

次に，`時刻と言語>入力>キーボードの詳細設定`の，入力言語のホットキーから，`キーシーケンスの変更`を行い，割り当てを無効化してください．

![ja_to_en_03](/assets/img/2025/how_to_setup_ja_input_with_en_Keyboard_05.png)


最後に，`時刻と言語>言語と地域>Microsoft IME>キーとタッチとカスタマイズ`から，ctrl + spaceに`IME-オン/オフ`を設定すれば完了です．

![ja_to_en_04](/assets/img/2025/how_to_setup_ja_input_with_en_Keyboard_01.png)

> ※ このIME切り替えを行うためには，ctrl+iで設定を開き，`時刻と言語>言語と地域>Microsoft IME>全般`から下記の画像のように最新バージョンのIMEを使用する必要があります．
> **2026/04/12 追記**: 過去のIMEでも，`詳細設定>編集操作>キー設定` から，好きなキーの `入力/変換済み文字なし` に `IME-オン/オフ`　を設定すれば可能です．

![ja_to_en_05](/assets/img/2025/how_to_setup_ja_input_with_en_Keyboard_02.png)
