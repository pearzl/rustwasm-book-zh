*commit/e1697d9d35d1b4d3b6486952bc98608ff028bbce*

# 调试

在我们开始进一步编写代码之前，我们希望在程序出错时有一些可以使用的调试工具。
花点时间阅读一下[参考页面列出的可用于调试Rust生成的Assembly代码的工具和方法][reference-debugging].

[reference-debugging]: ../reference/debugging.html

## 启用错误日志

[如果我们的代码出错，我们想要在开发者控制台显示信息性的错误消息。]
(../reference/debugging.html#logging-panics)

我的的`wasm-pack-template`有一个配置在`wasm-game-of-life/src/utils.rs`中的
默认启动的可选配置项[the `console_error_panic_hook` crate][panic-hook]。 
我们需要做的就是在初始化函数或公用代码路径中安装钩子。
我们可以在`wasm-game-of-life/src/lib.rs`的`Universe::new`构造器中调用它：

```rust
pub fn new() -> Universe {
    utils::set_panic_hook();

    // ...
}
```

[panic-hook]: https://github.com/rustwasm/console_error_panic_hook

## 给我们的生命游戏添加日志

我们[通过`web-sys` crate使用`console.log`函数来在我们的`Universe::tick`中添加一些每个细胞的日志][logging]。

首先。在`wasm-game-of-life/Cargo.toml`中添加`web-sys`为依赖并启用它的`"console"`特性：

```toml
[dependencies.web-sys]
version = "0.3"
features = [
  "console",
]
```

为了好用些，我们我们把`console.log`封装为一个`println!`风格的宏： 

[logging]: ../reference/debugging.html#logging-with-the-console-apis

```rust
extern crate web_sys;

// 一个提供了`println!(..)`风雨语法的`console.log`日志宏：
macro_rules! log {
    ( $( $t:tt )* ) => {
        web_sys::console::log_1(&format!( $( $t )* ).into());
    }
}
```

现在，我们在Rust代码中插入调用`log`来在控制台记录消息了。
例如，为了记录每个细胞的状态，邻居数，以及下一个状态，我们可以像这样修改`wasm-game-of-life/src/lib.rs`：

```diff
diff --git a/src/lib.rs b/src/lib.rs
index f757641..a30e107 100755
--- a/src/lib.rs
+++ b/src/lib.rs
@@ -123,6 +122,14 @@ impl Universe {
                 let cell = self.cells[idx];
                 let live_neighbors = self.live_neighbor_count(row, col);

+                log!(
+                    "cell[{}, {}] is initially {:?} and has {} live neighbors",
+                    row,
+                    col,
+                    cell,
+                    live_neighbors
+                );
+
                 let next_cell = match (cell, live_neighbors) {
                     // Rule 1: Any live cell with fewer than two live neighbours
                     // dies, as if caused by underpopulation.
@@ -140,6 +147,8 @@ impl Universe {
                     (otherwise, _) => otherwise,
                 };

+                log!("    it becomes {:?}", next_cell);
+
                 next[idx] = next_cell;
             }
         }
```

## 使用调试器在每次Tick之间暂停

[浏览器的步进调试器用于检查与我们的Rust生成的WebAssembly交互的JavaScript非常有用。](../reference/debugging.html#using-a-debugger)

例如，我们可以通过在我们呢调用`universe.tick()`的上方放置一个
[一个JavaScript `debugger;`语句][dbg-stmt]来使用调试器，使得我们`renderLoop`函数的每次迭代都暂停。

```js
const renderLoop = () => {
  debugger;
  universe.tick();

  drawGrid();
  drawCells();

  requestAnimationFrame(renderLoop);
};
```

这给我们检查日志消息和比较当前渲染的帧和前一帧提供了一个方便的检查点。

[dbg-stmt]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/debugger

[![Screenshot of debugging the Game of Life](../images/game-of-life/debugging.png)](../images/game-of-life/debugging.png)

## 练习

* 给`tick`函数添加日志记录将状态从存活转换为死亡的每个细胞的行和列，反之亦然

* 引入一个`panic!()`到`Universe::new`方法中。在你Web浏览器的JavaScript调试器中观察错误的回溯信息。
  禁用调试符号，不带`console_error_panic_hook`可选依赖重新构建，然后再次观察栈追踪。 
  不是很有用了吗？
