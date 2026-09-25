---
title: Dockerを使用したOpenFOAM-11導入の備忘録
description: WSL2 (Ubuntu 24.04) と Docker Desktop で OpenFOAM-11 + ParaView 5.10 の公式イメージを導入し，X11転送でGUIを表示する手順
date: 2024-09-17 12:00:00 +0900
categories: [Environment]
tags: [openfoam, docker, wsl]
---

## 導入環境

- Windows10 home
- Docker Desktop for Windows
- WSL2 on Ubuntu24.04LTS

## Pull image

[Image](https://hub.docker.com/r/openfoam/openfoam11-paraview510)

```bash
❯ docker pull openfoam/openfoam11-paraview510
Using default tag: latest
latest: Pulling from openfoam/openfoam11-paraview510
Status: Downloaded newer image for openfoam/openfoam11-paraview510:latest
docker.io/openfoam/openfoam11-paraview510:latest
```

## Docker で GUI を動作

- -e DISPLAY=$DISPLAY: ホスト側の DISPLAY 環境変数を Docker コンテナに渡します。
- -v /tmp/.X11-unix:/tmp/.X11-unix: ホストの X11 ソケットをコンテナにマウントします。  
  これにより、コンテナがホストの X サーバーにアクセスできるようになります。

```bash
docker run -it \
    --mount type=bind,source=/home/[username]/workspace/openfoam/simplified_inflow_model,target=/root \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix:/tmp/.X11-unix \
    openfoam/openfoam11-paraview510:latest
```
