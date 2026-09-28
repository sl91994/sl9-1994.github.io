---
title: Pwnable.kr 「collision (Toddler's Bottle)」Writeup
description: 線形・非暗号学的ハッシュの衝突攻撃
date: 2026-09-28 12:06:00 +0900
categories: [CyberSecurity, CTF]
tags: [pwnable_kr, pwn]
image:
  path: /assets/img/ctf/pwnable_kr/collision.png
  alt: pwnable_kr_collision_icon
---

# pwnable_kr-collision

## Summary

本門は，線形・非暗号学的ハッシュの衝突攻撃の問題です．

> - **Category**: Pwn
> - **Description**: Daddy told me about cool MD5 hash collision today. I wanna do something like that too!
> - **Tools & TechStack**:
>   - C
> - **Release**: `N/A`
{: .prompt-info }

**階層構造**

```
.
├── col
├── col.c
├── Dockerfile
├── flag
├── loader.sh
├── readme
├── run.sh
└── super.pl

1 directory, 8 files
```

---

## ソースコードの解析

`check_password()` は，**20byte の入力を，4byte ずつ5つの int として解釈し，その総和を返す** ように実装されています．  
そのため，**4byte \* 5 = 20byte** として，総和が `0x21DD09EC` と一致する入力を見つければ良いとわかりました．  
解は無数にあるため，適当に選びます．  
今回は，`0x21DD09EC = 0x06C5CEC8 * 4 + 0x06C5CECC` としました．

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
	int* ip = (int*)p;
	int i;
	int res=0;
	for(i=0; i<5; i++){
		res += ip[i];
	}
	return res;
}

int main(int argc, char* argv[]){
	if(argc<2){
		printf("usage : %s [passcode]\n", argv[0]);
		return 0;
	}
	if(strlen(argv[1]) != 20){
		printf("passcode length should be 20 bytes\n");
		return 0;
	}

	if(hashcode == check_password( argv[1] )){
		setregid(getegid(), getegid());
		system("/bin/cat flag");
		return 0;
	}
	else
		printf("wrong passcode.\n");
	return 0;
}
```

今回は，接続先にpython3が存在したため，そのままリトルエンディアンでバイトを組み立て流し込みました．

```shell
$ NIXPKGS_ALLOW_UNFREE=1 nix-shell -p steam-run --run "steam-run ./col \$(python3 -c 'import sys; sys.stdout.buffer.write(b\"\\xc8\\xce\\xc5\\x06\"*4 + b\"\\xcc\\xce\\xc5\\x06\")')"
flag{this_is_a_dummy_flag}

$ socat STDIO,raw,echo=0,opost,onlcr TCP:pwnable.kr:10002

=====================================================================================
  [ Collision Shell ]
  Daddy told me about cool MD5 hash collision today
  I wanna do something like that too!

  ※ You got 500s in /bin/sh. Good luck.
  ※ Available: python2 python3
  ※ Interactive terminal available: vim-tiny nano
=====================================================================================
$ ./col $(python3 -c 'import sys; sys.stdout.buffer.write(b"\xc8\xce\xc5\x06"*4 + b"\xcc\xce\xc5\x06")')
<REDACTED>
```

---

## Post-Mortem & Dead ends

N/A

## References

N/A
