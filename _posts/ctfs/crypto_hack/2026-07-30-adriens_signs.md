---
title: CryptoHack 「Adrien's Signs (Modular Arithmetic)」Writeup
description: モジュラ演算におけるルシャンドル記号の性質を利用して，ビット化されたFlagを逆算するCrypto問題
date: 2026-07-30 23:37:00 +0900
categories: [CyberSecurity, CTF]
tags: [cryptohack, crypto]
math: true
---

# 20260730-crypto_hack-adriens_signs

## Summary

本問は，モジュラ演算におけるルシャンドル記号の性質を利用して，ビット化されたFlagを逆算するCrypto問題です．

> - **Category**: Crypto
> - **Description**: アドリアンは、記号やマイナス記号を使ってメッセージを暗号化する方法を探っています。フラグを復元する方法を見つけられますか？
> - **Tools & TechStack**:
> 	- Python
> - **Release**: N/A
{: .prompt-info }

**階層構造**
```
.
├── output_80fc6398d2fd9f272186d0af510323f9.txt
└── source_734d7e14251f950935f83d228f8694ab.py

1 directory, 2 files
```

---
## ソースコードの調査

- フラグ文字列を `10110011...` のような0と1のビット列に変換する．
- 1ビットずつループを回し，毎回**ランダムな数 $e$** を生成する．
- そのビットが `1` なら，$a^e \bmod p$ を暗号文としてリストに追加する．
- そのビットが `0` なら，$-(a^e) \bmod p$ を暗号文としてリストに追加する．

ここで，$e$ は $0 \leq e \leq p$ の範囲でランダムに変化します．暗号文の数字から，$e$ を逆算することは，巨大な素数 $p$ では不可能です．  
そのため，乱数 $e$ が変化しても，変わらない性質を見つけることが必要です．

`source_734d7e14251f950935f83d228f8694ab.py`
```python
from random import randint

a = 288260533169915
p = 1007621497415251 # 素数であり，(p % 4) == 3 を満たす

FLAG = b'crypto{????????????????????}'

def encrypt_flag(flag):
    ciphertext = []
    # 8bitの2進数に変換
    plaintext = ''.join([bin(i)[2:].zfill(8) for i in flag])

    # 文字毎に演算
    for b in plaintext:
        # e は 1 以上 p 以下の整数
        e = randint(1, p)
        n = pow(a, e, p)
        if b == '1':
            ciphertext.append(n)
        else:
            n = -n % p
            ciphertext.append(n)
    return ciphertext

print(encrypt_flag(FLAG))
```

## 解法

そこで，以下のルシャンドル記号の性質を用いることを考えました．

- $2次剰余 \times 2次剰余 = 2次剰余$
- $2次剰余 \times 平方非剰余 = 平方非剰余$
- $平方非剰余 \times 平方非剰余 = 2次剰余$

$$
(a/p) \equiv a^{(p-1)/2} \pmod p\tag{1}
$$

固定値 $a$ と $p$ を **式(1)** のルシャンドル記号の定義式に代入して計算してみると **二次剰余** になるような数字が選ばれています．  
$a$ が2次剰余であるのならば，それを何回か掛けた $a^{e}$ もルシャンドル記号の性質から必ず2次剰余になります．

$$
(a^e / p) = (a / p)^e = 1^e = 1\tag{2}
$$

そのため，**式(2)** より $e$ がどんな値においても，$a^{e}$ は常にルシャンドル記号が `1` になります．

### 元のビットが `0` だった場合

次に，元のビットが `0` だった場合の処理 `n = -n % p` について考えます．  
モジュラ演算の世界においても，$-n$ は $-1 \times n$ と同じです．

よって，暗号文は以下の2通りになります．

- **ビットが `1` の時**: $a^{e}$
- **ビットが `0` の時**: $-1 \cdot a^{e}$

これらをルシャンドル記号で計算すると，以下になります．

- **ビット `1` の判定**: $(a^e / p) = 1$
- **ビット `0` の判定**: $(-1 \cdot a^e / p) = (-1 / p) \times (a^e / p)$

ここで，$(a^e / p) = 1$ であることは先ほど証明しました．  
したがって，ビット `0` の判定結果は **$-1$ が2次剰余か平方非剰余か (つまり $(-1 / p)$ が $1$ か $-1$ か)** という1点のみに依存することになります．

```python
>>> True if (1007621497415251 % 4) == 3 else False
True
```

### $p$ の条件

$p = 1007621497415251$ を **factordb**[^1] で素因数分解してみると，**$p \equiv 3 \pmod 4$ を満たす素数** であることが分かりました．  

暗号を解読するためには，ビット `0` の判定結果が，ビット `1` の判定結果と明確に区別できなければなりません．  
つまり，**$(-1 / p)$ が絶対に $-1$ (平方非剰余) になってほしい** ので，そのような $p$ が存在しなければいけません．  

**式(1)** を用いて $(-1 / p)$ を計算してみると，以下の **式(3)** のようになります．

$$
(-1 / p) \equiv (-1)^{(p-1)/2} \pmod p\tag{3}
$$

$-1$ を何乗かする計算なため，結果が $-1$ になるためには，指数である $(p-1)/2$ が **奇数** でなければなりません．  
ここで，$p \equiv 3 \pmod 4$ が満たされていれば，この条件は，$p$ は 4の倍数に3を足した数であると言いかえれるため，$p = 4k + 3 \ \text{(kは整数)}$ と置くことができます．  
これを，指数部に代入すると，**式(4)** と変形でき，$2k + 1$ は奇数であるため，$(-1)^{\text{奇数}} = -1$ となり，$(-1 / p) = -1$ が保証されます．

$$
\frac{p-1}{2} = \frac{(4k + 3) - 1}{2} = \frac{4k + 2}{2} = 2k + 1\tag{4}
$$

そのため，前半と後半の部分で，それぞれルシャンドル記号が計算され，$-1 \times 1 = -1$ となります．

```python
# -1
>>> pow(-1, (1007621497415251 - 1) // 2, 1007621497415251)
1007621497415250

# 1
>>> pow(1, (1007621497415251 - 1) // 2, 1007621497415251)
1
```

### ソルバを書く

`solver.py`
```python
ENCRYPTED_FLAG = [] # 省略
A = 288260533169915
P = 1007621497415251

def decrypt_flag(encrypted):
    flag = ""

    for enc in encrypted:
        # ルシャンドル記号を計算
        legendre = pow(enc, (P - 1) // 2, P)
        print(f"Legendre symbol: {legendre}")
        if legendre == 1:
            flag += str(1)
        else:
            # モジュロ演算の世界における -1 = (P - 1)
            flag += str(0)

    return "".join([chr(int(flag[i : i + 8], 2)) for i in range(0, len(flag), 8)])

print(decrypt_flag(ENCRYPTED_FLAG))
```

```shell
$ python3 solver.py
Legendre symbol: 1
Legendre symbol: 1
Legendre symbol: 1007621497415250 # P - 1
Legendre symbol: 1
crypto{REDACTED}
```

---
## Post-Mortem & Dead ends

CryptoHackで学んだ知識を使えて，めっちゃ面白かった!  
けど，発想が普通に難しかった...

## References

[^1]: [factordb.com](https://factordb.com/index.php?query=1007621497415251)
