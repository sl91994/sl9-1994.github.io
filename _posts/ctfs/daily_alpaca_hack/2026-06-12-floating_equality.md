---
title: Daily-AlpacaHack 「Floating Equality」Medium
description: 2種類のIEEE754 浮動小数点数計算での桁丸め誤差
date: 2026-06-13 10:30:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, misc, vuln/floating_point]
---

# 20260612-htb-chall-misc-madium-floating_equality

## Summary

本問は，2種類の浮動小数点数への計算・変換方法において，誤差が発生する入力を調べ，EPS以上の誤差を発生させる問題です．

> - **Category**: Misc
> - **Description**: 浮動小数点型の比較には == の代わりに EPS を用いなさいという人もいますが…
> - **Tools & TechStack**:
> 	- Python
> - **Release**: 2026/06/12
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

1. `x_str` として，16文字以下かつ，小数点以下が10桁未満の小数を受け入れる．
2. 入力を別々の方法で処理 (`0.0001` としたとき)
	1. `0.001e9` という科学記数法の文字列が生成され，`float()` の内部処理で十進文字列全体を一度に解釈し，最後に1回だけ IEEE754での値へ丸められる．
	2. `x_1e9`: `0.001` を `float()` で変換する．その後，その値を10億倍する．
3. もし， `xe9` と `x_1e9` の誤差が，`EPS`[^1] 以上であれば `Flag` が表示される．

そのため，**2.1と2.2** の2種類で差を取ったときに，誤差が発生するような入力を見つけることが目標になります．

`chal.py`
```python
import os
import re

EPS = 1e-10
FLAG = os.environ.get("FLAG", "Alpaca{dummy}")


def validated_input(prompt: str) -> str:
    res = input(prompt)
    # 入力文字が16以下であること
    assert len(res) <= 16
    # 小数点以下は9桁以下，整数部と小数部は必須
    assert re.fullmatch(r"(?:0|[1-9][0-9]*)\.[0-9]{1,9}", res)
    return res


x_str = validated_input("value> ")
x = float(x_str)

# 丸め1回
xe9 = float(f"{x_str}e9")
# 丸め2回
x_1e9 = x * 1e9 # x * 1000000000.0
err = abs(xe9 - x_1e9)

print(f"{xe9 = :.100g}")
print(f"{x_1e9 = :.100g}")
print(f"{err = :.100g}")

try:
    assert err < EPS
except AssertionError:
    print(f"FLAG: {FLAG}")
```

**pwntools** で全探索しようかと思いましたが，連続して入力できる機構になっていませんし，そもそもローカルで検証できるのでする意味ありませんでした．:(
ので，ローカルで `abs(xe9 - x_1e9)` で差が生まれるような値を総当たりします．

## 誤差が発生する値を探索

Python標準での `float` (**IEEE 754**) は，倍精度浮動小数点数 (**64bit**) です．
基本的には正確な計算が行えるのですが，一部の十進小数は IEEE754 の2進浮動小数点数で正確に表現できず，最も近い値へ丸められます．

| 変数      | 計算方式                                                                             | 変換回数 |
| ------- | -------------------------------------------------------------------------------- | ---- |
| `xe9`   | 科学記数法を含む十進文字列全体を解釈し，その結果を IEEE754 に変換する．                                         | 1回   |
| `x_1e9` | 文字列の小数を2進数に変換した後，その値を10億倍 (誤差ごと) する．<br>掛け算を行った結果を再び2進数で表現し直す際に，丸め誤差が発生する可能性がある． | 2回   |

本問の本質は、**十進数表現のまま結合してからパースした結果 (`float(f"{a}e9")`)** と，**パースしてから乗算した結果 (`float(a) * 1e9`)** が必ずしも一致しないことです．
`xe9` は後者に対して丸めが1回，`x_1e9` は丸めが2回発生するため，両者の結果が異なる入力が存在します．

`solver.py`
```python
def find_differences():
    count = 0
    # 0.001 ~ 10.000 までの小数で検証
    for i in range(1, 10000):
        x_str = f"0.{i:03d}".rstrip("0")
        
        xe9 = float(f"{x_str}e9")
        x_1e9 = float(x_str) * 1e9
        
        if xe9 != x_1e9:
            err = abs(xe9 - x_1e9)
            print(f"x_str = '{x_str}'")
            print(f"| {xe9 = :.100g}")
            print(f"| {x_1e9 = :.100g}")
            print(f"| {err = :.100g}")
            count += 1
            
            if count == 1:
                break

find_differences()
```

```shell
$ python3 solver.py 
x_str = '0.067'
| xe9 = 67000000
| x_1e9 = 67000000.000000007450580596923828125
| err = 7.450580596923828125e-09

$ nc localhost 1337
value> 0.067
xe9 = 67000000
x_1e9 = 67000000.000000007450580596923828125
err = 7.450580596923828125e-09
FLAG: Alpaca{REDACTED}
```

---
## Post-Mortem & Dead ends


## References

[^1]: [計算機イプシロン - Wikipedia](https://ja.wikipedia.org/wiki/%E8%A8%88%E7%AE%97%E6%A9%9F%E3%82%A4%E3%83%97%E3%82%B7%E3%83%AD%E3%83%B3)
