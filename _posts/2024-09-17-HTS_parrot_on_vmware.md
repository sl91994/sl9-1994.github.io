---
title: VMwareに対してのParrotOSの導入方法
description: VMware Workstation Pro 17 に Parrot OS Security Edition をインストールする手順．ISOのハッシュ検証，VM設定，fcitx-mozcでの日本語化，キー入力遅延の対処まで
date: 2024-09-16 12:00:00 +0900
categories: [Environment]
tags: [linux, vmware]
---

## 環境

- VMware Workstation Pro17  
- Windows10 Home

## インストール方法

### Parrot の ISO をダウンロード

[parrot OS](https://www.parrotsec.org/download/)

Live の Security バージョンを選択
![parrot_01](/assets/img/2024/how_to_install_parrot_on_vmware_01.png)

今回はペンテスト・CTF に使用する予定なので，Security バージョンをダウンロードします．
HackTheBox で使用する場合は HTB でも大丈夫です．
![parrot_02](/assets/img/2024/how_to_install_parrot_on_vmware_02.png)

ISO をダウンロード，自身がダウンロードした ISO の Hash を確認する．
![parrot_03](/assets/img/2024/how_to_install_parrot_on_vmware_03.png)

PowerShell を起動し，Hash が一致しているかを確認する．  
**ダウンロード時に破損している場合があるので，一応やっておきます**

```
powershell> certutil -hashfile .\Downloads\Parrot-security-6.1_amd64.iso SHA512
SHA512 ハッシュ (対象 .\Downloads\Parrot-security-6.1_amd64.iso):
5a66151241ec4b6e56253542281b92a22e6a4ecf1bfb628867a49b8ba7795cd4692c4a1cc43f32d3f41c96abc716703faeb976ce86284d49275c000950ca2416
CertUtil: -hashfile コマンドは正常に完了しました．
```

### VMware にインストール

VMware を起動し，「新規仮想マシンの作成」をクリックします．  
標準インストールを選択し，ISO のパスを指定します．  
Linux Debian 12.x (64bit)を選択し VM 名を決定
ディスクサイズは 60GB に設定します．  
**後々困らないように 50GB 以上設定することをお勧めします**
![parrot_04](/assets/img/2024/how_to_install_parrot_on_vmware_04.png)

また，RAM は 4096MB(4GB)・CPU は 4 プロセッサコアを割り当てます．
**性能に余裕がある方は，もっと割り当てても大丈夫です．**

### ParrotOS の設定

VMware を起動し，Desktop の Install Parrot を実行し，初期セットアップを行います．  
![parrot_05](/assets/img/2024/how_to_install_parrot_on_vmware_05.png)

インストール完了すると再起動を促すダイアログが表示されるので，再起動するとゲストから抜けインストールが完了します．

## 日本語化

```bash
> sudo apt install fcitx-mozc

> reboot
```

fctix
![parrot_06](/assets/img/2024/how_to_install_parrot_on_vmware_06.png)

私は英字キーボードで日本語入力を行いたいため，Generic 101 key の英語と mozc を設定しています．

![parrot_07](/assets/img/2024/how_to_install_parrot_on_vmware_07.png)

![parrot_08](/assets/img/2024/how_to_install_parrot_on_vmware_08.png)

### キー入力遅延の解消(オプション)

VM 内でキー入力を行う際に，遅延が発生する場合があります．
下記の解決策は私の環境ではうまくいきましたが，読者の方の環境で動作するかは不明です．

#### [Heinz Wrobel さんによる解決策](https://community.broadcom.com/vmware-cloud-foundation/discussion/ws-1761-keyboard-lag-with-ubuntu-guest)
