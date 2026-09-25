---
title: HTB 「The Puppet Master」VeryEasy Writeup
description: osint入門問題
date: 2026-06-06 22:02:00 +0900
categories: [CyberSecurity, CTF]
tags: [hack_the_box, osint]
math: true
---

# 20260606-htb-chall-osint-very_easy-The_Puppet_Master

## Summary

本問は，OSINTの入門として軍事車両の画像から関連情報を調べるものです．

> - **Category**: OSINT
> - **Description**:
>   - 研修生各位、仮想通貨の資金の流れに関する調査の結果、RivalTech Corporationにたどり着きました。同社の企業サーバーで最近発生したデータ漏洩により、社内通信、従業員記録、および戦略計画文書が流出しています。
>   - あなたの任務は、BreachScopeデータベースを調査し、XyloPhone社に対する大規模な偽レビューキャンペーンを主導した上級幹部を特定することです。直接的な承認、予算の承認、およびキャンペーンの調整に関する証拠を探してください。
>   - 調査結果は、以下の形式で提出してください：HTB{幹部名}. 例：HTB{CEO_Thomas_Wilson_Approved_Budget}
>   - 研修部 デジタルフォレンジック課
> - **Tools & TechStack**:
> 	- Google
> 	- X
> - **Release**: 2025/09/26
{: .prompt-info }

---
## OSINT

サイトにアクセスすると，とある軍事車両の画像が与えられました．
この画像をもとに，検索エンジンで調べると，**x**[^1] の投稿がヒットしました．この投稿から車両名が判明したため，Wikipedia[^2] で検索するとその他の問題も解けました．
最終的に，すべて解き終わるとFlagが入手できました．

### 1. Vehicle Identification

- **問題概要**: この画像に写っているのはどのような軍用車両でしょうか？車両の特徴を見てみましょう。車輪付きで、装甲が施されており、人員輸送車のように見えます。インターネットで似たような車両について調べてみましょう。
- **答え**: `bushmaster`

### 2. Manufacturer Identification

- **問題概要**: この車両の製造元・設計者はどこですか？この特定の装甲車を設計・製造している企業について調べてください。
- **答え**: `Thales Australia`

### 3. Service History

- **問題概要**: この車両はいつ初めて実戦配備されたのでしょうか？この特定の車種が初めて実戦配備された年について調べてください。
- **答え**: `1997`

### 4. Country of Origin

- **問題概要**: この車両の原産国はどこですか？この特定の車両がどこで設計・製造されたのか調べてください。
- **答え**: `Australia`

### 5. Vehicle Capacity

- **問題概要**: この車両の定員はいくつですか？乗客と乗務員を合わせて何人まで乗せられるか調べてください（形式：乗客X名、運転手Y名）
- **答え**: `9 passengers and 1 driver`

---
## Post-Mortem & Dead ends

初めてOSINTやった~

## References

[^1]: [X](https://x.com/i/status/1661521287137865728)

[^2]: [ブッシュマスター (装甲車) - Wikipedia](https://ja.wikipedia.org/wiki/%E3%83%96%E3%83%83%E3%82%B7%E3%83%A5%E3%83%9E%E3%82%B9%E3%82%BF%E3%83%BC_(%E8%A3%85%E7%94%B2%E8%BB%8A))