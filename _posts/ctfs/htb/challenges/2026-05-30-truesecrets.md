---
title: HTB「TrueSecrets」Easy Writeup
description: メモリダンプファイルからの，C2サーバのソースコード特定と暗号化解除
date: 2026-05-30 22:30:00 +0900
categories: [CyberSecurity, CTF]
tags: [hackthebox, forensics]
---

# 20260530-htb-chall-forensics-easy-TrueSecrets

> - **Category**: Forensics
> - **Description**: 当社のサイバー犯罪部門は、数か月間、有名な APT グループを捜査してきました。このグループは、企業組織に対するいくつかの注目を集めた攻撃に関与してきました。しかし、このケースで興味深いのは、彼らが独自のカスタム コマンド & コントロール サーバーを開発したことです。幸いなことに、私たちの部隊はAPTグループのリーダーの家を襲撃し、電源が入っている間に彼のコンピューターのメモリキャプチャを取得することができました。キャプチャを分析して、サーバーのソース コードを見つけようとします。
> - **Tech Stack**: CSharp, TrueCrypt
> - **Keyword**: Windows
> - **Flag**: `{*** REDACTED ***}`
{: .prompt-info }


**階層構造**
```
.
└── TrueSecrets.raw

1 directory, 1 files
```

---
## Solution Path

### メモリダンプファイルの調査

C2サーバのソースコードを見つけるための調査を行います．
ファイルを `strings` に投げると，**windows** に関連するパスや名前が多く表示されるので，このメモリダンプファイルはwindows環境のものようです．

2つ気になるものを見つけました．

1. `2176 7zFM.exe "C:\Program Files\7-Zip\7zFM.exe" "C:\Users\IEUser\Documents\backup_development.zip"`
   - **根拠**: 既存のファイルではないため，対象のソースコードの可能性
2. `2128 TrueCrypt.exe "C:\Program Files\TrueCrypt\TrueCrypt.exe"`
   - **根拠**: **TrueSecrets** というチャレンジ名と近いため，関連性がある可能性

```shell
$ vol -f TrueSecrets.raw windows.cmdline

PID     Process Args
4       System  -
252     smss.exe        \SystemRoot\System32\smss.exe
320     csrss.exe       %SystemRoot%\system32\csrss.exe ObjectDirectory=\Windows SharedSection=1024,12288,512 Windows=On SubSystemType=Windows ServerDll=basesrv,1 ServerDll=winsrv:UserServerDllInitialization,3 ServerDll=winsrv:ConServerDllInitialization,2 ServerDll=sxssrv,4 ProfileControl=Off MaxRequestThreads=16
356     wininit.exe     -
368     csrss.exe       %SystemRoot%\system32\csrss.exe ObjectDirectory=\Windows SharedSection=1024,12288,512 Windows=On SubSystemType=Windows ServerDll=basesrv,1 ServerDll=winsrv:UserServerDllInitialization,3 ServerDll=winsrv:ConServerDllInitialization,2 ServerDll=sxssrv,4 ProfileControl=Off MaxRequestThreads=16
396     winlogon.exe    -
452     services.exe    C:\Windows\system32\services.exe
468     lsass.exe       C:\Windows\system32\lsass.exe
476     lsm.exe C:\Windows\system32\lsm.exe
584     svchost.exe     C:\Windows\system32\svchost.exe -k DcomLaunch
644     VBoxService.ex  C:\Windows\System32\VBoxService.exe
696     svchost.exe     C:\Windows\system32\svchost.exe -k RPCSS
752     svchost.exe     C:\Windows\System32\svchost.exe -k LocalServiceNetworkRestricted
864     svchost.exe     C:\Windows\System32\svchost.exe -k LocalSystemNetworkRestricted
904     svchost.exe     C:\Windows\system32\svchost.exe -k LocalService
928     svchost.exe     C:\Windows\system32\svchost.exe -k netsvcs
992     svchost.exe     -
1116    svchost.exe     C:\Windows\system32\svchost.exe -k NetworkService
1228    spoolsv.exe     C:\Windows\System32\spoolsv.exe
1268    svchost.exe     C:\Windows\system32\svchost.exe -k LocalServiceNoNetwork
1352    taskhost.exe    "taskhost.exe"
1448    dwm.exe -
1464    explorer.exe    C:\Windows\Explorer.EXE
1636    svchost.exe     C:\Windows\System32\svchost.exe -k utcsvc
1680    svchost.exe     C:\Windows\system32\svchost.exe -k LocalServiceAndNoImpersonation
1776    wlms.exe        -
1832    VBoxTray.exe    "C:\Windows\System32\VBoxTray.exe"
352     sppsvc.exe      -
1632    svchost.exe     -
856     SearchIndexer.  C:\Windows\system32\SearchIndexer.exe /Embedding
2128    TrueCrypt.exe   "C:\Program Files\TrueCrypt\TrueCrypt.exe"
2760    svchost.exe     C:\Windows\System32\svchost.exe -k secsvcs
2332    WmiPrvSE.exe    C:\Windows\system32\wbem\wmiprvse.exe
2580    taskhost.exe    -
2176    7zFM.exe        "C:\Program Files\7-Zip\7zFM.exe" "C:\Users\IEUser\Documents\backup_development.zip"
3212    DumpIt.exe      "C:\Users\IEUser\Downloads\DumpIt.exe"
272     conhost.exe     \??\C:\Windows\system32\conhost.exe "-180402527637560752-8319479621992226886-774806053592412399-20651748-1013740728
```

目星をつけたファイルをダンプし，調査します．
`backup_development.zip` の物理アドレスを調べ，`dumpfiles` プラグインで中身を抽出し，アーカイブを解除すると，`development.tc` という **TrueCrypt** 形式の暗号化ドライブファイルが入手できました．

```shell
$ vol -f TrueSecrets.raw windows.filescan | grep -i "backup_development.zip"
0xbbf6158  100.0\Users\IEUser\Documents\backup_development.zip

$ mkdir output && vol -f TrueSecrets.raw -o ./output windows.dumpfiles --physaddr 0xbbf6158               
Volatility 3 Framework 2.27.0
Progress:  100.00               PDB scanning finished
Cache   FileObject      FileName        Result
DataSectionObject       0xbbf6158       backup_development.zip  file.0xbbf6158.0x839339d0.DataSectionObject.backup_development.zip.dat
SharedCacheMap  0xbbf6158       backup_development.zip  file.0xbbf6158.0x9185db40.SharedCacheMap.backup_development.zip.vacb

$ file file.0xbbf6158.0x839339d0.DataSectionObject.backup_development.zip.dat
file.0xbbf6158.0x839339d0.DataSectionObject.backup_development.zip.dat: Zip archive data, at least v1.0 to extract, compression method=store

$ unzip file.0xbbf6158.0x839339d0.DataSectionObject.backup_development.zip.dat
Archive:  file.0xbbf6158.0x839339d0.DataSectionObject.backup_development.zip.dat
 extracting: development.tc

$ file development.tc
development.tc: data
```

### TrueCrypt パスワードのダンプ

`truecrypt` はメモリ内にパスフレーズを保持するため，プラグインを使用して抽出します．

```shell
$ vol -f TrueSecrets.raw windows.truecrypt.Passphrase
Volatility 3 Framework 2.27.0
Progress:  100.00               PDB scanning finished
Offset  Length  Password

0x89ebf064      28      X2Hk2XbEJqWYsh8VdbSYg6WpG9g7
```

**TrueCrypt** のパスフレーズ (`X2Hk2XbEJqWYsh8VdbSYg6WpG9g7`) を入手したので，`development.tc` に使用できるか試します．
実験用のWindows仮想環境に，**TrueCrypt** を導入して，パスフレーズを入力し，開くと `malware_agent` というディレクトリが入手できました．

`malware_agent/`
```
❯ tree .
.
├── AgentServer.cs
└── sessions
    ├── 5818acbe-68f1-4176-a2f2-8c6bcb99f9fa.log.enc
    ├── c65939ad-5d17-43d5-9c3a-29c6a7c31a32.log.enc
    └── de008160-66e4-4d51-8264-21cbc27661fc.log.enc

2 directories, 4 files
```

このプログラムは，感染させたマシン (Agent) からの接続を待ち受け，遠隔操作を行うための **C2サーバ** のようです．
また，**暗号化鍵と初期化ベクトル** の情報がハードコードされています．

- **待ち受けポート**: `40001` (TCP)
- **セッション管理**: 接続ごとに固有の `sessionID` (GUID) を生成する．
- **ログの保存**: 実行したコマンドとAgentから返ってきた実行結果（`cmdOut`）を結合し，**DES暗号 (CBC mode)** で暗号化した上で `sessions\<Guid>.log.enc` というファイルに追記していく．
- **base64**: ログは暗号化した上で，base64 エンコードされている．

`malware_agent/AgentServer.cs`
```csharp
using System;
using System.IO;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Security.Cryptography;

class AgentServer {
  
    static void Main(String[] args)
    {
        var localPort = 40001;
        IPAddress localAddress = IPAddress.Any;
        TcpListener listener = new TcpListener(localAddress, localPort);
        listener.Start();
        Console.WriteLine("Waiting for remote connection from remote agents (infected machines)...");
    
        TcpClient client = listener.AcceptTcpClient();
        Console.WriteLine("Received remote connection");
        NetworkStream cStream = client.GetStream();
    
        string sessionID = Guid.NewGuid().ToString();
    
        while (true)
        {
            string cmd = Console.ReadLine();
            byte[] cmdBytes = Encoding.UTF8.GetBytes(cmd);
            cStream.Write(cmdBytes, 0, cmdBytes.Length);
            
            byte[] buffer = new byte[client.ReceiveBufferSize];
            int bytesRead = cStream.Read(buffer, 0, client.ReceiveBufferSize);
            string cmdOut = Encoding.ASCII.GetString(buffer, 0, bytesRead);
            
            string sessionFile = sessionID + ".log.enc";
            File.AppendAllText(@"sessions\" + sessionFile, 
                Encrypt(
                    "Cmd: " + cmd + Environment.NewLine + cmdOut
                ) + Environment.NewLine
            );
        }
    }
    
    private static string Encrypt(string pt)
    {
        string key = "AKaPdSgV";
        string iv = "QeThWmYq";
        byte[] keyBytes = Encoding.UTF8.GetBytes(key);
        byte[] ivBytes = Encoding.UTF8.GetBytes(iv);
        byte[] inputBytes = System.Text.Encoding.UTF8.GetBytes(pt);
        
        using (DESCryptoServiceProvider dsp = new DESCryptoServiceProvider())
        {
            var mstr = new MemoryStream();
            var crystr = new CryptoStream(mstr, dsp.CreateEncryptor(keyBytes, ivBytes), CryptoStreamMode.Write);
            crystr.Write(inputBytes, 0, inputBytes.Length);
            crystr.FlushFinalBlock();
            return Convert.ToBase64String(mstr.ToArray());
        }
    }
}
```

### DES暗号の復号

鍵情報とログファイルは入手済みなため，**CyberChef** を用いてデコードします．
3つあるうちの最後のファイルをデコードすると，最後の行に **Flag** がありました．

- `From Base64`: Standard (RFC4048)
	- `Remove non-alphabet chars`
- `DES Decrypt`
	- `Key (UTF8)`: `AKaPdSgV`
	- `Key (UTF8)`: `QeThWmYq`
	- `Mode`: CBC
	- `Input`: Raw
	- `Output`: Raw

```
wENDQtzYcL3CKv0lnnJ4hk0JYvJVBMwTj7a4Plq8h68=
M35jHmvkY9WGlWdXo0ByOJrYhHmtC8O0hn+gLHaClb4QbACeOoSiYA==
hufGZi+isAzspq9AOs+sI/u+AS/aWPrAYd+mctDo7qEt+SpW2sELvSaxx6RRdK3vDavTsziAtb4/iCZ72v3QGh78yhY2KXZFu8qAcYdN7ltOOlg1LSrdkhjgr+CWTlvWh7A8IS7NwwI=
6ySb2CBt+Z1SZ4GlB7/yL4rJGeZ0WVaYW7N15aUsDAqzIYJWL/f0yw==
U2ltlIYcyGaSmL5xmAkEop+/f5MGUEWeWjpCTe5eStd/cg9FKp89l/EksGB90Z/hLbT44/Ur/6XL9aI27v0+SzaMFsgAeamjyYTRfLQk2fQlsRPCY/vMDj0FWRCGIZyHXCVoo4AePQB93SgQtOEkTQ2oBOeVU4X5sNQo23OcM1wrFrg8x90UOk2EzOm/IbS5BR+Wms1M2dCvLytaGCTmsUmBsATEF/zkfM2aGLytnu5+72bD99j7AiSvFDCpd1aFsogNiYYSai52YKIttjvao22+uqWMM/7Dx/meQWRCCkKm6s9ag1BFUQ==
+iTzBxkIgVWgWm/oyP/Uf6+qW+A+kMTQkouTEammirkz2efek8yfrP5l+mtFS+bWA7TCjJDK2nLAdTKssL7CrHnVW8fMvc6mJR4Ismbs/d/fMDXQeiGXCA==

Cmd: hostname
DESKTOP-MRL1A9O....¾.l-¦¶±ami
desktop-mrl1a9o\greg.........d0ÌM..c c:\users\greg\documents
 Volume in drive C is Windows 7
 Volume Serial Number is 1A9Q-0313.....ö..;.Ãî.ry of C:\Users\greg\Documents...Óvù.Kµ..22  09:07 AM    <DIR>          .
12/13/2022  09:07 AM    <DIR>          ..
12/13/2022  09:15 AM                41 flag.txt
               1 File(s)             41 bytes
               2 Dir(s)  25,326,063,616 bytes free.....´âÿ.ôIePe c:\users\greg\documents\flag.txt
{*** REDACTED ***}
```

---
## Post-Mortem & Dead ends

## References
