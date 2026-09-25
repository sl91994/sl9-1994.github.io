---
title: Windows10 Inetsimが正常に動作しない
description: Windows10 と REMnux のマルウェア解析環境で INetSim の偽ページが表示されない問題の解決方法．原因はプロキシ設定の自動検出 (WPAD)
date: 2025-08-19 12:00:00 +0900
categories: [CyberSecurity, MalwareAnalysis]
tags: [windows]
---

## 発生した問題

Windows10 & Remnux(Ubuntu 22.04)でマルウェア解析を行うための環境構築を行った．
攻撃者端末で inetsim を起動，疑似ネットワークを構築し，適切に DNS サーバーを設定したにもかかわらず，
ブラウザでのアクセス時に inetsim の fakepage が表示されない．

## 解決策

- Automatically Detect Settings を Off にする．

1. Windows メニューから Settings を開く．
2. Network & Internet を開き，Proxy に移動後，Automatically Detect Settings を Off にする．

> この設定が ON になっていたことにより，意図しないプロキシ設定が適用され，DNS サーバーを利用して正常に接続されなかった？

### Automatically Detect Settings

このオプションを有効にすると，Web Proxy Auto Discovery（WPAD）プロトコルを使用して，ネットワーク接続のプロキシ設定を自動的に検出および構成します．  
Internet Explorer やその他の Windows コンポーネントではデフォルトで有効になっているため，ネットワーク環境で特定の手動設定が必要な場合に問題が発生することがあります．