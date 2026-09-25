---
title: Pwnable.kr 「fd (Toddler's Bottle)」Writeup
description: file descriptor の理解を試すpwn問題
date: 2026-07-27 22:34:00 +0900
categories: [CyberSecurity, CTF]
tags: [pwnable_kr, toddlers_bottle, pwn]
---

# 20260727-pwnable_kr-toddlers_bottle-fd

## Summary

本問は，**file descriptor** の理解を試すpwn問題です．

> - **Category**: Pwn
> - **Description**: Mommy! what is a file descriptor in Linux?
> - **Tools & TechStack**:
> 	- C
> - **Release**: N/A
{: .prompt-info }

**階層構造**
```
.
├── fd
├── fd.c
├── flag
└── readme

0 directories, 4 files
```

---
## ソースコードの調査

メインの操作を抜き出すと，大まかに以下の処理で構成されています．

- `buf[32]` で 32byte の文字列型の配列を用意する．
- `atoi(argv[1])` でコマンドライン引数で入力された **Ascii文字列を整数に変換** し，そこから `0x1234 = 4660` を減算したものを，**fd (file descriptor)** として設定する．
- `read(fd, buf, 32)` で，`fd` に応じた入力で，`buf[]` に書き込まれる．
- `buf[]` 内の文字列が，`"LETMEWIN\n"` と一致すれば，`cat flag` される．

`fd.c`
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
char buf[32];
int main(int argc, char* argv[], char* envp[]){
        if(argc<2){
                printf("pass argv[1] a number\n");
                return 0;
        }
        // ascii を int に変換し．0x1234 を引く
        int fd = atoi( argv[1] ) - 0x1234;
        int len = 0;
        len = read(fd, buf, 32);
        if(!strcmp("LETMEWIN\n", buf)){
                printf("good job :)\n");
                setregid(getegid(), getegid());
                system("/bin/cat flag");
                exit(0);
        }
        printf("learn about Linux file IO\n");
        return 0;
}
```

## 解法

`atoi( argv[1] ) - 0x1234;` より，10進数で `4660` を入力することで，**fd** を `0 (標準入力)` に設定することができます．  
その後，標準入力の待ち受けで `"LETMEWIN"` と入力し，Enter キーを押すと改行文字 `\n` が入力データに付加されます．  
よって，その後の `!strcmp("LETMEWIN\n", "LETMEWIN\n")` が `1` となり，`flag` が表示されます．

```shell
fd@ubuntu:~$ ./fd 4660
LETMEWIN
good job :)
Mama! Now_I_understand_what_file_descriptors_are!
```

---
## Post-Mortem & Dead ends

毎回思うんだけど，`!strcmp(A, B)` って書き方分かりづらくない?  
普通に読んだら，否定演算子 `!` があるせいで，非一致を判定してるように見えるから，個人的には `strcmp(A, B) == 0` の書き方の方が好み．

## References

N/A