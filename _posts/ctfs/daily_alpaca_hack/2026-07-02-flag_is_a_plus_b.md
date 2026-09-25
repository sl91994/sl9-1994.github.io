---
title: Daily-AlpacaHack 「Flag is A+B」Easy
description: 論理演算と算術演算の関係式を用いた復号
date: 2026-07-03 20:26:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, crypto]
math: true
---

# 20260702-daily_alpaca-crypto-easy-flag_is_a_plus_b

## Summary

本問は，論理演算と算術演算の関係式を用いて復号する問題です．

> - **Category**: Crypto
> - **Description**: A+B を計算すればフラグがわかるよ! あれ、A と B っていくつだっけ…?
> - **Tools & TechStack**:
> 	- Python
> 	- XOR
> 	- OR
> - **Release**: 2026/07/02 
{: .prompt-info }

**階層構造**
```
.
├── chall.py
└── output.txt

1 directory, 3 files
```

---
## ソースコードの調査

暗号化のアルゴリズムを読み解くと，以下が分かりました．

- `FLAG` を数値化したもの未満のランダムな整数 $A$ を生成
- $B = \text{FLAG} - A$ とし，結果として $A + B = \text{FLAG}$ となるように設定
- $A$ と $B$ の論理和と排他的論理和を出力

$A$ と $B$ の排他的論理和・論理和が与えられています．$A + B$ を求める方法を考えます．
そこで，半加算器で使われる性質を用います．

`chall.py`
```python
from Crypto.Util.number import bytes_to_long
import os
import random

FLAG = bytes_to_long(os.getenv("FLAG", "Alpaca{DUMMY}").encode())

A = random.randint(0, FLAG)
B = FLAG - A

assert A + B == FLAG

print(f"A or B = {A | B}")
print(f"A xor B = {A ^ B}")
```

### 数学的な導出

ビットごとの加算において，$A$ と $B$ の足し算は以下の式で表せます．
ここで，係数の2は繰り上がりのために左シフトをするためのものです．

$$A + B = (A \text{ xor } B) + 2(A \text{ and } B)$$

ここで，左辺の $A+B$ を求めるために未知の $A \text{ and } B$ が必要になります．そこで，以下の関係式を用いることで $A \text{ and } B$ を導き出すことができます．

$$(A \text{ or } B) - (A \text{ xor } B) = A \text{ and } B$$

したがって，これらを組み合わせると次のような式になります．

$$A + B = (A \text{ xor } B) + 2((A \text{ or } B) - (A \text{ xor } B))$$

右辺の括弧を展開し，同類項をまとめると以下のようになります．

$$A + B = 2(A \text{ or } B) - (A \text{ xor } B)$$

## 解法

`solver.py`
```python
from Crypto.Util.number import long_to_bytes

aorb = 1708520672692343497693425015709016883325158039728511260268583494549501
axorb = 1653224853895272618878301831150773792186632776885840473347068872416893

encoded_flag = 2 * aorb - axorb

flag = long_to_bytes(encoded_flag).decode()
print(flag)
```

---
## Post-Mortem & Dead ends

## References
