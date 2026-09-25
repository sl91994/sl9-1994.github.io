---
title: NeoForgeでパーティクルの描画範囲を拡張
description: NeoForge の MOD 開発で，sendParticles の overrideLimiter と alwaysShow を使ってパーティクルの描画距離の制限を外し，遠くまで表示する方法
date: 2025-08-30 12:00:00 +0900
categories: [MinecraftModding]
tags: [neoforge, java]
---

### パーティクルの描画可能範囲を拡張する

パーティクルでレーザー等を作成するとき，デフォルトのパーティクル描画では決まった範囲しか描画が行われません．  
これを解決する方法は以下です．

```java
serverLevel.sendParticles(
        player,
        ParticleTypes.ENCHANT,
        true, // overrideLimiter - 通常のパーティクル制限を無視
        true, // alwaysShow - 描画距離制限を無視
        pos.x, pos.y, pos.z,
        1, 0, 0, 0, 0
);
```
