+++
date = "2016-09-24T17:27:31+09:00"
slug = "rust-inside-python-for-using-hyper-http-client"
tags = ["Rust", "Python"]
title = "Rust inside Python for using hyper HTTP client"
description = "RustのFFI(Foreign Function Interface)を用いて、Rustの関数をPythonから利用する。この関数では、Rust向けのHTTPライブラリhyperを利用して、受け取ったURLリストのHTMLを並行処理で取得する。これらのプログラムは、Mac OS X 10.11.6上で、動作を確認している。"
+++

## 概要
- [Rust](https://www.rust-lang.org)の[FFI(Foreign Function Interface)](https://ja.wikipedia.org/wiki/Foreign_function_interface)を用いて、Rustの関数をPythonから利用する。
- この関数では、Rust向けのHTTPライブラリ[hyper](https://github.com/hyperium/hyper)を利用して、受け取ったURLリストのHTMLを並行処理で取得する。
- これらのプログラムは、Mac OS X 10.11.6上で、動作を確認している。

## はじめに
Rust言語は、安全性、速度、並行性を目標に掲げたシステムレベルのプログラミング言語である。
Rustの並行処理では、Python等の言語で見られる[GIL(Global Interpreter Lock)](https://ja.wikipedia.org/wiki/%E3%82%B0%E3%83%AD%E3%83%BC%E3%83%90%E3%83%AB%E3%82%A4%E3%83%B3%E3%82%BF%E3%83%97%E3%83%AA%E3%82%BF%E3%83%AD%E3%83%83%E3%82%AF)の制約がなく、高い並行性を実現している[^1]。
[^1]: [Rust Inside Other Languages](https://rust-lang-ja.github.io/the-rust-programming-language-ja/1.6/book/rust-inside-other-languages.html)
この記事では、Rustで記述した並行処理の関数を、Pythonから実行し、RustのFFIや並行処理についてのサンプルプログラムを示すことを目的としている。
基本的なRustプログラミングとFFIの使用法については、参考サイトを参照されたい[^2][^3]。
尚、紹介するサンプルプログラムでは、基本的にエラー処理を割愛している。

[^2]: [プログラミング言語Rust](https://rust-lang-ja.github.io/the-rust-programming-language-ja/1.6/book/README.html)
[^3]: [The Rust FFI Omnibus](http://jakegoulding.com/rust-ffi-omnibus/)

以下に解説するプログラムはGithub上に公開している[^4]。
[^4]: [sample-hyper-thread](https://github.com/mayoyamasaki/sample-hyper-thread)

## 環境と依存
この記事では、Rust 1.10とPython 3.5、および、以下のクレートを使用している。
尚、OS X上でhyperクレートを使用する場合は、hyperがrust-opensslを使用しているため、opensslへの環境変数の設定が必要[^5]。

[^5]: [rust-openssl](https://github.com/sfackler/rust-openssl)

パッケージ名  | Version | 用途
------------- | ------------- | -------------
hyper | 0.9.10 | HTTPクライアント
libc | 0.2.16 | FFI
rustc-serialize | 0.3 | jsonエンコード

## FFI、あるいはPythonからのRustの利用
まず初めに、Rust側の関数は以下のように定義した。
```rust
// src/lib.rs
pub extern fn get_htmls_from(urls:*const *const c_char, lenght: size_t) -> *mut c_char {
 ...
}
```
引数は、Pythonにおける以下のような文字列のリストで、Rust側ではc言語の文字型```c_char```のポインタのポインタとして定義している。
```python
# src/main.py
urls = [
    b"http://example.com/",
    b"https://www.yahoo.com/",
]
```
Rust側で受け取った```*const *const c_char```は、次の手続きで、```String```に変換できる。

```rust
// src/lib.rs
    let urls = unsafe {
        slice::from_raw_parts(urls, lenght as usize)
    };
    let urls: Vec<String> = urls.iter()
        .map(|&p| unsafe { CStr::from_ptr(p) })         // make iter of &CStr
        .map(|cs| cs.to_bytes())                        // make iter of &[u8]
        .map(|bs| str::from_utf8(bs).unwrap_or(""))     // make iter of &str
        .map(|s| s.to_string())                         // make iter of String
        .collect();
```

FFIでは引数や戻り値の構造が複雑になるに従い、その扱いが難しくなる。そのため、このサンプルプログラムでは、戻り値はjson形式の文字列として定義している。戻り値のjsonは、keyがURL、valueがHTMLのHashMapを単にjson形式に変換したものである。
```rust
// src/lib.rs
    let mut url2bodies  = HashMap::new();
    ...
    let result = json::encode(&url2bodies).unwrap();
    CString::new(result).unwrap().into_raw()    // return value
```
このように定義したRustの関数は、Cargo.tomlにlibの設定を記述し、```cargo build --release```することで、```target/release/```以下に、実行環境に合わせて、DLL(Dynamic Link Library)が生成される。

Mac OS X上では、dylibが生成され、以下のようなプログラムで、DLLから関数の呼び出しを行う。PythonからRustで生成したdylibを扱うには、C言語の場合と同様で、```ctypes```ライブラリを用いる[^6]。
[^6]: [16.16. ctypes — Pythonのための外部関数ライブラリ](http://docs.python.jp/3.5/library/ctypes.html)

```python
# src/main.py
def get_htmls_as_dict_from(urls):
    """ get html content using rust's dylib concurrently """
    DYLIB_PATH = "target/release/libhyper_thread.dylib"
    lib = ctypes.cdll.LoadLibrary(DYLIB_PATH)
    C_CHAR_P_P = ctypes.c_char_p * len(urls)
    c_urls = C_CHAR_P_P(*urls)
    lib.get_htmls_from.argtypes = (C_CHAR_P_P, ctypes.c_size_t)
    lib.get_htmls_from.restype = ctypes.c_void_p
    htmls = lib.get_htmls_from(c_urls, len(urls))
    try:
        return json.loads(ctypes.cast(htmls, ctypes.c_char_p).value.decode('utf-8'))
    except:
        return None
```
## hyperと並行処理
Rustの並行処理ではチャネルを用いて、URLから取得したHTMLをsendしている。hyperによるGETリクエストでは、あらかじめURL structに変換してから、クエリをsendし、戻り値のレスポンスをStringに変換している。

```rust
// src/lib.rs
    let (tx, rx) = mpsc::channel();
    for url in urls.iter() {
        let url = url.clone();
        let tx = tx.clone();
        thread::spawn(move || {
            let client = Client::new();
            let mut html = String::new();
            match Url::parse(&url) {
                Result::Err(_) => {},
                Result::Ok(hyper_url) => {
                    match client.get(hyper_url).send() {
                        Result::Err(_) => {},
                        Result::Ok(mut res) => {
                            match res.read_to_string(&mut html) {
                                _ => {}
                            }
                        }
                    }
                }
            };
            tx.send((url, html)).unwrap();
        });
    }
```
最後に、全てのrxで受け取った値をHashMapにinsertする。尚、recvする順序は不定のためkeyとなるURLもsendしている。
```rust
// src/lib.rs
    for _ in urls.iter() {
        match rx.recv() {
            Result::Ok((url, html)) => { url2bodies.insert(url, html); },
            Result::Err(_) => {}
        }
    }
```
## おわりに
この記事では、PythonからRustの並行処理を利用するためのサンプルプログラムを紹介し、題材としてhyperのHTTPクライアントを扱った。
PythonからRustで生成したDLLを使用する際は、それぞれctypesとlibc/ffiを使用するため、C言語での値の受渡しを考える必要があり、単にPythonからC言語で生成したDLLを使用するのに比べて関心ごとが増えてしまう。
なので、モダンな言語使用を持つRustが、C言語に比べてある程度有効な場合においては、Python/Cの代わりにPython/Rustを検討してみるのはありかもしれない。
