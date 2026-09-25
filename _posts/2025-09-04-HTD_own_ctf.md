---
title: EC2の無料枠を用いてCTFプラットフォームを構築
description: AWS EC2 の無料枠 (t2.micro) と Docker で CTFd を立ち上げ，自前のCTFプラットフォームを構築する手順．セキュリティグループやSSH接続の設定，テーマ変更まで
date: 2025-09-04 12:00:00 +0900
categories: [Infrastructure & OS]
tags: [deploy]
---


## 開発環境

- AWS EC2
- Docker
- CTFd & CTFd themes
- Host: WSL2 ubuntu

## EC2 インスタンスの設定

### インスタンス構成

- Instance type: t2.micro
- OS image: Ubuntu Server 24.04 LTS
- Strage: 汎用 SSD 25Gib
- Connection: SSH ログインキーペア
- 自分の IP からの SSH を許可
- パブリック IP の自動割り当てを許可
- http, https からのインバウンドトラフィックを許可


> セキュリティグループの設定で必ず，既知のIPからの通信のみに絞ってください．<br>
  基本的には，0.0.0.0/0の設定を使用しないでください．<br>
  学校などで使用する場合は，学内IPに絞るようにしてください．
{: .prompt-warning }

### SSH 接続

SSH のログインキーペアを作成したときにダウンロードされた .pem を`~/.ssh`以下に配置してください．<br>
`~/.ssh/conifg`に接続設定を書き込み，ssh 接続を開始

- ~/.ssh/config

```txt
Host MY_CTF_S
    HostName [public IP]
    User ubuntu
    IdentityFile ~/.ssh/MyCTFServer_SSH.pem
    ServerAliveInterval 60
```

```zsh
$ ssh MY_CTF_S

ubuntu@ip-172-31-46-26:~$ sudo nano /etc/ssh/sshd_config
# custom settings
ClientAliveInterval 60
ClientAliveCountMax 3
```

## CTFd のインストールとテーマの設定

今回は[CTFd](https://github.com/CTFd/CTFd)と[CTFd Themes](https://github.com/CTFd/themes)を利用します．<br>
![own_ctf_01](assets/img/2025/how_to_build_own_ctf_01.png)

```bash
ubuntu@ip-172-31-46-26:~$ git clone https://github.com/CTFd/CTFd.git

ubuntu@ip-172-31-46-26:~/CTFd/CTFd$ git clone https://github.com/chainflag/ctfd-neon-theme.git themes/neon

# Dockerの導入を行っておいてください (https://docs.docker.com/engine/install/ubuntu/)
ubuntu@ip-172-31-46-26:~/CTFd/CTFd$ sudo docker compose up
```

コンテナを起動し，EC2 のパブリック IP にアクセスすると CTFd のサイトが表示されます．<br>

スタイルタブから先ほどインストールした`neon`テーマを選択することで，導入が完了しました．

![own_ctf_02](assets/img/2025/how_to_build_own_ctf_02.png)