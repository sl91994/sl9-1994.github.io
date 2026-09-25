---
title: (B-SIDE) AlpacaHack 「Long Flag Printer 2026」Medium Writeup
description: コンテナの設定と通信のキープ処理を利用するMisc問題
date: 2026-08-01 22:46:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, misc]
math: true
---

# 20260717-daily_alpaca-misc-medium-long_flag printer_2026

## Summary

本問は，コンテナの設定と通信のキープ処理を利用するMisc問題です．

> - **Category**: Misc
> - **Description**: おかげさまで [Flag Printer 2026](https://alpacahack.com/daily/challenges/flag-printer-2026) のフラグが長くなりました
> - **Tools & TechStack**:
> 	- Python
> 	- Socat
> 	- Pwntools
> - **Release**: 2026/07/19
{: .prompt-info }

**階層構造**
```
.
├── compose.yaml
├── Dockerfile
└── server.py

1 directory, 3 files
```

---
## ソースコードの調査

ソースコードを見ると，`0~30` までの総和でスリープしているため，トータルで `465` 秒 (**7.75分**) 待てば全ての **Flag** が表示されるはずです．

$$
0+1+2+\cdots+30=\frac{30\cdot 31}{2}=465
$$

しかし，途中で出力がされなくなります．

`server.py`
```python
import time

flag = "Alpaca{******** DUMMY ********}"
assert len(flag) == 31

for i, c in enumerate(flag):
    print(c, end="", flush=True)
    time.sleep(i)
```

ソースコードにはそのような処理が無かったため，`Dockerfile` を読むと `-T5`[^1] があるせいで，5秒以上双方向でデータのやり取りが行われないと，サーバからタイムアウトしてしまいます．

`Dockerfile`
```dockerfile
CMD ["socat", "-T5", "tcp-listen:1337,fork,reuseaddr", "exec:'python server.py'"]
```

じゃあ，「5秒になる前にデータを送り続ければいいのでは? 簡単じゃん!」 と思い，色々試しましたが軒並み途中で出力が切れました．

- ダミーデータを送った時刻を記録し2秒ごとにチャンクのチェックを行い，変化があれば再びダミーデータを送り接続を維持する方法
- とにかく，データを短い間隔で送り絶対に接続が切れないようにする方法

なんでこんな簡単な問題が **B-SIDE** なんだろうと思っていましたが，普通にデバッグと調整が難しいです...

## 解法

ローカル環境では，`TIMEOUT = 4.9` でうまくいきましたが，リモート環境だとタイムアウトしてしまうため `4.8` に下げています．  

なぜか，最後の `}` 記号が出力されませんが，とりあえず動きました．  
正直，通信状況?によって結果に揺らぎが生じます．

`solver.py`
```python
from pwn import *
import re

PATTERN = re.compile(rb"Alpaca\{.*\}")
TIMEOUT = 4.8

p = remote("localhost", 1337)
part = b""

while True:
    try:
        chunk = p.recv(1, timeout=TIMEOUT)
    except EOFError:
        break

    if chunk:
        # データを受信できたとき
        part += chunk
        print(chunk.decode(), end="", flush=True)

        # 最後まで取得できたとき
        if re.search(PATTERN, part):
            print("\n")
            break
    else:
        # 応答がなければダミーデータを送信し，socatのタイマーリセット
        p.send(b"\0")

print(f"Flag: {part.decode()}")
```


---
## Post-Mortem & Dead ends

初めて，**B-SIDE** 解けた~!
面白かったけど，タイムアウトしまくるから調整がムズカッタ...

## References

[^1]: [socat(1) - Linux manual page](https://man7.org/linux/man-pages/man1/socat.1.html)
