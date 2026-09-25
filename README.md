# Chirpy Starter

[![Gem Version](https://img.shields.io/gem/v/jekyll-theme-chirpy)][gem]&nbsp;
[![GitHub license](https://img.shields.io/github/license/cotes2020/chirpy-starter.svg?color=blue)][mit]

When installing the [**Chirpy**][chirpy] theme through [RubyGems.org][gem], Jekyll can only read files in the folders
`_data`, `_layouts`, `_includes`, `_sass` and `assets`, as well as a small part of options of the `_config.yml` file
from the theme's gem. If you have ever installed this theme gem, you can use the command
`bundle info --path jekyll-theme-chirpy` to locate these files.

The Jekyll team claims that this is to leave the ball in the user’s court, but this also results in users not being
able to enjoy the out-of-the-box experience when using feature-rich themes.

To fully use all the features of **Chirpy**, you need to copy the other critical files from the theme's gem to your
Jekyll site. The following is a list of targets:

```shell
.
├── _config.yml
├── _plugins
├── _tabs
└── index.html
```

To save you time, and also in case you lose some files while copying, we extract those files/configurations of the
latest version of the **Chirpy** theme and the [CD][CD] workflow to here, so that you can start writing in minutes.

## Usage

Check out the [theme's docs](https://github.com/cotes2020/jekyll-theme-chirpy/wiki).

**run server**: bundle exec jekyll serve -> `nix-shell`
```
$ nix-shell -p bundler bundix
$ bundle install

$ bundle exec jekyll clean
$ bundle exec jekyll s
```

## カテゴリとタグの運用ルール

### categories (1記事につき1つ．サブカテゴリは CyberSecurity のみ)

| categories | 内容 |
| --- | --- |
| `[CyberSecurity, CTF]` | CTF の Writeup |
| `[CyberSecurity, Bughunt]` | 脆弱性報告 (GHSA 等) |
| `[CyberSecurity, MalwareAnalysis]` | マルウェア解析 |
| `[Development]` | 自作プロダクトの紹介 (projects タブに表示される) |
| `[MinecraftModding]` | MOD 開発の手順・Tips |
| `[Environment]` | 環境構築・ツールの設定・トラブル対処 |
| `[Learning]` | 学習記録・入門・資格 |

### tags (小文字の snake_case．1記事につき 1〜4 個)

- **CTF**: `出題元` + `ジャンル` + (任意で) `脆弱性の種類`
  - 出題元: `alpacahack`, `hackthebox`, `picoctf`, `overthewire`, `cryptohack`, `pwnable_kr`, `vulnhub`
  - ジャンル: `web`, `pwn`, `rev`, `crypto`, `misc`, `forensics`, `osint`, `boot2root` (マシン攻略)
- **Bughunt**: `言語` + `ghsa` + `脆弱性の種類`
- **脆弱性の種類** (CTF・Bughunt 共通): `xss`, `path_traversal`, `sql_injection`, `use_after_free`, `buffer_overflow`, `rsa`, `hardcoded_credentials`, `dos`
- **その他**: 言語・OS・ツール名 (`rust`, `java`, `linux`, `wsl`, `docker`, `neoforge` 等)
  - バージョン番号はタグに含めない (`neoforge_26.1.2` ではなく `neoforge`)
  - 自作プロダクトの種類は `cli` / `web_app` (`web` は CTF のジャンル用)

新しいタグを作る前に，既存のタグで表せないかを確認する．

## Contributing

This repository is automatically updated with new releases from the theme repository. If you encounter any issues or want to contribute to its improvement, please visit the [theme repository][chirpy] to provide feedback.

## License

This work is published under [MIT][mit] License.

[gem]: https://rubygems.org/gems/jekyll-theme-chirpy
[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[CD]: https://en.wikipedia.org/wiki/Continuous_deployment
[mit]: https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE
