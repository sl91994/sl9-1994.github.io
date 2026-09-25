---
title: ホストブラウザの通信を，WSL2上のNixOSで動作するBurpsuiteでインターセプトする方法
description: WSL2 上の NixOS で動かす Burp Suite で Windows ホストのブラウザ通信をインターセプトする方法．全インターフェースでの待ち受け，CA証明書の導入，ミラーモードネットワークの設定
date: 2026-04-29 20:25:00 +0900
categories: [Environment]
tags: [wsl, burpsuite]
---

WSL2上のNixOSで，名前付きdevshell内で使用しているburpsuiteでホストブラウザの通信をインターセプトする方法

---
## Request Listeners の変更 (恒久的)

```nix
(pkgs.writeShellScriptBin "burpsuite" ''
  # 一時ディレクトリに設定用JSONを書き出す
  CONFIG_FILE="/tmp/burp_proxy_all_interfaces.json"
  cat <<EOF > "$CONFIG_FILE"
  {
      "proxy":{
          "request_listeners":[
              {
                  "certificate_mode":"per_host",
                  "listen_mode":"all_interfaces",
                  "listener_port":8080,
                  "running":true,
                  "support_http2":true
              }
          ]
      }
  }
  EOF
  
  # 元のburpsuiteバイナリを、生成した設定ファイルを付与して実行
  exec ${pkgs.burpsuite}/bin/burpsuite --config-file="$CONFIG_FILE" "\$@"
'')
```

## CA証明書のインポート
https通信でも傍受できるように，`127.0.0.1:8080` にアクセスして証明書をダウンロードし，使用しているブラウザにインポートします．
必ず，`信頼されたルート証明機関` としてインポートします．

また，このままではWSL2が再起動したときに割り当てられているIPが変更されてしまいます．
毎回，設定しなおすのは面倒なのでネットワークのミラー化を行います．

## WSL2 ミラー化ネットワークの有効化

ミラーモードを使用することで，WindowsホストとWSL2がネットワークインターフェースを完全に共有することができます．
このおかげで，`127.0.0.1:8080` を介したインターセプトが可能になります．

`C:\Users\<username>\.wslconfig`
```txt
[wsl2]
networkingMode=mirrored
```

```powershell
PS:> wsl --shutdown
```

### Foxy Proxy の設定

`Foxy Proxy` に，設定を行えば完了です．
タブ上部の `ping` を使って正常なレスポンスが返れば使用できます．

![ENV-Intercepting_host_traffic_in_Burp_Suite_on_WSL2_NixOS-01](/assets/img/2026/ENV-Intercepting_host_traffic_in_Burp_Suite_on_WSL2_NixOS-01.jpeg)