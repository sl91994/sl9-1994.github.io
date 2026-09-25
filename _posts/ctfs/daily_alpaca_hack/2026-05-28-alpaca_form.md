---
title: Daily-AlpacaHack 「Alpaca Form」Medium Writeup
description: .Netプログラムの逆解析とRVA埋め込み
date: 2026-05-29 06:15:00 +0900
categories: [CyberSecurity, CTF]
tags: [daily_alpaca_hack, rev, vuln/hardcoded_credentials, windows, dotnet]
---

# 20260528-daily_alpaca-rev-medium-Alpaca_Form

> - **Category**: Rev
> - **Description**: "この問題はLinux環境だけでも解けます。"
> - **Tech Stack**: CSharp, .Net
{: .prompt-info }

**階層構造**
```
.
├── AlpacaForm.deps.json
├── AlpacaForm.dll
├── AlpacaForm.exe
└── AlpacaForm.runtimeconfig.json

1 directory, 4 files
```

---
## Solution Path

### 表層解析

```shell
$ file AlpacaForm.dll
AlpacaForm.dll: PE32+ executable (GUI) x86-64 Mono/.Net assembly, for MS Windows, 2 sections

$ file AlpacaForm.exe
AlpacaForm.exe: PE32+ executable (GUI) x86-64, for MS Windows, 6 sections
```

### 動的解析

デコンパイルする前に，実際に `.exe` を起動し，大まかな動作を確認します．
起動すると，**GUI** が表示され，ただ入力された **Flag** 文字列が正解であるかを判定するだけのようです． 

### 静的解析

`AlpacaForm.dll` を，**DnSpy**[^1] に渡して，デコンパイルします．
難読化などが一切されていないため，非常にクリーンなコードが復元できました．
明らかにフラッグ判定を行う名前である `IsValidFlag(String)` を確認します．

`PrivateImplementationDetails` 型に埋め込まれた静的文字列と，入力した **Flag** 文字列を一文字ずつ `u (0x75/117)` で **XOR** しています．
なので，埋め込まれた静的文字列が分かれば，**Flag** を復元することが可能です．

`{} AlpacaForm::AlpacaForm::IsValidFlag(String)`
```csharp
// AlpacaForm.AlpacaForm
// Token: 0x06000003 RID: 3 RVA: 0x00002094 File Offset: 0x00000294
[NullableContext(1)]
private unsafe static bool IsValidFlag(string flag)
{
	ReadOnlySpan<byte> readOnlySpan = new ReadOnlySpan<byte>((void*)(&<PrivateImplementationDetails>.E7FA4689F40C30105EA7D7783F036683B8E200105687796846C09EC688C7C146), 34);
	if (flag.Length != readOnlySpan.Length)
	{
		return false;
	}
	for (int i = 0; i < flag.Length; i++)
	{
		if ((flag[i] ^ 'u') != (char)(*readOnlySpan[i]))
		{
			return false;
		}
	}
	return true;
}
```

`PrivateImplementationDetails` 型にアクセスしても，**C#** 表示では **static field参照** として抽象化されているため，静的文字列が表示されません．
そこで，表示を **IL** に切り替えることで，`.data cil I_00004930` としてバイト配列が表示されました．
復元に必要な情報は入手したので，復元スクリプトを書きます．

`{} -::<PrivateImplementationDetails>`
```csharp
// Token: 0x02000005 RID: 5
.class private auto ansi sealed '<PrivateImplementationDetails>'
	extends [System.Runtime]System.Object
{
	.custom instance void [System.Runtime]System.Runtime.CompilerServices.CompilerGeneratedAttribute::.ctor() = (
		01 00 00 00
	)
	// Nested Types
	// Token: 0x02000006 RID: 6
	.class nested assembly explicit ansi sealed '__StaticArrayInitTypeSize=34'
		extends [System.Runtime]System.ValueType
	{
		.pack 1
		.size 34

	} // end of class __StaticArrayInitTypeSize=34


	// Fields
	// Token: 0x04000005 RID: 5 RVA: 0x00004930 File Offset: 0x00002B30
	.field assembly static initonly valuetype '<PrivateImplementationDetails>'/'__StaticArrayInitTypeSize=34' E7FA4689F40C30105EA7D7783F036683B8E200105687796846C09EC688C7C146 at I_00004930
	.data cil I_00004930 = bytearray (
		34 19 05 14 16 14 0e 18 10 10 07 1e 14 01 2a 12
		14 2a 13 19 14 18 1c 1b 12 1a 54 2a 1a 01 1a 01
		1a 08
	)

} // end of class <PrivateImplementationDetails>
```

### **Flag** の復元

バイト列をPython配列に変換してもらう処理 (0x prefixの付加) は，**gemini** にやってもらいました．
入手したバイトと，`u` を **ascii** 変換したものを **XOR** します．すると **Flag** 文字列が復元できました．

**Flag復元 Pythonスクリプト**
```python
data = bytes([
    0x34, 0x19, 0x05, 0x14, 0x16, 0x14, 0x0e, 0x18,
    0x10, 0x10, 0x07, 0x1e, 0x14, 0x01, 0x2a, 0x12,
    0x14, 0x2a, 0x13, 0x19, 0x14, 0x18, 0x1c, 0x1b,
    0x12, 0x1a, 0x54, 0x2a, 0x1a, 0x01, 0x1a, 0x01,
    0x1a, 0x08
])

flag = ''.join(chr(b ^ ord('u')) for b in data)

print(flag)
```

試しに，`AlpacaForm.exe` に復元した **Flag** を入力しても，正しく **Correct** と表示されました．

```shell
$ python3 exploit.py
{*** REDACTED ***}
```

---
## Post-Mortem & Dead ends

### .NETバイナリにおける `ReadOnlySpan<byte>` の最適化挙動 (**RVA埋め込み**)

C#において，コード内に固定値の初期化データを `ReadOnlySpan<byte>` や `byte[]` で直書きすると，コンパイラはそれをコード領域ではなく，メタデータ内の特殊な静的領域（`.data cil` / `PrivateImplementationDetails`）に配置する最適化 **(RVA埋め込み)**[^2] を行います．
- **C#表示での抽象化**: dnSpyやILSpyのデフォルトのC#ビューでは，コンパイラが自動生成したフィールド名しか見えず，生データが隠されます．
- **解決策**: .NETのRevにおいて **配列の中身がC#コード上で見えない** ケースに遭遇した際は，**IL (Intermediate Language) 表示** に切り替えます．
  または，バイナリの `.text` や `.data` セクションを直接探すのが定石です．

## References

[^1]: [GitHub - dnSpy/dnSpy: .NET debugger and assembly editor · GitHub](https://github.com/dnSpy/dnSpy)

[^2]: [c# - Understanding .NET CIL array initialization code using \<PrivateImplementationDetails\> and hasfieldrva - Stack Overflow](https://stackoverflow.com/questions/54013077/understanding-net-cil-array-initialization-code-using-privateimplementationdet)
