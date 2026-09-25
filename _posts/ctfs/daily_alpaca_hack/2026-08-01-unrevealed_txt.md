---
title: Daily-AlpacaHack 「Unrevealed TXT」Easy
description: 権威DNSサーバのゾーン転送設定を利用して，ホスト名が不明なTXTレコードを読み取るMisc問題
date: 2026-08-01 15:11:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, misc]
---

# 20260427-daily_alpaca-misc-easy-unrevealed_txt

## Summary

本問は，権威DNSサーバのゾーン転送設定を利用して，ホスト名が不明なTXTレコードを読み取るMisc問題です．

> [!info] Challenge Info
> - **Category**: Misc
> - **Description**: TXTレコードにフラグを入れても，名前を無茶苦茶な値にすれば，バレないよね...？
> - **Tools & TechStack**:
> 	- Python
> 	- dig
> - **Release**: 2026/04/27
{: .prompt-info }

**階層構造**
```
.
├── app
│   ├── Dockerfile
│   └── server.py
├── compose.yaml
└── dns
    ├── db.alpaca.internal
    ├── Dockerfile
    └── named.conf

3 directories, 6 files
```

---
## ソースコードの調査

`app/server.py` は，DNS問い合わせ用のラッパーになっており，ユーザ入力を `dig` コマンドに渡して，`@dns` に問い合わせる実装になっています．

`app/server.py`
```python
import subprocess
import shlex

print("Example: paca.alpaca.internal TXT")
subprocess.run(["dig", "@dns"] + shlex.split(input("$ dig ")))
```

## 権威DNSサーバの構成調査

`dns/Dockerfile`
```dockerfile
#...
RUN sed -i "s/REPLACE_ME/$(tr -dc 'A-Za-z0-9' </dev/urandom | head -c 32)/g" /etc/bind/zones/db.alpaca.internal
#...
```

`dns/db.alpaca.internal`
```
$TTL 3600
@               IN   SOA        ns.alpaca.internal. admin.alpaca.internal. (
                                    2026041701   ; serial
                                    3600         ; refresh
                                    1800         ; retry
                                    604800       ; expire
                                    300          ; minimum
                                    )
                IN   NS         ns.alpaca.internal.

ns              IN   A          127.0.0.1

paca            IN   TXT        "pacapaca"
llama           IN   TXT        "alpaca"

REPLACE_ME      IN   TXT        "Alpaca{REDACTED}"
```

`dns/db.alpaca.internal` では，**Flag文字列** を持っている `TXT` レコードのホスト名は `REPLACE_ME` となっており，`dns/Dockerfile` によってコンテナ起動時に，**32文字の英数字** に置き換えられています．  

そのため，直接 `<ホスト名>.alpaca.internal` のような形でアクセスして，レコードの中身を見ることはできません．  

なので，ホスト名が分からない状態で，全ての `TXT` レコードの値を読み出す方法を調べます．

`dns/named.conf`
```conf
options {
    directory "/var/cache/bind";

    recursion no;
    dnssec-validation no;
};

zone "alpaca.internal" IN {
    type master;
    file "/etc/bind/zones/db.alpaca.internal";
    allow-transfer { any; };
};
```

`zone "alpaca.internal"` において，`allow-transfer { any; };` が有効化されているため，**AXFR (ゾーン転送)** が許可されています．

そのため，`alpaca.internal AXFR +noall +answer` と入力することで，フルゾーン転送で全てのレコードを取得し，`Answer` だけ表示させることで，**Flag** を入手できました．

1. `AXFR` 
    フルゾーン転送を要求．成功すると，そのゾーン内のレコード一覧がまとまって返る．
2. `+noall +answer`
    余計な表示を消して，`Answer` セクションだけを表示する．

```shell
$ nc localhost 1337

Example: paca.alpaca.internal TXT
$ dig alpaca.internal AXFR +noall +answer
alpaca.internal.        3600    IN      SOA     ns.alpaca.internal. admin.alpaca.internal. 2026041701 3600 1800 604800 300
alpaca.internal.        3600    IN      NS      ns.alpaca.internal.
3Y41j2hJfq592y5oKmeodObPYsLMVR51.alpaca.internal. 3600 IN TXT "Alpaca{REDACTED}"
llama.alpaca.internal.  3600    IN      TXT     "alpaca"
ns.alpaca.internal.     3600    IN      A       127.0.0.1
paca.alpaca.internal.   3600    IN      TXT     "pacapaca"
alpaca.internal.        3600    IN      SOA     ns.alpaca.internal. admin.alpaca.internal. 2026041701 3600 1800 604800 300
```

---
## Post-Mortem & Dead ends

N/A

## References

- **Host**:
  - ネットワークに接続されているコンピュータやサーバーなどの機器そのものを指し，DNSでは，特定の機器やサービスを識別するためにつけられた名前を **ホスト名** と呼ぶ．また，ドメインの接頭辞として，特定のサーバを指定する役割を持つ．
- **DNS Record Types**:
  - DNSサーバーが保持する **名前と値の対応表(辞書)** の各エントリを指す．用途に応じて異なるレコードタイプが存在する．
  - **A**: ホスト名に対応するIPv4アドレスを定義する
  - **AAAA**: ホスト名に対応するIPv6アドレスを定義する
  - **CNAME**: ホスト名の別名(エイリアス)を定義する
  - **MX**: ドメイン宛てのメール送信先(メールサーバー)を指定する
  - **TXT**: 任意のテキスト情報を定義する．SPF/DKIMなどの送信ドメイン認証や所有権確認に広く利用される
  - **NS**: 該当ドメインの権威DNSサーバーのホスト名を指定する
  - **SOA**: ゾーンの管理責任者，シリアル番号，更新間隔などの管理情報を定義する
- **Zone**:
  - 特定のDNSサーバーが管理権限を持つドメインの管理範囲およびその設定データの集合を指す．ドメインツリー構造において，親ゾーンから委譲された範囲ごとに独立したゾーンファイルとして管理される．
- **Zone Transfer (AXFR)**:
  - プライマリDNSサーバーからセカンダリDNSサーバーへ，ゾーンの全レコード情報を一括して複製・同期する仕組み(フルゾーン転送)．