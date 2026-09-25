---
title: HTB「CubeMadness1」VeryEasy Writeup
description: Il2Cppビルド形式でコンパイルされたunityゲームのPwn
date: 2026-05-17 16:18:00 +0900
categories: [CyberSecurity, CTF]
tags: [rev, hack_the_box, windows, game_pwn]
---

# 20260517-htb-chall-gamepwn-veryeasy-CubeMadness1

初めての，GamaPwnでした．

- **DnSpy v6.5.1 64-bit .NET**
- **Il2CppDumper:** [Il2CppDumper Online](https://il2cppdumper.com)
- **Ghidra**

---

## Surface Analysis

```
❯ tree .
.
├── baselib.dll
├── GameAssembly.dll
├── HackTheBox CubeMadness1_Data
│   ├── app.info
│   ├── boot.config
│   ├── globalgamemanagers
│   ├── globalgamemanagers.assets
│   ├── globalgamemanagers.assets.resS
│   ├── il2cpp_data
│   │   ├── etc
│   │   │   └── mono
│   │   │       ├── 2.0
│   │   │       │   ├── Browsers
│   │   │       │   │   └── Compat.browser
│   │   │       │   ├── DefaultWsdlHelpGenerator.aspx
│   │   │       │   ├── machine.config
│   │   │       │   ├── settings.map
│   │   │       │   └── web.config
│   │   │       ├── 4.0
│   │   │       │   ├── Browsers
│   │   │       │   │   └── Compat.browser
│   │   │       │   ├── DefaultWsdlHelpGenerator.aspx
│   │   │       │   ├── machine.config
│   │   │       │   ├── settings.map
│   │   │       │   └── web.config
│   │   │       ├── 4.5
│   │   │       │   ├── Browsers
│   │   │       │   │   └── Compat.browser
│   │   │       │   ├── DefaultWsdlHelpGenerator.aspx
│   │   │       │   ├── machine.config
│   │   │       │   ├── settings.map
│   │   │       │   └── web.config
│   │   │       ├── browscap.ini
│   │   │       ├── config
│   │   │       └── mconfig
│   │   │           └── config.xml
│   │   ├── Metadata
│   │   │   └── global-metadata.dat
│   │   └── Resources
│   │       └── mscorlib.dll-resources.dat
│   ├── level0
│   ├── Resources
│   │   ├── unity_builtin_extra
│   │   └── unity default resources
│   ├── RuntimeInitializeOnLoads.json
│   ├── ScriptingAssemblies.json
│   ├── sharedassets0.assets
│   └── sharedassets0.assets.resS
├── HackTheBox CubeMadness1.exe
├── UnityCrashHandler64.exe
└── UnityPlayer.dll

15 directories, 37 files
```

`il2cpp_data` というディレクトリ名からもわかるように，このバイナリファイルはUnityの **IL2CPP (Intermediate Language to C++)** というビルド形式でコンパイルされています．
### IL2CPP とは

- **従来のUnity製ゲーム:** C#コードは `Assembly-CSharp.dll` という中間言語 (IL) 形式のファイルにコンパイルされます．
  また，中間言語には関数名やクラス構造がそのまま綺麗に残っているため，**dnSpy** や **ILSpy** などのC#逆コンパイラに放り込むだけで，**開発者が書いた元のC#コードとほぼ同じ状態に復元** することができます．
- **IL2CPP製のUnityゲーム:** 開発者が書いたC#コードをUnityが一度C++のソースコードに変換します．
  そのC++コードを，各OSのコンパイラを使って，完全に生の機械語 (`GameAssembly.dll` や `libil2cpp.so` などのネイティブバイナリ) へとコンパイルします．
  このプロセスの中で，元のC#の **クラス構造，関数名，変数名** といった情報がバイナリから完全に消え，すべてアドレスへと置き換わってしまうことで，`GameAssembly.dll` をそのまま逆アセンブラで読み込んでもアドレスの羅列として表示され，解析が困難になります．

Unityはゲームを実行する，(アニメーションの紐付け，コンポーネントの検索等) ために，最低限の **クラス名や関数名** の情報を内部で保持しておく必要があります．
**IL2CPP** ビルドでは，この消えたはずのデータが `global-metadata.dat` という別のファイルに暗号化・カプセル化されて一括保存されています．

### Il2CppDumper とは

そこで，**Il2CppDumper** を使用します．
**Il2CppDumper** は，この **`GameAssembly.dll` (機械語)** と **`global-metadata.dat` (名前のリスト)** の2つを結合して解析し，以下のような解析のヘルプを自動生成してくれます．
- **`DummyDll/Assembly-CSharp.dll`**: 中身のロジックは空ですが，**元のゲームにどんなクラス・メソッドが存在し，それらがバイナリのどのアドレスに配置されているか** というフレーム構造だけを再現したC#のDLLファイルを復元します (これをdnSpyで読むことで，ゲームの構造を把握できます)．
- **`ghidra.py` / `ida.py`**: GhidraやIDA Proなどの逆アセンブラに読み込ませるためのスクリプトです．これを実行すると，バイナリ内の `sub_180A66fd0` となっていた無名関数が，自動的に `Cube$$OnTriggerEnter2D` のように **元の正しい関数名に一括リネーム** されます．

## Dynamic Analysis

`HackTheBox CubeMadness1.exe` を起動すると2Dのゲームプログラムが起動します．
どうにかして **20/20を取得(又は，クリア条件を改竄)** することができればFlagが入手できるのではないかと推測しました．

![2026-05-17-cube_madness1-01](/assets/img/ctf/htb/challenges/2026-05-17-cube_madness1-01.png)

## Static Analysis

今回は，オンラインで使用できる [Il2CppDumper Online](https://il2cppdumper.com) を使用します．
このサイトに，`GameAssembly.dll` と `global-metadata.dat` を渡して，出力されたzipの `DummyDll/Assembly-CSharp.dll` を **DnSpy** に渡します．

無名空間に存在する一部のクラスがハングル文字になっているのは，難読化の影響です．

解析環境の理由により，`ghidra.py` は使用せずに，**DySpy** でメソッドの **仮想アドレス (Virtual Address)** を読み，**ghidra** で，そのアドレスに **ジャンプ(`g`)** しメソッドを解析する方法を行います．
また，フラグの条件が **20/20** なため，**0x14** が出現しているメソッドや，当たり判定処理，関連しそうなメソッドを詳しく調べます．

- `Cube::Cube()`
- `CubeCounter::CubeCounter()`
- `FlagChack::FlagChack()`
- `Player::Update()`
- `가::OnTriggerEnter2D()`
- `각::Update()`
- `갂::Update()`

**無名空間{}**
```
// Types:  
//   
// <Module>  
// Cube  
// CubeCounter  
// FlagCheck  
// Player  
// 가  
// 각  
// 갂
```

`가::OnTriggerEnter2D()` というメソッドがあります．名称から考えると，これはUnityが標準で用意している **2Dの物理空間において，オブジェクト同士が接触した瞬間を検知して自動的に実行される** メソッドでないかと推測しました．
Cubeを取得する際の接触処理が記載されている可能性が高いため，このメソッドが存在する `0x180A66FD0` アドレスをghidraで解析します．
すると，45行目で，`**(int **)(DAT_180cfbc68 + 0xb8) = **(int **)(DAT_180cfbc68 + 0xb8) + 1;` という処理が記載されていました．これは，PlayerがCubeに接触した際に，スコアをインクリメントしている処理だと推測しました．

ここで，加算量にパッチを当てて一気に20にする方法がありますが，命令サイズの差によって命令の改竄を行えませんでした．
`INC dword ptr [RAX]` は `FF00` の2バイト構成ですが，`ADD dword ptr [RAX], 0x14` にすると，`83 00 14` となり3バイトになってしまいます．
もし **Patch Instruction** でそのまま3バイトの命令を打ち込むと，元々あった2バイトの枠からはみ出し，**次の命令の最初の1バイトを上書きして破壊** してしまいます．
なので，加算量ではなく，判定自体を改竄する方法を考えます．

`가::OnTriggerEnter2D()`
```c
void FUN_180a66fd0(undefined8 param_1,longlong param_2)

{
  char cVar1;
  longlong lVar2;
  undefined8 uVar3;
  
  if (DAT_180d5a7b5 == '\0') {
    thunk_FUN_1801192d0(&DAT_180cf71a8);
    thunk_FUN_1801192d0(&DAT_180cfbc68);
    thunk_FUN_1801192d0(&DAT_180cfbc18);
    DAT_180d5a7b5 = '\x01';
  }
  if (((*(byte *)(DAT_180cfbc18 + 0x133) & 4) != 0) && (*(int *)(DAT_180cfbc18 + 0xe0) == 0)) {
    il2cpp_runtime_class_init();
  }
  if (DAT_180d5a7a3 == '\0') {
    thunk_FUN_1801192d0(&DAT_180cfbc18);
    DAT_180d5a7a3 = '\x01';
  }
  if (((*(byte *)(DAT_180cfbc18 + 0x133) & 4) != 0) && (*(int *)(DAT_180cfbc18 + 0xe0) == 0)) {
    il2cpp_runtime_class_init();
  }
  lVar2 = *(longlong *)(*(longlong *)(DAT_180cfbc18 + 0xb8) + 0x238);
  if (lVar2 != 0) {
    if (*(int *)(lVar2 + 0x18) == 0) {
      uVar3 = thunk_FUN_18010f090();
                    /* WARNING: Subroutine does not return */
      FUN_180150c40(uVar3,0);
    }
    lVar2 = *(longlong *)(lVar2 + 0x20);
    if (lVar2 == 0) {
      if (((*(byte *)(DAT_180cfbc18 + 0x133) & 4) != 0) && (*(int *)(DAT_180cfbc18 + 0xe0) == 0)) {
        il2cpp_runtime_class_init();
      }
      lVar2 = FUN_180a68f30(0,0,6);
    }
    if (param_2 != 0) {
      cVar1 = FUN_1802d23f0(param_2,lVar2,0);
      if (cVar1 != '\0') {
        if (((*(byte *)(DAT_180cfbc68 + 0x133) & 4) != 0) && (*(int *)(DAT_180cfbc68 + 0xe0) == 0)) {
          il2cpp_runtime_class_init(DAT_180cfbc68);
        }
        **(int **)(DAT_180cfbc68 + 0xb8) = **(int **)(DAT_180cfbc68 + 0xb8) + 1;
        uVar3 = FUN_1802d2710(param_1,0);
        if (((*(byte *)(DAT_180cf71a8 + 0x133) & 4) != 0) && (*(int *)(DAT_180cf71a8 + 0xe0) == 0)) {
          il2cpp_runtime_class_init();
        }
        FUN_1802e3c10(uVar3,0);
      }
      return;
    }
  }
                    /* WARNING: Subroutine does not return */
  FUN_180150c70();
}
```

**インクリメント処理のアセンブリ**
```nasm
180a67119 ff 00  INC dword ptr [RAX]
```

まず，難読化されたクラスの各役割を推測します．

- `가`: 内部に，`OnTriggerEnter2D(Collider2D)` を持っていることから，ステージ上に配置されているキューブ (アイテム) 自体の挙動とアタリ判定を管理するクラス．
- `각`: メンバ変数に `UnityEngine.UI.Text` 型 **(難読化名：`각갟갱`)** を保持していたことから，画面左上に表示されているテキスト `(Cubes: X / 20)` のUI更新をリアルタイムに行うクラス．
- `갂`: ゲーム全体の進行と，フラグの出現条件を管理するクラス．

`갂` が (フラグ判定・ゲームクリア管理) になっている可能性が高いため，`갂::Update()` の `0x180A681A0` を調べます．
クリア判定で使用される **20 (0x14)** がありました．**Patch Instruction (Ctrl + Shift + G)** を用いて，`CMP dword ptr [RAX],0x14` を `CMP dword ptr [RAX],0x1` にパッチを当てます．

`갂::Update()`
```c
void FUN_180a681a0(longlong param_1)

{
  longlong lVar1;
  
  if (DAT_180d5a7c6 == '\0') {
    thunk_FUN_1801192d0(&DAT_180cfbc68);
    DAT_180d5a7c6 = '\x01';
  }
  if (((*(byte *)(DAT_180cfbc68 + 0x133) & 4) != 0) && (*(int *)(DAT_180cfbc68 + 0xe0) == 0)) {
    il2cpp_runtime_class_init(DAT_180cfbc68);
  }
  lVar1 = *(longlong *)(param_1 + 0x18);
  if (**(int **)(DAT_180cfbc68 + 0xb8) < 20) {
    if (lVar1 != 0) {
      FUN_180214f10(lVar1,0,0);
      return;
    }
  }
  else if (lVar1 != 0) {
    FUN_180214f10(lVar1,1,0);
    return;
  }
                    /* WARNING: Subroutine does not return */
  FUN_180150c70();
}
```

**クリア判定で比較される定数のアセンブリ**
```nasm
180a681f8  83 38 14  CMP dword ptr [RAX],0x14
```

### 改竄したDLLを実行ファイルに読み込ませる

**File>Export Program** から， **Format: Original File** を選び，改竄した `GameAssembly.dll` を `HackTheBox CubeMadness1.exe` と同じ階層に配置します．
`HackTheBox CubeMadness1.exe` を実行し，キューブを1つでも取得すると，Flagが表示されました．

![2026-05-17-cube_madness1-02](/assets/img/ctf/htb/challenges/2026-05-17-cube_madness1-02.png)

- **Flag:** `<confidential>`

---

## References




