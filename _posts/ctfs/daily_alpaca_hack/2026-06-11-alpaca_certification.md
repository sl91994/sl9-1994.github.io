---
title: Daily-AlpacaHack 「Alpaca Certification」Easy Writeup
description: SSL証明書のCustom OIDに対するハードコードされたFlag
date: 2026-06-12 07:23:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, web, hardcoded_credentials]
---

# 20260611-daily_alpaca-web-easy-alpaca_certification

## Summary

本問は，SSL証明書のOIDフィールドに，直接機密情報が記載されていることを利用する問題です．

> - **Category**: Web
> - **Description**: Daily Alpacahackを半年以上プレイしたことをここに賞します。
> - **Tools & TechStack**:
> 	- openssl
> 	- custom_oid
> - **Release**: 2026/06/11
{: .prompt-info }

**階層構造**
```
.
├── compose.yaml
└── nginx
    ├── default.conf
    ├── Dockerfile
    ├── index.html
    └── openssl.cnf

2 directories, 5 files
```

---
## ソースコードの調査

素直に，`index.html` に記載されているヒントに従い，`openssl.conf` を見ます．

`nginx/index.html`
```html
<!doctype html>
<!-- This file is fully slopped by LLM, and totally unrelated this challenge -->
<!-- Hint: The flag is in openssl.cnf -->
 
<!-- 以下のファイルは、全てLLMを利用して生成され、全く見る必要のないファイルとなっています。 -->
<!-- ヒント: フラグはopenssl.cnfにあります。 -->
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Certificate of Achievement - Daily AlpacaHack</title>
<!-- ... -->
```

## OpenSSLの設定を調査

各文の意味を調べると，`x509_extensions = v3_req` で，**X.509 v3 拡張が有効** であることが分かりました．
また，`[ v3_req ]` における設定で，**Custom OID**[^1] が設定されており，UTF8文字列として，**Flag** が埋め込まれています．

`nginx/openssl.conf`
```ini
[ req ]
distinguished_name = dn # DN定義
x509_extensions = v3_req # X.509 v3 拡張を有効化
prompt = no # 対話機能を無効化

[ dn ]
O = Alpacahack
OU = Alpacahack

[ v3_req ]
1.2.3.4.5.6 = ASN1:UTF8String:Alpaca{REDACTED} # カスタムOIDの拡張属性
```

### 証明書確認 (ブラウザ)

ブラウザのセキュリティタブから，ssl関連の情報を確認することができます．そこで，`View certificate>詳細>証明書のフィールド>拡張機能>OID.1.2.3.4.5.6` に進むと，16進数での値が記載されていたため，デコードするとフラグが入手できました．

```
0C 2F 41 6C 70 61 63 61 7B 4C 33 74 73 5F 63 6F 6E 74 69 6E 75 33 5F 74 68 31 73 5F 66 30 72 5F 61 6E 6F 74 68 65 72 5F 36 5F 6D 6F 6E 37 68 73 7D
```

### 証明書確認 (CLI)

ブラウザではなく，`openssl` からも確認可能です．こちらはオプションで文字列として変換した結果を表示することができます．

```shell
$ openssl s_client -connect 34.170.146.252:21751  -showcerts </dev/null | openssl x509 -text -noout

Connecting to 34.170.146.252
Can't use SSL_get_servername
depth=0 O=Alpacahack, OU=Alpacahack
verify error:num=18:self-signed certificate
verify return:1
depth=0 O=Alpacahack, OU=Alpacahack
verify return:1
DONE
Certificate:
    Data:
        Version: 3 (0x2)
#...
        X509v3 extensions:
            1.2.3.4.5.6: 
                ./Alpaca{REDACTED}
#...
```

---
## Post-Mortem & Dead ends

- **OIDを確認する**: ブラウザのセキュリティタブから証明書を見る．

## References

[^1]: [Inserting Custom OIDs into OpenSSL](https://knowledge.digicert.com/quovadis/ssl-certificates/csr-generation/inserting-custom-oids-into-openssl)
