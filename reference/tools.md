*commit/51062afdd6cc575e0517598fbbc0cf140d060c48*

# 你应该知道的Tools

这是一个用于开发Rust和WebAssembly时你应该知道的精选列表。

## 开发构建和工作流编排

### `wasm-pack` | [repository](https://github.com/rustwasm/wasm-pack)

`wasm-pack` 寻求成为构建和使用Rust生成的WebAssembly的一站式商店，
您希望通过Web或Node.js与JavaScript进行互操作。
`wasm-pack`帮助您构建和生成Rust生成的WebAssembly到npm注册表，
以便与您已经使用的工作流中的任何其他JavaScript包一起使用。

## 优化和操作`.wasm`二进制文件

### `wasm-opt` | [repository](https://github.com/WebAssembly/binaryen)

`wasm-opt`工具读取WebAssembly作为输入，执行转换，优化，
或者检测传递，然后发送转换后的WebAssembly作为输出。
通过在由LLVM生成的`.wasm`二进制文件上运行它，
`rustc`通常将会产生更小执行更快速的二进制`.wasm`。
这个工具是`binaryen`项目的一部分。

### `wasm2js` | [repository](https://github.com/WebAssembly/binaryen)

`wasm2js`工具把WebAssembly编译为"almost asm.js"。
这对于支持没有WebAssembly实现的浏览器很好，例如IE11。
这个工具是`binaryen`项目的一部分。

### `wasm-gc` | [repository](https://github.com/alexcrichton/wasm-gc)

一个用户垃圾回收WebAssembly模块和移除所有不需要的导入导出函数等的小工具。
这对于WebAssembly是一个有效的`--gc-sections`连接器标识。

通常你不需要使用这个工具，因为这两个原因：

1. `rustc` 现在又一个足够新版本的`lld`能够对WebAssembly提供`--gc-sections`标识的支持。 
    这对于LTO构建是自动启用的。
2. `wasm-bindgen` CLI工具会自动为你运行`wasm-gc`。

### `wasm-snip` | [repository](https://github.com/rustwasm/wasm-snip)

`wasm-snip`使用一个`unreachable`指令替换所有WebAssembly函数体。

可能你知道有些函数将永远不会在运行时被调用，但是编译器在编译时不能证明这个？
去掉它！然后再次运行`wasm-gc`并且所有它传递调用的函数（这也可能在运行时从不被调用）也将被移除。

这对于在非调试的生产构建中强行移除Rust的错误基础设施很有用。

## 检查`.wasm`二进制文件

### `twiggy` | [repository](https://github.com/rustwasm/twiggy)

`twiggy`是一个`.wasm`二进制文件的代码大小分析器。它分析一个二进制文件的调用图来解决这样的问题：

* 为什么二进制文件中的这个函数在第一个位置？即，那个被导出的函数被传递嗲用它？
* 这个函数的保留尺寸是多少？即，如果我移除它和移除它之后将变成无效代码的所有函数后能减少多少空间。

使用`twiggy`来给你的二进制文件瘦身！

### `wasm-objdump` | [repository](https://github.com/WebAssembly/wabt)

打印出一个`.wasm`二进制文件的和它的么欸个部分的低等级详细详细。
也支持拆解为WAT文本模式。它有点像objdump，不过是针对WebAssembly的。
这是WABT项目的一部分。

### `wasm-nm` | [repository](https://github.com/fitzgen/wasm-nm)

列出定义在`.wasm`二进制文件中的导入，导出和私有函数符号。有点像nm，不过是针对WebAssembly的。
