---
title: Daily-AlpacaHack 「duplicate entry」easy Writeup
description: 重複したエントリが存在する場合のZIPの挙動
date: 2026-07-07 06:35:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, misc]
---

# 20260706-daily_alpaca-misc-easy-duplicate_entry

## Summary

本問は，複数の同一ファイル名が存在する際のZIPの挙動を用いた問題です．

> - **Category**: Misc
> - **Description**: 解凍しても本物の flag.txt は得られません！　たぶん
> - **Tools & TechStack**:
> 	- Python
> 	- ZIP
> - **Release**: 2026/07/06
{: .prompt-info }

**階層構造**
```
.
├── flag.zip
└── make_zip.py

1 directory, 2 files
```

---
## ソースコードの調査

このプログラムは，`flag.zip` というアーカイブファイルを作成します．  
最初に `README` を書き込み，以降は `i = 0~99` までの100回 `flag.txt` という同名のファイルが別々のエントリとして内部に保存されています．  
また，`i == 50` では，`FLAG` が書き込まれますが，次のループで最新のエントリから外されてしまいます．  
そのため，どうにかして `i == 50` の状態のファイルを抽出する方法が必要です．

> **ZIPの仕様**[^1]: ZIP形式は，内部に **ファイル名のリスト (Central Directory)** と **実際のデータ (Local File Header)** を保持します．ZIPアーカイブ内に同じファイル名のデータが複数存在しても，ZIPの仕様上は問題ありませんが，`unzip` コマンドなどは，基本的には **最後に追加されたファイル** が優先的に読み込まれます．
{: .prompt-info }

`make_zip.py`
```python
import os, zipfile

FLAG = os.getenv("FLAG", "Alpaca{DUMMY}")

def writestr(z, name, data):
    info = zipfile.ZipInfo(name)
    z.writestr(info, data, compress_type=zipfile.ZIP_DEFLATED)

with zipfile.ZipFile("flag.zip", "w") as z:
    writestr(z, "README.txt", "The flag is in flag.txt.")
    for i in range(100):
        if i == 50:
            writestr(z, "flag.txt", FLAG)
        else:
            writestr(z, "flag.txt", "No flag here. Try harder.")
```

## 解法

まず，`i == 50` で書き込まれるファイルが，何番目に書き込まれるかのインデックスを考える必要があります．  
最初に `README.txt` が書き込まれ，次に `i = 0` からダミーファイルが順に追加されていきます．そのため，`i = 50` のファイルは **インデックス51 (52番目のファイル)** になることがわかります．  
また，`unzip -l` の結果をよく観察すると，他の `flag.txt` が 25バイト (ダミー文字列の長さ) であるのに対し，インデックス51のファイルだけが **32バイト** になっています．

```shell
$ unzip -l flag.zip
Archive:  flag.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
       24  01-01-1980 00:00   README.txt # Index 1
       25  01-01-1980 00:00   flag.txt
       25  01-01-1980 00:00   flag.txt
#...
       32  01-01-1980 00:00   flag.txt # 51番目かつ32byte
#...
       25  01-01-1980 00:00   flag.txt
       25  01-01-1980 00:00   flag.txt # Index 101
---------                     -------
     2531                     101 files
```

`z.infolist()` を使用すると，ZIP内に保存されている全ファイルのメタ情報 (`Central Directory`) がリスト形式で取得できます．このリストのインデックスは追加された順序に対応しているため，これを利用して抽出します．

`extract.py`
```python
import zipfile

with zipfile.ZipFile("flag.zip", "r") as z:
    info_list = z.infolist()
    
    target_index = 51  # README(0) + i=0(1) ... + i=50(51)
    
    target_info = info_list[target_index]
    
    with z.open(target_info) as f:
        content = f.read()
        print(f"Index {target_index} ({target_info.filename}): {content.decode()}")
```

---
## Post-Mortem & Dead ends

- `unzip -l`:  **インデックス表をそのまま出力** するオプション

## References

[^1]: [ZIP (ファイルフォーマット) - Wikipedia](https://ja.wikipedia.org/wiki/ZIP_(%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%83%95%E3%82%A9%E3%83%BC%E3%83%9E%E3%83%83%E3%83%88))
