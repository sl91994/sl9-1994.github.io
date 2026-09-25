---
title: Daily-AlpacaHack 「Catrunner」Easy
description: Pythonの仕様を利用したPath Traversal
date: 2026-05-25 20:03:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, linux, misc, vuln/path_traversal]
---

# 20260525-daily_alpaca-misc-Catrunner

> - **Category**: misc
> - **Description**: Path Traversal
> - **Tech Stack**: Python
{: .prompt-info }

**階層構造**
```
.
├── compose.yaml
├── Dockerfile
├── flag.txt
├── hello.txt
└── jail.py

1 directory, 5 files
```

---
## Solution Path

`Dockerfile` から，`flag.txt` が存在するディレクトリはルートであることが分かりました．

`Dockerfile`
```Dockerfile
#...
COPY jail.py hello.txt ./
COPY flag.txt /
#...
```

このプログラムは，入力されたパスのファイルを **cat** する機能を持ちます．
また，パストラバーサルを防ぐため，1つ上の階層を示す `..` が含まれるパスの場合に弾きます．

`jail.py`
```python
import os

filename = input("Example: hello.txt\n$ cat /app/")
assert ".." not in filename, "Path traversal is not allowed"
path = os.path.join("/app", filename)
if os.path.isfile(path):
    os.system(f"cat {path}")
else:
    print("File not found")
```

### `os.path.join` の仕様を利用した **PathTraversal**

`os.path.join("/app", "/A")` が実行されるとします．
Pythonの仕様上，2つ目の引数が絶対パスである場合，それ以前のパス (`/app`) は破棄され，結果は `/A` になります． 
これにより，`..` を一切使わずにパストラバーサルが成立します．

### Exploitation

```shell
$ nc <ip> <port>
Example: hello.txt
$ cat /app//flag.txt 
{*** REDACTED ***}
```

---
## Post-Mortem & Dead ends

- `os.path.join`の仕様による絶対パス指定でのパストラバーサル:
  2つ目の引数が絶対パスである場合，それ以前のパスは破棄され，結果は2つ目の引数のパスになります． 

## References

- [os.path --- 一般的なパス名操作 — Python 3.14.5 ドキュメント](https://docs.python.org/ja/3/library/os.path.html#os.path.join)