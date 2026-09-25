---
title: Zola SSGのGithub Pagesデプロイ方法
description: 静的サイトジェネレーター Zola で作ったサイトを GitHub Actions で GitHub Pages にデプロイし，カスタムドメインとHTTPSを設定する方法
date: 2025-09-28 12:00:00 +0900
categories: [Infrastructure & OS]
tags: [deploy]
---

## 使用技術

- zola
- zola-hacker theme
- X-Domain
- Github Pages
- Github Actions

## カスタムドメインの DNS レコード設定

まずは，ドメインを取得した後，ネームサーバーを有効化します．
![zola_01](/assets/img/2025/how_to_deploy_zola_on_github_pages_02.png)
次は，任意のサブドメインに対する CNAME レコードと GithubPages の 4 つの A レコードを設定します．
![zola_02](/assets/img/2025/how_to_deploy_zola_on_github_pages_01.png)
Github Pages で`DNS check successful`と表示されれば，正常に動作するはずです．<br>
また，https を設定する場合は`Enforce HTTPS`を有効化してください．
![zola_03](/assets/img/2025/how_to_deploy_zola_on_github_pages_03.png)

## Zola のデプロイ設定

> 任意の Zola テーマで初期セットアップが終了していることを前提としています．

- [Zola Github Pages Deploy](https://www.getzola.org/documentation/deployment/github-pages/)

### Step1: Static ディレクトリに`CNAME`ファイルを作成し，カスタムドメインを記載

- static/CNAME

```txt
blog.fit-isec.net
```

### Step2: `config.toml`の`base_url`をカスタムドメインに書き換え

- config.toml

```toml
# The URL the site will be built for
base_url = "https://blog.fit-isec.net/"
```

### Step3: [`zola-deploy-action`](https://github.com/shalzz/zola-deploy-action)を使用して，権限を付与

プロジェクトの設定から，`Actions>General>Workflow permissions`を選び，`Read and write permissions`を有効化する．<br>

zola-deploy-action をプロジェクトの Actions にコピーし，Push すると自動的にデプロイされます．
