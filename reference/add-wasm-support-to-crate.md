*commit/2b36157ed81d2286a15c2d040b2fee736fe5df0d*

# 如何给一个通用crate添加WebAssembly支持

这节适用于想要对通用crate添加WebAssembly支持的crate作者。

## 可能你的Crate已经支持WebAssembly了！

回顾一下[什么内容会使一个通用crate*不能*移植到WebAssembly](./which-crates-work-with-wasm.html)。
如果你的crate没有任何其他内容， 它可能已经支持WebAssembly了！

你总是可以可以通过运行`cargo build`来检查WebAssembly目标：

```
cargo build --target wasm32-unknown-unknown
```

如果命令失败，那么你的crate现在还不支持WebAssembly。
入宫它没有失败，那么你的crate*可能*支持WebAssembly。 
你可以[为wasm添加测试并在CI中运行这些测试](#maintaining-ongoing-support-for-webassembly)
来100%的确定支持（并继续这么做！）。

## 为WebAssembly添加支持

### 避免直接执行I/O

在Web上，I/O总是异步的，并且没有文件系统。
将I/O移除你的库，让用户执行I/O，然后把输入片段传递给你的库。

例如，重构下面这个代码：

```rust
use std::fs;
use std::path::Path;

pub fn parse_thing(path: &Path) -> Result<MyThing, MyError> {
    let contents = fs::read(path)?;
    // ...
}
```

修改为这样：

```rust
pub fn parse_thing(contents: &[u8]) -> Result<MyThing, MyError> {
    // ...
}
```

### 把`wasm-bindgen`添加为依赖

如果你需要和外面的世界互操作（即你不能让库的消费者为你驱动互操作），
在编译为WebAssembly是你需要把`wasm-bindgen`（以及`js-sys`和`web-sys`，如果你需要他们的话）添加为依赖。

```toml
[target.'cfg(target_arch = "wasm32")'.dependencies]
wasm-bindgen = "0.2"
js-sys = "0.3"
web-sys = "0.3"
```

### 避免同步I/O

如果你必须在你的库中执行I/O，那么它不能是同步的。
Web上只有异步I/O。使用[`futures` crate](https://crates.io/crates/futures) 和
[`wasm-bindgen-futures` crate](https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/)
来管理异步I/O。如果你的库函数是某些future类型`F`上的泛型函数，那么那个future可以通过Web上的`fetch`
或者由操作系统提供的非阻塞I/O来实现。

```rust
pub fn do_stuff<F>(future: F) -> impl Future<Item = MyOtherThing>
where
    F: Future<Item = MyThing>,
{
    // ...
}
```

你也可以定义一个trait并WebAssembly和Web以及本地目标分别实现它：

```rust
trait ReadMyThing {
    type F: Future<Item = MyThing>;
    fn read(&self) -> Self::F;
}

#[cfg(target_arch = "wasm32")]
struct WebReadMyThing {
    // ...
}

#[cfg(target_arch = "wasm32")]
impl ReadMyThing for WebReadMyThing {
    // ...
}

#[cfg(not(target_arch = "wasm32"))]
struct NativeReadMyThing {
    // ...
}

#[cfg(not(target_arch = "wasm32"))]
impl ReadMyThing for NativeReadMyThing {
    // ...
}
```

### 避免生成线程

Wasm还不支持线程（但是[实验性的工作正在进行](https://rustwasm.github.io/2018/10/24/multithreading-rust-and-wasm.html)），在wasm中试图生成线程会导致错误。

你可以基于目标是否为WebAssembly来使用`#[cfg(..)]`启用线程和非线程代码路径：

```rust
#![cfg(target_arch = "wasm32")]
fn do_work() {
    // Do work with only this thread...
}

#![cfg(not(target_arch = "wasm32"))]
fn do_work() {
    use std::thread;

    // Spread work to helper threads....
    thread::spawn(|| {
        // ...
    });
}
```

另一个选项是将线程生成从你的库中移除，并允许用户“引入他们自己的线程”，
类似于移除文件I/O并允许用户引入他们的I/O一样。
这与想用拥有自定义线程池的应用程序工作时有一个副作用。

## 保持对WebAssembly不断的支持

### 在CI中为`wasm32-unknown-unknown`进行构建

通过让CI脚本运行以下命令，确保在定位WebAssembly时编译不会失败：

```
rustup target add wasm32-unknown-unknown
cargo check --target wasm32-unknown-unknown
```

例如，你可以把这个添加到Travis CI的`.travis.yml`配置中：

```yaml

matrix:
  include:
    - language: rust
      rust: stable
      name: "check wasm32 support"
      install: rustup target add wasm32-unknown-unknown
      script: cargo check --target wasm32-unknown-unknown
```

### 在NJode.js和无头浏览器中测试

你可以使用`wasm-bindgen-test`和`wasm-pack test`子命令在Node.js或者一个无头浏览器中运行wasm测试。你甚至可以在你的CI中整合这些测试。

[了解更多关于测试wasm的内容。](https://rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/index.html)
