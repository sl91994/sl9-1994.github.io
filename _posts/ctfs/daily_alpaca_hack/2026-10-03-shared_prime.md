---
title: Daily-AlpacaHack 「Shared Prime」Medium Writeup
description: RSA暗号における共有素数の性質を利用する問題
date: 2026-10-05 00:10:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, crypto, rsa/shared_factor_attack]
---

# daily_alpaca-crypto-medium-shared_prime

## Summary

本問は，RSA暗号における共有素数の性質を利用する問題です．

> - **Category**: Crypto
> - **Description**: simple!
> - **Tools & TechStack**:
> 	- Python
> - **Release**: 2026/10/03
{: .prompt-info }

**配布ファイル**
```
.
├── chall.py
└── output.txt

1 directory, 2 files
```

---
## 解法

問題では，`n1, n2, c1, c2` が与えられています．
また，`n1, n2` が同じ `p` を使いまわしています．

```python
import os

from Crypto.Util.number import bytes_to_long, getPrime


FLAG = os.environ.get("FLAG", "Alpaca{DUMMY}").encode()
e = 65537

p = getPrime(1024)
q1 = getPrime(1024)
q2 = getPrime(1024)

n1 = p * q1
n2 = p * q2
m = bytes_to_long(FLAG)
assert m < min(n1, n2)

c1 = pow(m, e, n1)
c2 = pow(m, e, n2)

print(f"{n1 = }")
print(f"{n2 = }")
print(f"{c1 = }")
print(f"{c2 = }")
```

`n1, n2` は単体で考えると，分解することが困難な2048bitのRSAモジュラスです．  
しかし，`n1, n2` が同じ `p` を素因数に持っていることで，`n1, n2` の最大公約数 `p` を計算することができます．  
ここで重要な点として，`p, q1, q2` が素数であり，かつ $q1 \ne q2$ であるため，**両方に共通する約数は `1` と `p` だけ** になります．

## Solver を書く

`n1, n2` を単体で素因数分解するのは困難ですが，最大公約数はユークリッドの互除法で高速に計算できるため，`gcd(n1, n2)` によって `p` を直接求めることができます．  
`p` が手に入れば，あとはそのまま復号できます．

```python
from math import gcd
from Crypto.Util.number import long_to_bytes

e = 65537
n1 = #...
n2 = #...
c1 = #...
c2 = #...

p = gcd(n1, n2)
assert 1 < p < n1

q1 = n1 // p
phi = (p - 1) * (q1 - 1)
d = pow(e, -1, phi)
m = pow(c1, d, n1)

print(long_to_bytes(m))
```

```shell
$ python3 solver.py
b'<REDACTED>'
```

---
## Post-Mortem & Dead ends

N/A

## References

N/A