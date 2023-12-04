- [测试和文档测试](#测试和文档测试)
- [详细解释crate和mod的文档](#详细解释crate和mod的文档)
- [项目和模块](#项目和模块)
  - [cargo](#cargo)
    - [cargo.toml](#cargotoml)
  - [错误处理](#错误处理)
  - [问号运算符](#问号运算符)
  - [和C的ABI兼容](#和c的abi兼容)
    - [从C调用Rust库](#从c调用rust库)
    - [从Rust调用C库](#从rust调用c库)
  - [文档](#文档)

# 测试和文档测试
rust原生支持测试, 还支持对文档中的example代码进行测试.
https://www.cs.brandeis.edu/~cs146a/rust/doc-02-21-2015/book/testing.html

# 详细解释crate和mod的文档
https://www.cs.brandeis.edu/~cs146a/rust/doc-02-21-2015/book/crates-and-modules.html

cargo build会根据约定俗成的规则来编译bin或者lib, 大体上是通过分析目录结构, 和特殊的文件名, 比如`main.rs`, `lib.rs`, `mod.rs`等.

```sh
$ tree .
.
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── english
│   │   ├── farewells.rs
│   │   ├── greetings.rs
│   │   └── mod.rs
│   ├── japanese
│   │   ├── farewells.rs
│   │   ├── greetings.rs
│   │   └── mod.rs
│   └── lib.rs
└── target
    ├── deps
    ├── libphrases-a7448e02a0468eaa.rlib
    └── native
```

# 项目和模块
Rust用了两个概念来管理项目：一个是crate，一个是mod。
* crate简单理解就是一个项目。crate是Rust中的独立编译单元。每个 crate对应生成一个库或者可执行文件（如.lib .dll .so .exe等）。官方有一 个crate仓库https://crates.io/ ，可以供用户发布各种各样的库，用户也可 以直接使用这里面的开源库。
* mod简单理解就是命名空间。mod可以嵌套，还可以控制内部元素的可见性。

crate和mod有一个重要区别是：crate之间不能出现循环引用；而mod是无所谓的，mod1要使用mod2的内容，同时mod2要使用mod1的内容，是完全没问题的。在Rust里面，crate才是一个完整的编译单元 （compile unit）。也就是说，rustc编译器必须把整个crate的内容全部读进去才能执行编译，rustc不是基于单个的.rs文件或者mod来执行编译的。作为对比，C/C++里面的编译单元是单独的.c/.cpp文件以及它们所有的include文件。每个.c/.cpp文件都是单独编译，生成.o文件，再把这些.o文件链接起来。

## cargo
Cargo是官方的项目管理工具
* 新建一个hello world工程, 新建`src/main.rs`  
`cargo new hello_world --bin`
* 如果是已经存在的目录  
`cargo init --bin`
* 新建一个hello world库, 新建`src/lib.rs`  
`cargo new hello_world --lib`
* 编译  
`cargo build --release`

### cargo.toml
举例:
```toml
[package]
name = "seccompiler"
version = "1.1.0"
authors = ["Amazon Firecracker team <firecracker-devel@amazon.com>"]
edition = "2018"
build = "../../build.rs"
description = "Program that compiles multi-threaded seccomp-bpf filters expressed as JSON into raw BPF programs, serializing them and outputting them to a file."
homepage = "https://firecracker-microvm.github.io/"
license = "Apache-2.0"

[[bin]]
name = "seccompiler-bin"
path = "src/seccompiler_bin.rs"

[dependencies]
bincode = "1.2.1"
libc = ">=0.2.39"
serde = { version = ">=1.0.27", features = ["derive"] }
serde_json = ">=1.0.9"

utils = { path = "../utils" }
```
The `Cargo.toml` file for each package is called its _manifest_. It is written in the [TOML](https://toml.io/) format. Every manifest file consists of the following sections:  
参考: https://doc.rust-lang.org/cargo/reference/manifest.html

*   [`cargo-features`](https://doc.rust-lang.org/cargo/reference/unstable.html) — Unstable, nightly-only features.
*   [`[package]`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-package-section) — Defines a package.
    *   [`name`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-name-field) — The name of the package.
    *   [`version`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-version-field) — The version of the package.
    *   [`authors`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-authors-field) — The authors of the package.
    *   [`edition`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-edition-field) — The Rust edition.
    *   [`rust-version`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-rust-version-field) — The minimal supported Rust version.
    *   [`description`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-description-field) — A description of the package.
    *   [`documentation`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-documentation-field) — URL of the package documentation.
    *   [`readme`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-readme-field) — Path to the package's README file.
    *   [`homepage`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-homepage-field) — URL of the package homepage.
    *   [`repository`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-repository-field) — URL of the package source repository.
    *   [`license`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-license-and-license-file-fields) — The package license.
    *   [`license-file`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-license-and-license-file-fields) — Path to the text of the license.
    *   [`keywords`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-keywords-field) — Keywords for the package.
    *   [`categories`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-categories-field) — Categories of the package.
    *   [`workspace`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-workspace-field) — Path to the workspace for the package.
    *   [`build`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-build-field) — Path to the package build script.
    *   [`links`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-links-field) — Name of the native library the package links with.
    *   [`exclude`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-exclude-and-include-fields) — Files to exclude when publishing.
    *   [`include`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-exclude-and-include-fields) — Files to include when publishing.
    *   [`publish`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-publish-field) — Can be used to prevent publishing the package.
    *   [`metadata`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-metadata-table) — Extra settings for external tools.
    *   [`default-run`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-default-run-field) — The default binary to run by [`cargo run`](https://doc.rust-lang.org/cargo/commands/cargo-run.html).
    *   [`autobins`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#target-auto-discovery) — Disables binary auto discovery.
    *   [`autoexamples`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#target-auto-discovery) — Disables example auto discovery.
    *   [`autotests`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#target-auto-discovery) — Disables test auto discovery.
    *   [`autobenches`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#target-auto-discovery) — Disables bench auto discovery.
    *   [`resolver`](https://doc.rust-lang.org/cargo/reference/resolver.html#resolver-versions) — Sets the dependency resolver to use.
*   Target tables: (see [configuration](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#configuring-a-target) for settings)
    *   [`[lib]`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#library) — Library target settings.
    *   [`[[bin]]`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#binaries) — Binary target settings.
    *   [`[[example]]`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#examples) — Example target settings.
    *   [`[[test]]`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#tests) — Test target settings.
    *   [`[[bench]]`](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#benchmarks) — Benchmark target settings.
*   Dependency tables:
    *   [`[dependencies]`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html) — Package library dependencies.
    *   [`[dev-dependencies]`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#development-dependencies) — Dependencies for examples, tests, and benchmarks.
    *   [`[build-dependencies]`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#build-dependencies) — Dependencies for build scripts.
    *   [`[target]`](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#platform-specific-dependencies) — Platform-specific dependencies.
*   [`[badges]`](https://doc.rust-lang.org/cargo/reference/manifest.html#the-badges-section) — Badges to display on a registry.
*   [`[features]`](https://doc.rust-lang.org/cargo/reference/features.html) — Conditional compilation features.
*   [`[patch]`](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html#the-patch-section) — Override dependencies.
*   [`[replace]`](https://doc.rust-lang.org/cargo/reference/overriding-dependencies.html#the-replace-section) — Override dependencies (deprecated).
*   [`[profile]`](https://doc.rust-lang.org/cargo/reference/profiles.html) — Compiler settings and optimizations.
*   [`[workspace]`](https://doc.rust-lang.org/cargo/reference/workspaces.html) — The workspace definition.

## 错误处理
比如使用Option表示some和none两种可能, 返回Result既有值又有错误
```rust
impl str {
    pub fn find<'a, P: Pattern<'a>>(&'a self, pat: P) -> Option<usize> {}
}

impl File {
    pub fn open<P: AsRef<Path>>(path: P) -> io::Result<File> {}
}
```

比如
```rust
use std::mem::{size_of, size_of_val};
use std::str::FromStr;
use std::string::ParseError;
fn main() {
    let r: Result<String, ParseError> = FromStr::from_str("hello");
    println!("Size of String: {}", size_of::<String>());
    println!("Size of `r`: {}", size_of_val(&r));
}
```

## 问号运算符
问号运算符意思是，如果结果是Err，则提前返回一个Result类型，否则继续执行。
标准库的Option、Result实现了问号需要的trait
```rust
fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, String> {
    let mut file = File::open(file_path).map_err(|e| e.to_string())?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)
        .map_err(|err| err.to_string())?;
    let n = contents
        .trim()
        .parse::<i32>()
        .map_err(|err| err.to_string())?;
    Ok(2 * n)
}
```
进一步简化:
```rust
use std::fs::File;
use std::io::Read;
use std::path::Path;
fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, Box<dyn std::error::Error>> {
    let mut file = File::open(file_path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    let n = contents.trim().parse::<i32>()?;
    Ok(2 * n)
}
fn main() {
    match file_double("foobar") {
        Ok(n) => println!("{}", n),
        Err(err) => println!("Error: {:?}", err),
    }
}
```
跟其他很多运算符一样，问号运算符也对应着标准库中的一个trait `std::ops::Try`
```rust
trait Try {
    type Ok;
    type Error;
    fn into_result(self) -> Result<Self::Ok, Self::Error>;
    fn from_error(v: Self::Error) -> Self;
    fn from_ok(v: Self::Ok) -> Self;
}
```

## 和C的ABI兼容
Rust有一个非常好的特性，就是它支持与C语言的ABI兼容.
所以，我们可以用Rust写一个库，然后直接把它当成C写的库来使用。或者反过来，用C写的库，可以直接在Rust中被调用。而且这个过程是没有额外性能损失的。C语言的ABI是这个世界上最通用的ABI，大部分编程语言都支持与C的ABI兼容。这也意味着，Rust与其他语言之间的交互是没问题的，比如用Rust为Python/Node.js/Ruby写一个模块等。

rust编译选项有`--crate-type [bin|lib|rlib|dylib|cdylib|staticlib|proc-macro]`  
其中，`cdylib`和`staticlib`就是与C的ABI兼容的
Rust 中有泛型，C语言里面没有，所以泛型这种东西是不可能暴露出来给C语言使用的，这就不是C语言的ABI的一部分。只有符合C语言的调用方式的函数，才能作为FFI的接口。这样的函数有以下基本要求： 
* 使用`extern "C"`修饰，在Rust中`extern fn`默认等同于`extern "C" fn`； 
* 使用`#[no_mangle]`修饰函数，避免名字重整； 
* 函数参数、返回值中使用的类型，必须在Rust和C里面具备同样的内存布局。

### 从C调用Rust库
假设我们要在Rust中实现一个把字符串从小写变大写的函数，然后 由C语言调用这个函数:
在Rust侧:
```rust
#[no_mangle]
pub extern "C" fn rust_capitalize(s: *mut c_char) {
    unsafe {
        let mut p = s as *mut u8;
        while *p != 0 {
            let ch = char::from(*p);
            if ch.is_ascii() {
                let upper = ch.to_ascii_uppercase();
                *p = upper as u8;
            }
            p = p.offset(1);
        }
    }
}
```
我们在Rust中实现这个函数，考虑到C语言调用的时候传递的是`char*`类型，所以在Rust中我们对应的参数类型是`*mut std::os:: raw::c_char`。这样两边就对应起来了。

用下面的命令生成一个c兼容的静态库:
`rustc --crate-type=staticlib capitalize.rs`

在C里面调用:
```rust
#include <stdlib.h>
#include <stdio.h>
// declare
extern void rust_capitalize(char *);
int main() {
    char str[] = "hello world";
    rust_capitalize(str);
    printf("%s\n", str);
    return 0;
}
```
用下面的命令链接:
`gcc -o main main.c -L. -l:libcapitalize.a -Wl,--gc-sections -lpthread -ldl`

### 从Rust调用C库
比如c的函数
```c
int add_square(int a, int b)
{
    return a * a + b * b;
}
```
在rust里面调用:
```rust
use std::os::raw::c_int;
#[link(name = "simple_math")]
extern "C" {
    fn add_square(a: c_int, b: c_int) -> c_int;
}
fn main() {
    let r = unsafe { add_square(2, 2) };
    println!("{}", r);
}

//编译:
//rustc -L . call_math.rs
```

## 文档
特殊的文档注释 是`///、//！、/**…*/、/*！…*/`，它们会被视为文档
```rust
mod foo {
    //! 这块文档是给 `foo` 模块做的说明
    /// 这块文档是给函数 `f` 做的说明
    fn f() {
    // 这块注释不是文档的一部分
    }
}
```
文档还支持markdown格式