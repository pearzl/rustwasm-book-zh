*commit/2b36157ed81d2286a15c2d040b2fee736fe5df0d*

# 哪些现成的Crate可以与WebAssembly工作?

最简单的是列出目前*不能*与WebAssembly工作的东西；
避免这些东西的crates往往是可以移植到WebAssembly并且通常恰*好能运行*。
一个好的经验法则是，如果一个库支持嵌入式的或者是`#![no_std]`用法，它可能也支持WebAssembly。

## 一个Crate可能无法与WebAssembly一起使用的内容

### C和系统库依赖

wasm中没有系统库，所以任何试图绑定到一个系统库的crate都不能工作。

使用C库也可能会无法工作，因为wasm没有一个稳定的跨语言通讯ABI，并且跨语言连接对于wasm非常挑剔。
每个人都希望这最终能狗生效，特别是因为`clang`现在默认发布他们的`wasm32`目标，但是故事还不是很完整。

### 文件I/O

WebAssembly不能访问文件系统，所以假设文件系统存在&mdash;并且没有针对wasm的解决方案&mdash;的crate将不能工作。

### 产生线程

[有计划为WebAssembly添加线程][wasm-threading]，但是他还没有完成。
试图在`wasm32-unknown-unknown`目标上启动一个线程会导致错误，这将触发一个wasm陷阱。

[wasm-threading]: https://rustwasm.github.io/2018/10/24/multithreading-rust-and-wasm.html

## 那么哪些通用Crates倾向于使用WebAssembly现成的工作？

### 算法和数据结构

提供特定[算法](https://crates.io/categories/algorithms)实现或者是
[数据结构](https://crates.io/categories/data-structures)的crate，
例如A*图搜索或者伸展树就能于WebAssembly很好的工作。

### `#![no_std]`

[不依赖于标准库的crate](https://crates.io/categories/no-std)能与WebAssembly很好工作。

### 解析器

[机械其](https://crates.io/categories/parser-implementations) &mdash; 
只要他们只是获取输入而不自己执行他们的I/O&mdash;就能与WebAssembly很好的工作。
well with WebAssembly.

### 文本处理

[处理以文字方式表达的复杂人类语言的crate](https://crates.io/categories/text-processing)
能够很好的与WebAssembly工作。

### Rust Patterns

[针对Rust中编程特定情况的共享解决方案](https://crates.io/categories/rust-patterns)能够与WebAssembly很好工作。
