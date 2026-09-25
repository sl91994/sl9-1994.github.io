---
title: Daily-AlpacaHack 「private method」easy Writeup
description: PythonにおけるPrivateメソッドに対する名前修飾
date: 2026-07-10 07:16:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, misc]
---

# 20260709-daily_alpaca-misc-easy-private_method

## Summary

本問は，PythonでのPrivateメソッドに対する名前修飾を用いる問題です．

> - **Category**: Misc
> - **Description**: privateなメソッドは呼び出せないはず...
> - **Tools & TechStack**:
> 	- Python
> - **Release**: 2026/07/09
{: .prompt-info }

**階層構造**
```
.
├── app.py
├── compose.yaml
└── Dockerfile

1 directory, 3 files
```

---
## ソースコードの調査

`Main` クラスが定義されており，Publicなメソッド `alpaca` と，Privateなメソッド `__flag` が定義されています．
ここで，`Main` クラス以下に対して，任意の関数を呼び出すことができる実装になっています．

- **正規表現の解析**: `\w+\(\)`
	- `\w+`: 1文字以上の英数字・アンダースコアの連続にマッチ
	- `\(\)`: 括弧をエスケープしているため，文字列の `()` にマッチ

```python
import re

class Main:
    # public 
    def alpaca():
        return "🦙"

    # private
    def __flag():
        return "Alpaca{REDACTED}"

code = input(">>> Main.").strip()

if re.fullmatch(r"\w+\(\)", code):
    print(eval(f"Main.{code}"))
else:
    print("Nope")
```

## 名前修飾 (Name Mangling)

Pythonにおいて，メソッド名の接頭辞として `__` を付けることでPrivateメソッドになります．
これによって，外部からのアクセスが制限されます．
実際には，名前空間の衝突を避けるために名前修飾が行われ，内部でメソッド名が書き換えられます．

具体的には，クラス名の頭にアンダースコアを1つ付け，その後に元のプライベート変数やメソッド名を結合する形になります．

- **`Foo` クラスの `__Bar` メソッドの名前修飾の例**: `_Foo__Bar`

## 名前修飾を利用した関数実行

- **Payload:** `_Main__flag()`

```shell
$ python3 app.py
>>> Main._Main__flag()
Alpaca{REDACTED}
```

---
## Post-Mortem & Dead ends

## References
