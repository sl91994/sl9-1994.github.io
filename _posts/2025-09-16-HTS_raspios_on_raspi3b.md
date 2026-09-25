---
title: RaspberryPI3 modelBにRaspberryOSを導入
description: Raspberry Pi Imager で Raspberry Pi 3 Model B に Raspberry Pi OS を書き込み，Wi-Fiと公開鍵認証のSSHを初期設定してヘッドレスで接続する手順
date: 2025-09-16 12:00:00 +0900
categories: [Infrastructure & OS]
tags: [raspberry_pi]
---

## OS のインストール

[RaspberryPI Imager](https://www.raspberrypi.com/software/)

Imager をインストールし，PC に **microSD card** を接続します．<br>
再現のために USB でやっていますが，ストレージは接続した SD card を設定してください．<br>
今回は，128GB の SD を使用しますが，64GB 以上を使用する場合は以下のオプションを行ってください．<br>

フォーマットが完了したら，使用したいラズベリーパイのバージョンを選択し，OS を選択してから`次へ`を選択します．<br>
![raspi_01](/assets/img/2025/how_to_install_raspberryos_on_raspberrypi3b_01.png)
`設定を編集する`から一般タブでホスト名や WIFI の SSID，パスワードを設定します．

### SSH 接続の設定

`サービスタブ`に移動し，`SSHを有効化`し`公開鍵認証のみを許可する`を選びます．<br>
次は，ホスト OS のターミナルに移動し，`ssh-keygen`を実行します．<br>
私の環境では，WSL2 で行っています．

```zsh
$ ssh-keygen

Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/xxxx/.ssh/id_ed25519): /home/xxxx/.ssh/id_ed25519_raspi
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/xxxx/.ssh/id_ed25519_raspi
Your public key has been saved in /home/xxxx/.ssh/id_ed25519_raspi.pub
The key fingerprint is:
SHA256:XXXX xxxx@DESKTOP-XXXXXX
The key's randomart image is:
+--[ED25519 256]--+
|.o..o=+==X@+=+.o.|
|..=o*.o.+O=o. o  |
|ooE+ .  B+.. o   |
|.o .   . =.      |
|        S .      |
|         o       |
|                 |
|                 |
|                 |
+----[SHA256]-----+
```

```zsh
$ cat ~/.ssh/id_ed25519_raspi.pub

ssh-ed25519 XXXX xxxx@DESKTOP-XXXXXX <--- 公開鍵
```

生成された，公開鍵を`authorized_keys`に設定し，保存を押し`はい`を選択し，OS の書き込みが終われば，完了です．<br>
あとは，RaspberryPI の背面のスロットに SD カードを差し込み，起動すると自動で WIFI に接続され，SSH クライアントでの接続が可能になります．<br>

![raspi_02](/assets/img/2025/how_to_install_raspberryos_on_raspberrypi3b_03.png)

```zsh
$ ssh -i /home/yurari/.ssh/id_ed25519_raspi st1@192.xxx.xx.xx
```

### SCPを用いて，ホスト-ゲスト間でファイル送受信

```zsh
scp -i ~/.ssh/id_ed25519_raspi_st1 st1@192.xxx.xx.xx:/home/[username]/[file_path] [ホスト側のパス]
```

### オプション: 64GB 以上の SD を使用する場合

![raspi_03](/assets/img/2025/how_to_install_raspberryos_on_raspberrypi3b_02.png)
OS のメニューで，FAT32 でフォーマットしてください．