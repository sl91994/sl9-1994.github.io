---
title: Daily-AlpacaHack 「True or False」Medium Writeup
description: 例外処理によるオラクル実装と真偽値の算術評価を利用して，3分探索することでFlagを入手するMisc問題
date: 2026-07-24 00:01:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, misc]
math: true
---

# 20260722-daily_alpaca-misc-true_or_false

## Summary

本問は，例外処理によるオラクル実装と，真偽値の算術評価を利用して，巨大数を3分探索することでFlagを入手するMisc問題です．

> - **Category**: Misc
> - **Description**: 全ては二元論に通じます。陰と陽、昼と夜、真と偽。
> - **Tools & TechStack**:
> 	- Python
> 	- Pwntools
> - **Release**: 2026/07/23
{: .prompt-info }

**階層構造**
```
.
├── chal.py
├── compose.yaml
└── Dockerfile

1 directory, 3 files
```

---
## ソースコードの調査

このプログラムは最初に $0$ 以上 $2 ^ {44} - 1 = 17592186044415 \ (14桁)$ の乱数 `a` を生成しています．  
その後，$0 \le i \le 27$ の範囲で28回のループを行い，`ALLOWED_CHARS` に含まれる任意の文字を利用した式の実行を許可しています．また，`a` 変数のみ `eval` サンドボックス内で使用することが可能です．  
ループ終了後，`guess` 変数に入力した値と変数 `a` が等しければ，`Flag` が出力されます．

`chal.py`
```python
import secrets

FLAG = "Alpaca{REDACTED}"
MAX_EVALS = 28

ALLOWED_CHARS = (
    "0123456789"
    "a"
    "+-*/()<>="
)

# 0~2^44
a = secrets.randbelow(2**44)

# 0~27 = 28回の少ない試行回数で，aを特定する方法が必要
for _ in range(MAX_EVALS):
    code = input("Eval > ")

    # 許可されていない文字判定
    if any(c not in ALLOWED_CHARS for c in code):
        print("Not allowed")
        continue

    try:
        # サンドボックス内でa変数のみ使用可能
        print(bool(eval(code, {"a": a, "__builtins__": {}})))   
    except Exception:
        print("Error")
guess = int(input("Guess > "))

if guess == a:
    print(f"Well done! Here's your flag: {FLAG}")
else:
    print("Wrong")
```

`eval` で任意コード実行をするのは難しそうです．  
そこで，どうにかして `a` を推測するため，不等式を利用した二分探索を思いつきましたが，`bool(eval(...))` は1回の試行で **True/False** の2通りの情報しかないため，これを繰り返しても $2 ^ {28} = 268435456 \ (9桁)$ となり，`a` の莫大な範囲では2分探索は不可能です．  

ここで，かなり悩んだため，以下の問題のヒントを見てみました．  
ヒントには，**Truthy/Falsy** に関するヒントが書かれていました．

> Python では、`True` はほとんど `1` のように、`False` はほとんど `0` のように振る舞います。この性質をうまく使いましょう。
{: .prompt-tip }

## 3分探索

もう一度，ソースコードを読んでみると `except Exception:` で `eval` が正常に実行できなかった際(`ZeroDivisionError`) 等で `Error` が表示されています．  
一度の試行で得られる情報量が **True/False/Error** となり底が $3$ になるため，$3 ^ {28} = 22876792454961 \ (14桁)$ の範囲をオラクル的に試行できます．  

$3 ^ {28} > 2 ^ {44}$ より，3分探索で28回の試行回数があれば，候補の中から必ず1つの数に確定させることができます!  

先ほどのヒントより，分母・分子の不等式の真偽値の出力は，**Truthy/Falsy** として算術演算では **1/0** で計算されます．  
これを利用して，$x1$ を範囲の $1/3$ の値に，$x2$ を $2/3$ に設定し，意図的にゼロ除算を発生させるため割り算を使用し，3通りの結果が出る **式(1)** を用います．

$$(a > x2)\ / \ (a \ge x1)\tag{1}$$

1. **`True` が返る場合 (全体の 2/3 より右側: $a > x2$)**:
	- **分子 ($a > x2$)**: `True` -> 1
	- **分母 ($a \ge x1$)**: `True` -> 1
	- **結果**: $1/1 = 1.0$ となり，`bool(1.0)` で `True` が出力されます．
2. **`False` が返る場合 (真ん中の 1/3 の領域: $x1 \le a \le x2$)**:
	- **分子 ($a > x2$)**: `False` -> 0
	- **分母 ($a \ge x1$)**: `True` -> 1 
	- **結果**: $0/1 = 0.0$ となり，`bool(0.0)` で `False` が出力されます．
3. **`Error` が返る場合 (全体の 1/3 より左側：$a < x1$)**:
	- **分子 ($a > x2$)**: `False` -> 0
	- **分母 ($a \ge x1$)**: `False` -> 0
	- **結果**: $0/0 = ZeroDivisionError$ となり，`try-except` によって `Error` が出力されます．

## 解法

数十回なので手動でもできますが...面倒くさいっ！  
なので，理論に基づいて `pwntools` で自動化ソルバを実装します．  
また，境界値の端数計算やコードのバグ取りには `Gemini` くんに助けてもらいました．

`solver.py`
```python
from pwn import *

MAX_EVALS = 28

p = process(['python3', './chal.py'])

# 初期探索上下限値
low = 0
high = 2**44 - 1

for i in range(MAX_EVALS):
	# 下限と上限の一致
    if low == high:
        p.sendlineafter(b"Eval > ", b"0")
        p.recvline()
        continue

    length = high - low + 1
    x1 = low + (length // 3)
    x2 = low + ((2 * length) // 3) - 1

    equation = f"(a>{x2})/(a>={x1})"

    res = p.sendlineafter(b"Eval > ", equation.encode())
    response_bytes = p.recvline()
    response_str = response_bytes.decode().strip()

    if response_str == "True":
        low = x2 + 1
    elif response_str == "False":
        low = x1
        high = x2
    elif response_str == "Error":
        high = x1 - 1
    else:
        print(f"Unexpected response: {response_str}")

p.sendlineafter(b"Guess > ", str(low).encode())
result = p.recvall().decode().strip()
print(f"\nFlag: {result}")
```

---
## Post-Mortem & Dead ends

競技プログラミングとCTFの合わせ技みたいな問題で，めっちゃ面白かった~!  
あと，普通に難しかった！ :)