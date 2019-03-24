*commit/ca122c2cfcd50dc529b19d5e58542ddf12670a61*

# 缩减`.wasm`大小

对于`.wasm`通过网络传递给客户端的二进制文件，例如我们的Web生命游戏应用，我们想要随时留意代码大小。
我们的`.wasm`越小，页面加载的也就越快，用于也就越开心。

## 通过构建配置文件，我们的生命游戏`.wasm`二进制文件能有多小？

[花店时间阅读一下我们可以调整的构建配置选项使我们得到更小的`.wasm`。](../reference/code-size.html#optimizing-builds-for-code-size)

使用默认的发布构建配置（不带调试符号），我们的WebAssembly二进制文件是29,410字节：

```
$ wc -c pkg/wasm_game_of_life_bg.wasm
29410 pkg/wasm_game_of_life_bg.wasm
```

在启用LTO，设置`opt-level = "z"`，并运行`wasm-opt -Oz`之后，`.wasm`二进制文件缩减到了17,317字节：

```
$ wc -c pkg/wasm_game_of_life_bg.wasm
17317 pkg/wasm_game_of_life_bg.wasm
```

如果我们使用`gzip`（这几乎是每个HTTP服务都会做的）我们降到了9,045字节！

```
$ gzip -9 < pkg/wasm_game_of_life_bg.wasm | wc -c
9045
```

## Exercises

* 使用[`wasm-snip`工具](../reference/code-size.html#use-the-wasm-snip-tool)
  从我们的生命游戏的`.wasm`二进制文件中移除基础错误函数。这节省了多少字节？

* 使用和不适用[`wee_alloc`作为全局分配器](https://github.com/rustwasm/wee_alloc)
  分别构建我们的生命游戏包。
  我们克隆以开始这个项目的模板`rustwasm/wasm-pack-template`有一个
  叫"wee_alloc"的cargo特性，使你可以通过将它添加到`default`键来启用它，
  这个键位于`wasm-game-of-life/Cargo.toml`文件的`[features]`部分。

  ```toml
  [features]
  default = ["wee_alloc"]
  ```

  使用`wee_alloc`减少了`.wasm`多少大小？

* 我们只初始化只初始化一个`Universe`，所以与其提供一个构造器，
  不如我们可以导出一个操作，它持有单个`static mut`的全局实例。
  这个全局实力也使用前面章节讨论过的双缓冲技术，我们可以使那些缓冲区也是全局`static mut`的。
  这从我们的生命游戏实现中移除了所有的动态分配，使得我们可以让它成为一个不需要包含分配器的`#![no_std]`库。 
  完全移除分配器依赖后`.wasm`减少了多少大小？
