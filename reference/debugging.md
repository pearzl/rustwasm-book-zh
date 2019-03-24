*commit/e1697d9d35d1b4d3b6486952bc98608ff028bbce*

# 调试Rust生成的WebAssembly

这节包含了调试Rust生成的WebAssembly的提示。

## 带调试符号构建

> ⚡在调试的时候，永远确保你的构建带有调试符号！

如果你没有启用调试符号，那么`"name"`自定义部分就不会出现在被调试的二进制文件中，
并且栈回溯将出现像 `wasm-function[42]`这样的函数名，而不是像
`wasm_game_of_life::Universe::live_neighbor_count`这样的Rust函数名。

当使用“debug”构建的时候（即`wasm-pack build --debug`或者`cargo build`）调试符号是默认启用的。

"release"构建中，调试符号默认是没有启用的。为了启用调试符，确保`debug = true`
在你`Cargo.toml`文件中的`[profile.release]`部分：

```toml
[profile.release]
debug = true
```

## 使用`console` API记录日志

日志是我们拥有的用于证明或反驳我们的程序为什么出错的假设最有效的工具之一。
在Web上[`console.log`函数](https://developer.mozilla.org/en-US/docs/Web/API/Console/log)
是记录消息到浏览器的开发者工具控制台的方法。

我们可以使用[the `web-sys` crate][web-sys]访问到`console`函数：

```rust
extern crate web_sys;

web_sys::console::log_1(&"Hello, world!".into());
```

另外，[`console.error`函数](https://developer.mozilla.org/en-US/docs/Web/API/Console/error)
和`console.log`有相同的签名，使得当`console.error`函数被使用的时候，
开发者工具倾向于也捕获并沿着日志消息显示堆栈树。

### 引用

* Using `console.log` with the `web-sys` crate:
  * [`web_sys::console::log` takes an array of values to log](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.log.html)
  * [`web_sys::console::log_1` logs a single value](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.log_1.html)
  * [`web_sys::console::log_2` logs two values](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.log_2.html)
  * Etc...
* Using `console.error` with the `web-sys` crate:
  * [`web_sys::console::error` takes an array of values to log](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.error.html)
  * [`web_sys::console::error_1` logs a single value](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.error_1.html)
  * [`web_sys::console::error_2` logs two values](https://rustwasm.github.io/wasm-bindgen/api/web_sys/console/fn.error_2.html)
  * Etc...
* [The `console` object on MDN](https://developer.mozilla.org/en-US/docs/Web/API/Console)
* [Firefox Developer Tools — Web Console](https://developer.mozilla.org/en-US/docs/Tools/Web_Console)
* [Microsoft Edge Developer Tools — Console](https://docs.microsoft.com/en-us/microsoft-edge/devtools-guide/console)
* [Get Started with the Chrome DevTools Console](https://developers.google.com/web/tools/chrome-devtools/console/get-started)

## 记录错误

[`console_error_panic_hook` crate 通过`console.error`记录不期望的错误到开发者控制台。][panic-hook]
而不是神秘，难以调试的`RuntimeError: unreachable executed`错误消息，
这给你一个Rust格式的错误消息。

你要做的所有就是通过在一个初始化函数或者公用代码路径中调用`console_error_panic_hook::set_once()`来安装钩子。

```rust
#[wasm_bindgen]
pub fn init() {
    console_error_panic_hook::set_once();
}
```

[panic-hook]: https://github.com/rustwasm/console_error_panic_hook

## 使用调试器

很不幸，调试WebAssembly的故事还不成熟。在大多数Unix系统上，
[DWARF][dwarf]被DWARF用于编码调试器需要提供的信息，以便为正在运行的程序提供源级检查。
在Windows上有一种替代格式可以编码类似的信息。目前，没有WebAssembly的等价物。
因此，调试器目前提供有限的实用程序，我们最终逐步执行编译器发出的原始WebAssembly指令，
而不是我们编写的Rust源文本。

> 这里有一个[W3C WebAssembly 小组针对调试的子章节][debugging-subcharter]，所以期待这个故事在未来有所进展吧！

[debugging-subcharter]: https://github.com/WebAssembly/debugging
[dwarf]: http://dwarfstd.org/

尽管如此，调试器仍然可用于检查与WebAssembly交互的JavaScript，以及检查原始状态。

### References

* [Firefox Developer Tools — Debugger](https://developer.mozilla.org/en-US/docs/Tools/Debugger)
* [Microsoft Edge Developer Tools — Debugger](https://docs.microsoft.com/en-us/microsoft-edge/devtools-guide/debugger)
* [Get Started with Debugging JavaScript in Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools/javascript/)

## 首先避免调试WebAssembly的需要

如果该错误特定于与JavaScript或Web API的交互，则使用[`wasm-bindgen-test`.][wbg-test]编写测试。

如果一个bug*没有*调用JavaScript或Web API的交互，那么尝试以普通的Rust`#[test]`函数复制它，
这样您可以在调试时利用操作系统的成熟本机工具。
使用像[`quickcheck`][quickcheck]这样的测试crates和它的测试用例收缩器来机械的减少测试用例。
最终，如果您可以在不需要与JavaScript交互的较小测试用例中隔离它们，您将更容易找到并修复错误。

注意，为了为了本地运行`#[test]`而不产生编译和连接错误，
你需要确保在`Cargo.toml`中，`"rlib"`被包含在`[lib.crate-type]`数组中。

```toml
[lib]
crate-type ["cdylib", "rlib"]
```

[quickcheck]: https://crates.io/crates/quickcheck
[web-sys]: https://rustwasm.github.io/wasm-bindgen/web-sys/index.html
[wbg-test]: https://rustwasm.github.io/wasm-bindgen/wasm-bindgen-test/index.html
