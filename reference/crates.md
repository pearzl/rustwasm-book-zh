*commit/cde26ad18895984504206b397f462476e95f2998*

# 你应该知道的Crates

这个一个精选的用于Rust和WebAssembly开发的crates的列表。

[你也可以浏览发布在crates.io的所有在WebAssembly分类下的crates。][wasm-category]

## 与JavaScript和DOM互操作

### `wasm-bindgen` | [crates.io](https://crates.io/crates/wasm-bindgen) | [repository](https://github.com/rustwasm/wasm-bindgen)

`wasm-bindgen`实现了Rust和JavaScript之间的高级别互操作。
它允许将JavaScript的内容导入到Rust和将Rust内容导出给JavaScript。

### `wasm-bindgen-futures` | [crates.io](https://crates.io/crates/wasm-bindgen-futures) | [repository](https://github.com/rustwasm/wasm-bindgen/tree/master/crates/futures)

`wasm-bindgen-futures` 是一个连接JavaSript的`Promise`和Rust的`Future`的桥梁。
它可以在两个方向执行转换并且当在Rust中执行异步任务的时候很有用，同时允许与DOM时间和I/O操作互操作。

### `js-sys` | [crates.io](https://crates.io/crates/js-sys) | [repository](https://github.com/rustwasm/wasm-bindgen/tree/master/crates/js-sys)

原生`wasm-bindgen`导入了所有JavaScript全局类型和方法，如`Object`，`Function`，`eval`等。
这些API可以在所有标准ECMAScript环境中移植，而不仅仅是Web，例如Node.js。

### `web-sys` | [crates.io](https://crates.io/crates/web-sys) | [repository](https://github.com/rustwasm/wasm-bindgen/tree/master/crates/web-sys)

原生的`wasm-bindgen`导入了所有的Web API，比如DOM操作，`setTimeout`，Web GL， Web Audio等。

## 错误报告和日志

### `console_error_panic_hook` | [crates.io](https://crates.io/crates/console_error_panic_hook) | [repository](https://github.com/rustwasm/console_error_panic_hook)

这个crate允许你通过提供一个转发错误消息到`console.error`的钩子来在`wasm32-unknown-unknown`上进行调试。

### `console_log` | [crates.io](https://crates.io/crates/console_log) | [repository](https://github.com/iamcodemaker/console_log)

这个crate为[`log` crate](https://crates.io/crates/log)提供了一个后端，它能把日志消息路由到开发工具控制台。

## Dynamic Allocation

### `wee_alloc` | [crates.io](https://crates.io/crates/wee_alloc) | [repository](https://github.com/rustwasm/wee_alloc)

**W**asm-**E**nabled, **E**lfin 分配器。
一个小的（~1K未压缩的`.wasm`）分配器实现，用在代码尺寸比分配性能更受关注的时候。 

## 解析和生成`.wasm`二进制文件

### `parity-wasm` | [crates.io](https://crates.io/crates/parity-wasm) | [repository](https://github.com/paritytech/parity-wasm)

用于序列化，反序列化和构建`.wasm`二进制文件的低级WebAssembly格式库。

### `wasmparser` | [crates.io](https://crates.io/crates/wasmparser) | [repository](https://github.com/yurydelendik/wasmparser.rs)

一个简单的事件驱动库，用于解析WebAssembly二进制文件。
提供每个已解析事物的字节偏移量，例如，在解释reloc时这是必需的。

## 翻译和编译WebAssembly

### `wasmi` | [crates.io](https://crates.io/crates/wasmi) | [repository](https://github.com/paritytech/wasmi)

来自Parity的可嵌入WebAssembly解释器。

### `cranelift-wasm` | [crates.io](https://crates.io/crates/cranelift-wasm) | [repository](https://github.com/CraneStation/cranelift)

将WebAssembly编译为本机主机的机器代码。 Cranelift（néCretonne）代码生成器项目的一部分。

[wasm-category]: https://crates.io/categories/wasm
