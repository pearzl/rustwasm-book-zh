*commit/5a78282a681c8134cdb1c12e7a86868febf10d19*

# 搭建

本节介绍如何设置工具链以将Rust程序编译为WebAssembly并将它们集成到JavaScript中。

## Rust工具链

你将会用到标准Rust工具链，包括`rustup`，`rustc`，和`cargo`。

[跟随这些说明来安装Rust工具链][rust-install]

Rust和WebAssembly的经验是在Rust发布版上稳定运行！
这意味着我们不要求任何实验性的特性标志。
不过，我们要求Rust1.30或者更新的版本。

## `wasm-pack`

`wasm-pack` wasm-pack是您构建，测试和发布Rust生成的WebAssembly的一站式商店。 

[在这里获取wasm-pack！][wasm-pack-install]

## `cargo-generate`

[`cargo-generate` 帮助你借由一个已经存在的git仓库作为模板来快速的启动并运行一个新的Rust项目。][cargo-generate]

使用这个命令安装`cargo-generate`：

```
cargo install cargo-generate
```

## `npm`

`npm` 是一个JavaScript的包管理工具。
我们将会使用它来安装和运行JavaScript的绑定和开发服务。
在教程的最后，我们将会发布我们编译的`.wasm`到`npm`注册处。

[跟随这些说明来安装`npm`。][npm-install]

如果你已经安装过`npm`，使用这个命令确保它更新过。

```
npm install npm@latest -g
```

[rust-install]: https://www.rust-lang.org/tools/install
[npm-install]: https://www.npmjs.com/get-npm
[wasm-pack]: https://github.com/rustwasm/wasm-pack
[cargo-generate]: https://github.com/ashleygwilliams/cargo-generate
[wasm-pack-install]: https://rustwasm.github.io/wasm-pack/installer/
