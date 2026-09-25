---
title: Daily-AlpacaHack 「Flag for localhost」Easy Writeup
description: リバースプロキシとサーバフレームワークの設定ミスによるIP認証バイパス
date: 2026-06-07 09:42:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, web, vuln/web/ip_address_spoofing, vuln/web/http_header_injecton]
---

# 20260606-daily_alpaca-web-easy-Flag_for_localhost

## Summary

本問は，リバースプロキシの挙動とサーバフレームワークの挙動の組み合わせにより発生するバイパス手法を利用する問題です．

> - **Category**: Web
> - **Description**: localhost からアクセスすればフラグが見えるよ！
> - **Tools & TechStack**:
> 	- Javascript
> 	- Nginx
> - **Release**: 2026/06/06
{: .prompt-info }

**階層構造**
```
.
├── compose.yaml
├── nginx
│   ├── default.conf
│   └── Dockerfile
└── web
    ├── Dockerfile
    ├── index.js
    ├── package.json
    └── package-lock.json

3 directories, 7 files
```

---
## ソースコードの調査

Nginxをリバースプロキシとして動作させ，受け取ったリクエストをWebサーバー (`http://web:3000`) へ転送するための標準的な設定がされています．
**localhost からアクセスするとフラグが見れる** と問題文に記載があり，`index.js` でも，`allowedIps[]` で `127.0.0.1` からのアクセスのみにフィルタリングされています．

**例: `127.0.0.1` 以外のアクセス**
```shell
curl http://34.170.146.252:42474
Access from xxx.xxx.xx.xxx is not allowed
```

`nginx/default.conf`
```nginx
server {
    listen 80;
    location / {
        proxy_pass http://web:3000;

        proxy_set_header Host              $host;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### リバースプロキシとExpressの設定ミス

重要な点として，**express** の `app.set("trust proxy", true);`[^1] が有効化されています．
`trust proxy` が有効化されている場合，アプリケーションがプロキシ (**Nginx**) を信頼するため，`req.ip` で取得されるアドレスの挙動は以下のようになります．

- **取得元**: HTTPヘッダーの `X-Forwarded-For` の値を参照します (ヘッダーがない場合はTCP接続元IPにフォールバック)．
- **挙動**: `X-Forwarded-For` にカンマ区切りで複数のIPが格納されている場合，Expressはそれを配列としてパースし，**インデックス値 `0` に書かれているIPアドレス (最左端)** を `req.ip` に格納します．

今回の，リバースプロキシの設定では，`$proxy_add_x_forwarded_for`[^2] が設定されています．
これが設定されている場合，クライアントのリクエストヘッダ `X-Forwarded-For` に，`$remote_addr` 変数を **カンマ区切りで格納する** と記載されていました．

`web/index.js`
```javascript
import express from "express";

const FLAG = process.env.FLAG ?? "Alpaca{REDACTED}";

const app = express();
app.set("trust proxy", true);

const allowedIps = new Set(["127.0.0.1", "::1", "::ffff:127.0.0.1"]);

app.get("/", (req, res) => {
  if (allowedIps.has(req.ip)) {
    res.send(`${FLAG}\n`);
  } else {
    res.status(403).send(`Access from ${req.ip} is not allowed\n`);
  }
});

app.get("/check", async (_req, res) => {
  const result = await fetch("http://localhost:3000/");
  res.send(`status: ${result.status} ${result.statusText}\n`);
});

app.listen(3000, () => {
  console.log("Server listening on port 3000");
});
```

## 偽装ヘッダを使用したバイパス

なので，偽装ヘッダ `X-Forwarded-For: 127.0.0.1` を付けてリクエストした場合の挙動は以下のようになります．

- **攻撃者**が `X-Forwarded-For: 127.0.0.1` という偽装ヘッダーを付けてNginxにアクセスする．
- **Nginx** は元のヘッダーの末尾に実際のIP (例: `203.0.113.5`) を追記し，バックエンドのExpressに `X-Forwarded-For: 127.0.0.1, 203.0.113.5` として転送する．
- **Express** (`trust proxy: true`) は，届いたヘッダーの**左端のIP** を見るため，`req.ip` に `127.0.0.1` を代入してしまう．
- アプリケーション側で `allowedIps.has(req.ip)` が突破され，`FLAG` がレスポンスされる．

```shell
$ curl -H "X-Forwarded-For: 127.0.0.1" http://34.170.146.252:42474
Alpaca{REDACTED}
```

---
## Post-Mortem & Dead ends

## References

[^1]: [プロキシの背後にあるエクスプレス · Express.js](https://expressjs.com/ja/guide/behind-proxies/)

[^2]: [Module ngx\_http\_proxy\_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
