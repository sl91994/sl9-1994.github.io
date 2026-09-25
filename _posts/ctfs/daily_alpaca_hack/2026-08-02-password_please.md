---
title: Daily-AlpacaHack 「Password, Please」Easy Writeup
description: bashの構文ミスによってワイルドカードが受け入れられる事を利用するMisc問題
date: 2026-08-03 00:05:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, misc]
math: true
---

# 20260802-daily_alpaca-misc-easy-password_please

## Summary

本問は，bashの構文ミスによってワイルドカードが受け入れられる事を利用するMisc問題です．

> - **Category**: Misc
> - **Description**: Bashで初めてのパスワードチェッカーを書きました。ログインしてください。
> - **Tools & TechStack**:
> 	- sh
> - **Release**: 2026/08/02
{: .prompt-info }

**階層構造**
```
.
├── chal.sh
├── compose.yaml
├── Dockerfile
└── flag.txt

1 directory, 4 files
```

---
## ソースコードの調査

`chal.sh`
```shell
secret="${RANDOM}${RANDOM}${RANDOM}${RANDOM}"

printf 'Password: '
read -r password

if [[ $secret == $password ]]; then
    cat /flag.txt
else
    echo 'Access denied'
fi
```

`${RANDOM}` は **15bit ($0 \leq RAMDOM \leq 32767$)** の範囲であるため，総当たり攻撃に対して脆弱です．  
しかし，4つ連結しているため総当たり総数も多く，入力後に接続が切れるため，乱数の予測や総当たりは困難なようです．

$$
32,768^4 = (2^{15})^4 = 2^{60} = 1,152,921,504,606,846,976 \text{ 通り}
$$

そのため，シェルスクリプトの構文をよく調べることにしました．

## 解法

Bash の `[[ ... ]]` 内において，`==` (`=`) の右辺の変数をダブルクォートで囲まない (`"$password"`) 場合，右辺の値はパターンマッチングとして解釈されます．  

そのため，文字列の一致を行うために `secret` を予測せずとも，`password` に `*` を入力するだけで `$secret` の値が何であっても，常に **true** になります．

```shell
$ ./chal.sh
Password: *
Alpaca{REDACTED}
```

---
## Post-Mortem & Dead ends

N/A

## References

N/A