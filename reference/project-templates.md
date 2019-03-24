*commit/179deeeb38966fca2087b222627874e15fdf7759*

# 项目模板

Rust和WebAssembly工作组负责管理和维护各种项目模板，以帮助您启动新项目并开始运行。

## `wasm-pack-template`

[此模板][wasm-pack-template]用于启动与[`wasm-pack`][wasm-pack]一起使用的Rust和WebAssembly项目。

使用`cargo generate`来克隆这个项目模板：

```
cargo install cargo-generate
cargo generate --git https://github.com/rustwasm/wasm-pack-template.git
```

## `create-wasm-app`

[这个模板][create-wasm-app]是给JavaScript项目使用的，
这些项目消费使用[`wasm-pack`][wasm-pack]从Rust创建的npm包。

用`npm init`来使用它：

```
mkdir my-project
cd my-project/
npm init wasm-app
```

这个模板通常与 `wasm-pack-template`一起使用，
其中`wasm-pack-template`项目使用`npm link`本地安装，
并作为`create-wasm-app`项目的依赖项引入。

## `rust-webpack-template`

[这个模板][rust-webpack-template]预先配置了所有样板文件，用于将Rust编译为WebAssembly，
并使用Webpack的[`rust-loader`][rust-loader]将其直接挂接到Webpack构建管道中。

用`npm init`来使用它：

```
mkdir my-project
cd my-project/
npm init rust-webpack
```

[wasm-pack]: https://github.com/rustwasm/wasm-pack
[wasm-pack-template]: https://github.com/rustwasm/wasm-pack-template
[create-wasm-app]: https://github.com/rustwasm/create-wasm-app
[rust-webpack-template]: https://github.com/rustwasm/rust-webpack-template
[rust-loader]: https://github.com/wasm-tool/rust-loader/
