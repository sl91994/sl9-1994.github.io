---
title: Daily-AlpacaHack 「strange filename」Easy Writeup
description: ファイル名がハイフンのみの特殊なファイルの表示
date: 2026-07-05 22:21:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, misc]
---

# 20260413-daily_alpaca-misc-easy-strange_filename

## Summary

本問は，Linuxにおいてファイル名がハイフンのみの特殊なファイルの表示をする問題です．

> [!info] Challenge Info
> - **Category**: Misc
> - **Description**: 変わった名前のファイルを表示できますか？
> - **Tools & TechStack**:
> 	- Python
> - **Release**: 2026/04/13
{: .prompt-info }

**階層構造**
```
.
├── compose.yaml
├── Dockerfile
├── flag.txt
└── server.py

1 directory, 4 files
```

---
## ソースコードの調査

`server.py` を見ると，最初に `ls` が実行されて，その後に任意のパスを渡すことが可能な `cat` が実行されることが分かりました．

`server.py`
```python
import subprocess

# list files
print("$ ls", flush=True)
subprocess.run(["ls"])

# cat a file, taking only one argument, without shell redirections
subprocess.run(["cat", input("$ cat ")])
```

また，実行してみると `Dockerfile` に記載されているのと同様に，`-` というファイルが表示されます．  
これが，問題名の **strange filename (奇妙なファイル名)** の理由だと思われます．  
しかし，`cat -` とすると標準入力をオウム返しされるだけです．

**実行例**
```shell
$ ls
-
server.py
#...
```

**`Dockerfile` の抜粋**
```dockerfile
# Copy the flag file with a strange filename
COPY flag.txt -
```

## 解法

`cat` コマンドのドキュメント[^1] を見ると **With no FILE, or when FILE is -, read standard input.** と書かれていました．  
そのため，`cat -` をすると，標準入力がそのまま標準出力されていたようです．  
そこで，`-` をファイルとして指定するために，明示的にカレントディレクトリを指定して，`cat ./-` とすることで **Flag** を見ることができました．

```shell
$ nc localhost 1337
$ ls
-
server.py
$ cat ./-
Alpaca{REDACTED}
```

---
## Post-Mortem & Dead ends

`cat -- -` でもいけると思ったんだけど，`cat` のバージョンやパッケージの差なのかな...？

## References

[^1]: [cat(1) - Linux manual page](https://www.man7.org/linux/man-pages/man1/cat.1.html)
