*commit/eac83fdf1281a9cee92358942cb61641d48a4564*

# Hello, World!

本节将会为你展示如何构建并运行你的第一个Rust和WebAssembly程序：一个带有警告弹窗“Hello, World!”的网页。

确保在开始之前你已经跟随[搭建说明](setup.html)进行了其中的操作。。

## 克隆项目模板

项目模板带有预先配置的合理默认值，是的你可以快速构建整合和打包你的Web代码。

使用这个命令克隆项目模板：

```text
cargo generate --git https://github.com/rustwasm/wasm-pack-template
```

这应该会提示你使用新的项目名称。
我们将使用**"wasm-game-of-life"**。

```text
wasm-game-of-life
```

## 里面有什么

进入新的`wasm-game-of-life`项目。

```
cd wasm-game-of-life
```

让我们看一看它的内容：

```text
wasm-game-of-life/
├── Cargo.toml
├── LICENSE_APACHE
├── LICENSE_MIT
├── README.md
└── src
    ├── lib.rs
    └── utils.rs
```

让我们看看这几个文件的细节。

### `wasm-game-of-life/Cargo.toml`

`Cargo.toml`文件为Rust的包管理器和构建工具`cargo`指定了依赖和元信息。
它预先配置了一个`wasm-bindgen`依赖，一些可选的依赖我们稍后介绍，并且正确初始化`crate-type`以生成`.wasm`库。

### `wasm-game-of-life/src/lib.rs`

`src/lib.rs`文件是我们将要编译为WebAssembly的Rust包的根。
它使用`wasm-bindgen`与JavaScript进行互操作。
它导入了`window.alert`JavaScript函数，并导出`greet`Rust函数，这提示一个问候消息。

```rust
extern crate cfg_if;
extern crate wasm_bindgen;

mod utils;

use cfg_if::cfg_if;
use wasm_bindgen::prelude::*;

cfg_if! {
    // 当`wee_alloc`特性启用的时候，使用`wee_alloc`作为全局分配器。
    if #[cfg(feature = "wee_alloc")] {
        extern crate wee_alloc;
        #[global_allocator]
        static ALLOC: wee_alloc::WeeAlloc = wee_alloc::WeeAlloc::INIT;
    }
}

#[wasm_bindgen]
extern {
    fn alert(s: &str);
}

#[wasm_bindgen]
pub fn greet() {
    alert("Hello, wasm-game-of-life!");
}

```

### `wasm-game-of-life/src/utils.rs`

`src/utils.rs`模块提供了常用的工具使得将Rust编译为WebAssembly的工作更容易。
在之后的教程中我们将详细了解这些工具，比如当我们在[调试我们的wasm代码一节](debugging.html)，
不过目前我们可以忽略这个文件。

## 构建项目

我们使用`wasm-pack`来编排下面的构建步骤：

* 确保我们已经拥有Rust1.30或者更新的版本，并且`wasm32-unknown-unknown`已经通过`rustup`完成安装，
* 通过`cargo`将我们的Rust源代码编译为WebAssembly的`.wasm`二进制文件，
* 使用`wasm-bindgen`生成JavaScript的API以使用我们的Rust生成的WebAssembly。

为了完成上述所有，在项目目录内运行这个命令：

```
wasm-pack build
```

当构建完成后，我们可以在`pkg`目录找到它的输出，它应该有这些内容。

```
pkg/
├── package.json
├── README.md
├── wasm_game_of_life_bg.wasm
├── wasm_game_of_life.d.ts
└── wasm_game_of_life.js
```

`README.md`文件是从主项目中拷贝的，其他文件则完全是新的。

### `wasm-game-of-life/pkg/wasm_game_of_life_bg.wasm`

`.wasm`文件是从Rust源代码编译而来的WebAssembly二进制文件。
它包含编译到wasm版本的所有Rust函数和数据。
例如，它有一个被导出的“greet”函数。

### `wasm-game-of-life/pkg/wasm_game_of_life.js`

`.js`文件是由`wasm-bindgen`生成的，并且包含JavaScript胶水代码，
胶水代码用于导入DOM和JavaScript函数到Rust并对JavaScript暴露一个很好的WebAssembly函数的API。
例如，有一个JavaScript的`greet`函数，它封装了从WebAssembly模块暴露的`greet`函数。
目前，这个胶水代码还没有做太多，不过当我们开始在wasm和JavaScript之间传递更多有趣的值时，
它将有助于将这些值传递到边界。

```js
import * as wasm from './wasm_game_of_life_bg';

// ...

export function greet() {
    return wasm.greet();
}
```

### `wasm-game-of-life/pkg/wasm_game_of_life.d.ts`

`.d.ts`文件包含JavaScript胶水代码的[TypeScript][]类型申明。
如果你正在使用TypeScript，你已经有调用WebAssembly函数的类型检查了，并且你的IDE可以提供自动完成和建议！
如果你没有使用TypeScript，你可以安全的忽略这个文件。

```typescript
export function greet(): void;
```

[TypeScript]: http://www.typescriptlang.org/

### `wasm-game-of-life/pkg/package.json`

[`package.json`包含了生成的JavaScript和WebAssembly包的元信息。][package.json]
这被npm和JavaScript绑定使用以决定在跨包，包名，版本和许多其他内容时的依赖关系。
它帮助我们与JavaScript工具集成，并允许我们将我们的包发布到npm。

```json
{
  "name": "wasm-game-of-life",
  "collaborators": [
    "Your Name <your.email@example.com>"
  ],
  "description": null,
  "version": "0.1.0",
  "license": null,
  "repository": null,
  "files": [
    "wasm_game_of_life_bg.wasm",
    "wasm_game_of_life.d.ts"
  ],
  "main": "wasm_game_of_life.js",
  "types": "wasm_game_of_life.d.ts"
}
```

[package.json]: https://docs.npmjs.com/files/package.json

## 放到网页上

为了获取`wasm-game-of-life`包并在Web页面上使用它，我们使用
[`create-wasm-app` JavaScript项目模板][create-wasm-app].

[create-wasm-app]: https://github.com/rustwasm/create-wasm-app

在`wasm-game-of-life`目录下运行这个命令：

```
npm init wasm-app www
```

这是我们新的`wasm-game-of-life/www`子目录所包含的：

```
wasm-game-of-life/www/
├── bootstrap.js
├── index.html
├── index.js
├── LICENSE-APACHE
├── LICENSE-MIT
├── package.json
├── README.md
└── webpack.config.js
```

再一次，让我们仔细看看这些文件。

### `wasm-game-of-life/www/package.json`

这个`package.json`预先配置了`webpack`和`webpack-dev-server`依赖，
也依赖于`hello-wasm-pack`，这是一个已经发布在npm上的`wasm-pack-template`包的初始化版本。

### `wasm-game-of-life/www/webpack.config.js`

这个文件配置了webpack和它的本地开发服务。
它是预先配置好的，你根本不需要进行调整就能使webpack和它的本地开发服务工作。

### `wasm-game-of-life/www/index.html`

这是Web页面的根HTML。
它没有做太多仅仅加载`bootstrap.js`，一个很简单的`index.js`封装。

```html
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Hello wasm-pack!</title>
  </head>
  <body>
    <script src="./bootstrap.js"></script>
  </body>
</html>
```

### `wasm-game-of-life/www/index.js`

`index.js`是我们的Web页面的JavaScript的主要入口点。
它导入了`hello-wasm-pack` npm包，
其中包含默认的`wasm-pack-template`的已编译WebAssembly和JavaScript胶水代码，
然后它调用`hello-wasm-pack`的`greet`函数。

```js
import * as wasm from "hello-wasm-pack";

wasm.greet();
```

### 安装依赖

首先，在`wasm-game-of-life/www`子目录内运行 `npm install`以确保本地开发服务和它的依赖已经安装：

```text
npm install
```

这个命令只需要运行一次就能安装`webpack`的JavaScript绑定和它的开发服务。

> 注意，`webpack`不是Rust和WebAssembly所要求的，它只是我们这里为了方便所选择的绑定和开发服务。
> Parcel和Rollup应该也支持导入WebAssembly作为ECMAScript模块。

### 在`www`中使用我们的本地`wasm-game-of-life`包

我们想要使用我们的本地`wasm-game-of-life`包而不是npm上的`hello-wasm-pack`包。
这允许我们增量地开发我们的Game of Life程序。

首先，在`wasm-game-of-life/pkg`目录内运行`npm link`，
使得本地包不用将他们发布到npm就可以被其他本地包依赖：

```bash
npm link
```

> 🐞 你运行`npm link`时遇到`EACCESS`或者权限错误了吗？ 
> [如何阻止`npm`的权限错误。]
> (https://docs.npmjs.com/getting-started/fixing-npm-permissions)

然后，通过在`wasm-game-of-life/www`目录内运行这个命令，
来在`www`包中使用`npm link`过的`wasm-game-of-life`版本： 

```
npm link wasm-game-of-life
```

最后，修改`wasm-game-of-life/www/index.js`来导入`wasm-game-of-life`包而不是`hello-wasm-pack`包：

```js
import * as wasm from "wasm-game-of-life";

wasm.greet();
```

我们的Web页面现在准备好本地服务了。

## 本地服务

接下来，为开发服务打开一个新的终端。
在新终端中运行服务使得我们让它在后台运行，而不会阻碍我们同时运行其他命令。
新终端中，在`wasm-game-of-life/www`目录下运行这个命令：

```
npm run start
```

将你的Web浏览器导航到[http://localhost:8080/](http://localhost:8080/)
你应该能得到一个警示弹窗的问候信息：

[![Screenshot of the "Hello, wasm-game-of-life!" Web page alert](../images/game-of-life/hello-world.png)](../images/game-of-life/hello-world.png)

任何使用你做出了改变并想要反映在[http://localhost:8080/](http://localhost:8080/)，
只需要在`wasm-game-of-life`目录中重新运行`wasm-pack build`

## 练习

* 修改`wasm-game-of-life/src/lib.rs`中的`greet`函数接受一个`name: &str`参数，
  这个参数自定义了弹窗信息，然后从`wasm-game-of-life/www/index.js`中的`greet`函数传递你的名字。
  使用`wasm-pack build`重新构建`.wasm`，然后再你的浏览器中刷新
  [http://localhost:8080/](http://localhost:8080/)
  你应该看到一个自定义的问候消息！

  <details>
    <summary>答案</summary>

    <p class="comments-section"><code>wasm-game-of-life/src/lib.rs</code>中新版本的<code>greet</code>函数：</P>

    <pre><code>
    #[wasm_bindgen]
    pub fn greet(name: &str) {
        alert(&format!("Hello, {}!", name));
    }
    </code></pre>

    <p class="comments-section"><code>wasm-game-of-life/www/index.js</code>中新的<code>greet</code>调用：</p>

    <code>
    wasm.greet("Your Name");
    </code>

  </details>
