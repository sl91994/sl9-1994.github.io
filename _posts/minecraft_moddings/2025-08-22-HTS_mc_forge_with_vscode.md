---
title: VScodeを用いたMinecraft Forge開発環境構築
description: VSCode で Minecraft Forge 1.21.4 のMOD開発環境を構築する手順．Extension Pack for Java，Eclipse Temurin JDK21，Forge MDK，Parchment マッピングの導入まで
date: 2025-08-22 12:00:00 +0900
categories: [MinecraftModding]
tags: [forge_1.21.4]
---

# Forge Mod 開発環境の構築

## VScode 拡張機能の導入

IntelliJ Idea 等であれば必要ないのですが，VScode で Java 開発を行うのであれば下記の拡張機能が必要です．  
この拡張機能を入れることで Forge 公式から推奨されている「Gradle for Java」拡張機能も同時にインストールされます．


> [Extension Pack for Java](https://marketplace.visualstudio.com/items?itemName=vscjava.vscode-java-pack)
{: .prompt-info }

## JDK の導入

1.21.x バージョンであれば Java 21 Development Kit(JDK)および，64bit Java 仮想マシン(JVM)のインストールが必要です．  
Forge 公式は[Eclipse Temurin](https://adoptium.net/temurin/releases?version=21&os=windows&arch=x64)を推奨しているので使用します．
JDK21-LTS を選択し，アーキテクチャに合わせた JDK の msi をダウンロードします．

![forge_01](/assets/img/minecraft_modding/minecraft_modding_with_forge_01_01.png)

念のため，ハッシュ値の確認も行っておきます．  
右側の checksum から正しいハッシュを確認し，cmd の certUtil コマンドでダウンロードされたファイルのハッシュと比較します．

```bash
certUtil -hashfile OpenJDK21U-jdk_x64_windows_hotspot_21.0.8_9.msi sha256
SHA256 ハッシュ (対象 OpenJDK21U-jdk_x64_windows_hotspot_21.0.8_9.msi):
d82030e8689b19efedfbce50ce38351ca81b302c06936584c6a27bda18339df8 # 公式: d82030e8689b19efedfbce50ce38351ca81b302c06936584c6a27bda18339df8
CertUtil: -hashfile コマンドは正常に完了しました。
```

ハッシュの確認が終われば，msi を実行します．
この時，「Set JAVA_HOME variable」を「ローカルハードドライブにインストール」を必ず選択してください．

![forge_02](/assets/img/minecraft_modding/minecraft_modding_with_forge_01_02.png)

> JDKの導入後に，必ずコンピュータを再起動してください．<br>
  再起動しないと環境変数が読み込まれず，gradlewコマンドが正常に動かない可能性があります．
{: .prompt-danger }

## Forge MDK の導入

MOD を作りたいバージョンの [Forge](https://files.minecraftforge.net/net/minecraftforge/forge/) MDK をダウンロード

今回は，現時点で Forge が対応している **Recommended** の最新版である 1.21.4 の MDK を使用します．

- MDK の zip を解凍し，解凍後のデフォルトのフォルダ名からプロジェクト名に変更します．
- 解凍された MDK の中身にある changelog.txt, CREDITS.txt, LICENSE.txt, README.txt は必要ないので削除しても構いません．

プロジェクトフォルダを VScode で開き，下記のコマンドを実行してください．

```bash
> ./gradlew :genVSCodeRuns
BUILD SUCCESSFUL in 3m 13s
8 actionable tasks: 8 executed
```

## Parchment MC の導入(難読化の再マッピング)

[Parchment MC](https://parchmentmc.org/docs/getting-started)から，「Install the mappings」セクションの通りに導入を行ってください．
注意として，gradle.properties を書き換える際に自身の Minecraft Version に合致した Mapping Version を，書くようにしてください．  
version の構文は「mappging_version-minecraft_version」です．  
例: `mapping_version=2025.03.23-1.21.4`

```gradle
// gradle.properties
# The mapping version to query from the mapping channel.
# This must match the format required by the mapping channel.
mapping_version=2025.03.23-1.21.4
```

最後に，インポートの解決とマッピング実行のために，再ビルドとソースの読み込みを行います．  
再ビルド後にエディタの再起動か「Java: Reload Projects」を行い，最後に./gradlew :runClient を行い正常に MOD が読み込まれていれば，成功です．

```
> ./gradlew --refresh-dependencies

> ./gradlew :runClient
```

![forge_03](/assets/img/minecraft_modding/minecraft_modding_with_forge_01_03.png)

これで，VScode で Minecraft Mod 開発を行うための環境構築が完了しました！

### 開発において便利な機能

- 継承関係をみたいクラスを選択し，右クリックから`Java: Open Type Hierarchy`を使用するとそのクラスを継承する全てのクラスをみることができます．
- クラス内で右クリックから`ソースアクション`から`Override/Implement Methods`を使用すると，そのクラスで実装可能なメソッドやオーバーライド可能なメソッドがリスト表示され，チェックをして OK を押すと自動でテンプレートが実装されます．

## 参考資料

- [How to setup Visual Studio Code to create Minecraft forge](https://medium.com/@bruce.kristelijn/how-to-setup-visual-studio-code-to-create-minecraft-forge-d85e1e24eeb2)
- [Forge docs](https://docs.minecraftforge.net/en/latest/gettingstarted/)