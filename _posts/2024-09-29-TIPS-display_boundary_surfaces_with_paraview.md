---
title: ParaView5.10.1でのフィールドの境界面の表示
description: ParaView 5.10.1 で Slice と Contour フィルタを使い，OpenFOAMの計算結果から alpha.water などのフィールドの境界面 (等値面) を表示する方法
date: 2024-09-29 12:00:00 +0900
categories: []
tags: [openfoam]
---

## Slice の作成

![paraview_01](/assets/img/2024/how_to_display_boundary_surfaces_with_paraview_01.png)

**paraview を起動し，計算領域を横断する面を作成します．**  
ナビゲーションメニューから，filters -> common -> slice と進みます．  
![paraview_02](/assets/img/2024/how_to_display_boundary_surfaces_with_paraview_02.png)

CameraNormal を使用すると，カメラの向きに対して直交する slice 面が生成されます．  
![paraview_03](/assets/img/2024/how_to_display_boundary_surfaces_with_paraview_03.png)

slice を生成したら，filters -> common -> contour から Contour を生成し，ContourBy を境界面を取得したいフィールドに設定します．  
今回は，alpha.Water の境界面を表示したいので，ContourBy に alpha.Water を設定し，isosurfaces に 0.5 と入力し等値面を生成します．  
物理量の分布や物理量の境界などをより複雑に視覚化したい方は，この値を変えてみてください．
![paraview_04](/assets/img/2024/how_to_display_boundary_surfaces_with_paraview_04.png)
