---
title: Daily-AlpacaHack 「C# File-based app」Easy Writeup
description: .NET 10におけるBase64とBrotliで埋め込まれたフラグを復元する問題
date: 2026-10-03 00:10:00 +0900
categories: [CyberSecurity, CTF]
tags: [alpacahack, rev, dotnet]
---

# daily_alpaca-rev-easy-csharp_file_based_app

## Summary

本問は，.NET 10におけるBase64とBrotliで埋め込まれたフラグを復元する問題です．

> - **Category**: Rev
> - **Description**: .NET 10で追加されたFile-based apps機能のおかげで、.csファイルそのものを実行できるようになりました！
> - **Tools & TechStack**:
> 	- .NET 10
> - **Release**: 2026/10/02
{: .prompt-info }

**階層構造**
```
.
├── chal.cs
└── run-using-docker.sh

1 directory, 2 files
```

---
## ソースコードの解析

Base64文字列をデコードし，そのバイト列を **Brotli展開**[^1] した結果と入力文字列 `input` を比較しているフラグチェッカーのようです．

- `temp1`: Base64文字列をデコードした結果
- `temp2`: `temp1` を Brotli 展開した結果 (正解のフラグ)
- `temp3`: 入力文字列 `input` をUTF-8のバイト列に変換した結果

`chal.cs`
```csharp
#!/usr/bin/env dotnet
// You can install .NET 10 SDK by following https://learn.microsoft.com/en-us/dotnet/core/install/linux
// or you can execute this file using `run-using-docker.sh`.

using System.Buffers.Text;
using System.IO.Compression;
using System.Text;

Console.Write("flag> ");
Console.Out.Flush();
string input = Console.ReadLine()?.Trim() ?? "";
Console.WriteLine(IsCorrect() ? $"Correct! The flag is {input}" : "Incorrect...");

bool IsCorrect()
{
    ReadOnlySpan<byte> encoded = "GzMA+MVtbG/XtFYgGNGLSpSiwoAdaiYvxraqBw=="u8;
    Span<byte> temp1 = stackalloc byte[256];
    Base64.DecodeFromUtf8(encoded, temp1, out _, out int bytesWritten1);

    Span<byte> temp2 = stackalloc byte[256];
    using var decoder = new BrotliDecoder();
    decoder.Decompress(temp1[..bytesWritten1], temp2, out _, out int bytesWritten2);

    Span<byte> temp3 = stackalloc byte[Encoding.UTF8.GetMaxByteCount(input.Length)];
    int bytesWritten3 = Encoding.UTF8.GetBytes(input, temp3);
    return temp2[..bytesWritten2].SequenceEqual(temp3[..bytesWritten3]);
}
```

> .NETプログラムにもかかわらず，`.cs` ファイルしか配布されていないのは，.NET 10 で導入された **file-based apps** という仕組みにより，`.csproj` 無しで単独実行が可能になったためです．
{: .prompt-info }

## 解法

Pythonで **Brotli** を扱えないかを調べてみると，`brotli` モジュールが見つかりました．
ハードコードされているBase64文字列をデコードし，`brotli` モジュールの `decompress` で展開します．

```shell
$ nix-shell -p 'python3.withPackages (ps: [ ps.brotli ])' --run "python3 -c \"
import base64, brotli
print(brotli.decompress(base64.b64decode('GzMA+MVtbG/XtFYgGNGLSpSiwoAdaiYvxraqBw==')))
\""
b'<REDACTED>'
```

```shell
$ ./run-using-docker.sh 
flag> <REDACTED>
Correct! The flag is <REDACTED>
```

---
## Post-Mortem & Dead ends

N/A

## References

[^1]: Googleが開発した汎用かつ高性能な可逆圧縮アルゴリズム
	[GitHub - google/brotli: Brotli compression format · GitHub](https://github.com/google/brotli)
