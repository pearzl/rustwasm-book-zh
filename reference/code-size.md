*commit/ca122c2cfcd50dc529b19d5e58542ddf12670a61*

# 缩减`.wasm`代码大小

本节将教您如何针对较小的代码大小来优化`.wasm`构建，
以及如何识别更改Rust源代码的机会，以便产生较少的`.wasm`代码。

## 为什么关心代码大小？

当通过网络提供一个`.wasm`文件时，文件越小客户端就能越快的下载完它。
`.wasm`下载的越快，网页加载时间也越短，用户也越开心。

不过重要的是要记住，尽管代码大小可能不是您感兴趣的最终所有指标，
但它更像是一些更加模糊且难以衡量的内容，如“首次交互时间”。
虽然代码大小在这个度量中占据了重要因素（如果你没有代码，任何事都无法进行），
但是它并不是唯一因素。

WebAssembly通常为用户提供gzip，因此您需要确保通过网络比较gzip大小的传输时间差异。
同时记住，WebAssembly二进制格式非常适合gzip压缩，通常能得到超过50%的较少。

此外，WebAssembly的二进制格式时为了快速解析和处理而优化的。
浏览器如今有“基线编译器”，他能解析WebAssembly并通过网络传入wasm尽快得发出编译后得代码。
这意味着[如果你正在使用`instantiateStreaming`][hacks]，
当第二个Web请求完成后，WebAssembly某块可能已经准备就绪。
另一方面，JavaScript通常需要更长时间才能解析，但也可以通过JIT编译等方式加快速度。

最后，记住WebAssembly在执行速度上远远胜于JavaScript。
您需要确保测量JavaScript和WebAssembly之间的运行时比较，以考虑代码大小的重要性。

如果你的`.wasm`文件比预期得大，不要马上赶到沮丧！代码大小可能只是端到端过程中得一个因素。
只关注代码大小得JavaScript和WebAssembly之间的比较可谓是丢了西瓜捡芝麻。

[hacks]: https://hacks.mozilla.org/2018/01/making-webassembly-even-faster-firefoxs-new-streaming-and-tiering-compiler/

## 为代码大小优化构建

有很多我们可用的配置选项让`rustc`能够生成更小的`.wasm`二进制文件。
在一些场合下，我们用更长的编译时间换取更小的`.wasm`文件。
在另一些场合下，我们用WebAssembly的运行时速度换取更小的代码大小。
我们应该认识到每个选项的权衡，并且在我们用运行时速度交换代码大小时，
分析并度量以做出这种交换是否值得的明确决定。

### 连接时间优化（LTO）的编译

在`Cargo.toml`中，把`lto = true`添加到`[profile.release]`部分：

```toml
[profile.release]
lto = true
```

这给了LLVM更多内联和修剪函数的机会，它不仅会使得生成的`.wasm`更小，而且也能使运行时更快！
缺点是编译将会花费更长时间。

### 告诉LLVM优化代码大小而不是速度

默认的，LLVM的优化传递被调整为提升速度而不是大小。
我们可以通过修改位于`Cargo.toml`中的`[profile.release]`部分来将目标修改为代码大小：

```toml
[profile.release]
opt-level = 's'
```

或者，更加激进的大小优化，以及潜在的速度损耗：

```toml
[profile.release]
opt-level = 'z'
```

注意，非常奇怪的是`opt-level = "s"`有时得到比`opt-level = "z"`更小的结果。永远要测量！

### 使用`wasm-opt`工具

[Binaryen][]工具包是一个专用于WebAssebmly的编译器工具集合。
它比LLVM的WebAssembly后端做的更多，并使用它的`wasm-opt`工具后置处理一个由LLVM更生的`.wasm`二进制文件，
这通常可以得到额外的15-20%的代码大小缩减。通常这同时也会产生运行时的速度提升。

```bash
# Optimize for size.
wasm-opt -Os -o output.wasm input.wasm

# Optimize aggressively for size.
wasm-opt -Oz -o output.wasm input.wasm

# Optimize for speed.
wasm-opt -O -o output.wasm input.wasm

# Optimize aggressively for speed.
wasm-opt -O3 -o output.wasm input.wasm
```

[Binaryen]: https://github.com/WebAssembly/binaryen

### 留意调试信息

wasm二进制大小的最大贡献者之一可以是调试信息和wasm二进制文件的`name`部分。
然而`wasm-pack`工具默认的移除调试信息。
此外`wasm-opt`默认移除`names`部分，除非指定了`-g`参数。

这意味着如果你跟随上面的步骤，默认的在你的wasm二进制文件中没有调试信息和名字部分。
不过，如果你手动将调试信息保留在二进制文件中，请确保这一点。

## 大小分析

如果调整配置选项进行优化得到的`.wasm`二进制文件不够小，是时候做一些分析看看剩余的代码小大都是从哪来的了。 

> ⚡ 就像我们让时间分析知道我们的提速工作一样，我们想让大小分析知道我们的代码大小缩减工作。
> 不这样做那么你只是在浪费你的时间！

### `twiggy`代码大小分析器

[`twiggy`是一个代码大小分析器][twiggy]，它支持WebAssembly作为输出。
它分析一个二进制文件的调用图来解决这些问题：

* 为什么这个函数被包含在二进制文件的第一个位置？

* 这个函数的*保留大小*是什么？即，如果我移除它和其他所以移除它以后将称为无效代码的函数后，能够节省多少空间？

<style>
/* For whatever reason, the default mdbook fonts fonts break with the
   following box-drawing characters, hence the manual style. */
pre, code {
  font-family: "SFMono-Regular",Consolas,"Liberation Mono",Menlo,Courier,monospace;
}
</style>

```text
$ twiggy top -n 20 pkg/wasm_game_of_life_bg.wasm
 Shallow Bytes │ Shallow % │ Item
───────────────┼───────────┼────────────────────────────────────────────────────────────────────────────────────────
          9158 ┊    19.65% ┊ "function names" subsection
          3251 ┊     6.98% ┊ dlmalloc::dlmalloc::Dlmalloc::malloc::h632d10c184fef6e8
          2510 ┊     5.39% ┊ <str as core::fmt::Debug>::fmt::he0d87479d1c208ea
          1737 ┊     3.73% ┊ data[0]
          1574 ┊     3.38% ┊ data[3]
          1524 ┊     3.27% ┊ core::fmt::Formatter::pad::h6825605b326ea2c5
          1413 ┊     3.03% ┊ std::panicking::rust_panic_with_hook::h1d3660f2e339513d
          1200 ┊     2.57% ┊ core::fmt::Formatter::pad_integral::h06996c5859a57ced
          1131 ┊     2.43% ┊ core::str::slice_error_fail::h6da90c14857ae01b
          1051 ┊     2.26% ┊ core::fmt::write::h03ff8c7a2f3a9605
           931 ┊     2.00% ┊ data[4]
           864 ┊     1.85% ┊ dlmalloc::dlmalloc::Dlmalloc::free::h27b781e3b06bdb05
           841 ┊     1.80% ┊ <char as core::fmt::Debug>::fmt::h07742d9f4a8c56f2
           813 ┊     1.74% ┊ __rust_realloc
           708 ┊     1.52% ┊ core::slice::memchr::memchr::h6243a1b2885fdb85
           678 ┊     1.45% ┊ <core::fmt::builders::PadAdapter<'a> as core::fmt::Write>::write_str::h96b72fb7457d3062
           631 ┊     1.35% ┊ universe_tick
           631 ┊     1.35% ┊ dlmalloc::dlmalloc::Dlmalloc::dispose_chunk::hae6c5c8634e575b8
           514 ┊     1.10% ┊ std::panicking::default_hook::{{closure}}::hfae0c204085471d5
           503 ┊     1.08% ┊ <&'a T as core::fmt::Debug>::fmt::hba207e4f7abaece6
```

[twiggy]: https://github.com/rustwasm/twiggy

### 手动检查LLVM-IR

LLVM-IR是在编译器工具链中LLVM生成WebAssembly之前的最终中间表示。
因此，它与最终生成的WebAssembly非常相似。
LLVM-IR越大通常意味着`.wasm`也越大秒如果一个函数占据了LLVM-IR的25%，那它通常也将占据`.wasm`的25%。
虽然这个数字只是一个大概，但是LLVM有中不存在的关键信息（因为WebAssembly缺乏像DWARF那样的调试格式），
这些关键信息表明哪些子程序被内联到给定的函数中。

你可以使用这个`cargo`命令来生成LLVM-IR：

```
cargo rustc --release -- --emit llvm-ir
```

然后你可以在`cargo`的`target`目录中使用`find`命令来定位包含LLVM-IR的`.ll`文件：

```
find target/release -type f -name '*.ll'
```

#### 参考

* [LLVM Language Reference Manual](https://llvm.org/docs/LangRef.html)

## 更激进的工具和技术

调整构建配置使`.wasm`二进制文件变得更小非常有用。
然而，当你需要更进一步的时候，你需要准备使用一些更激进的技术了，像是重写源代码以避免臃肿。
接下来的是一些你必须亲自动手的可以用于使代码更小的技术的集合。

### 避免字符串格式化

`format!`，`to_string`等会引入大量的臃肿代码。
如果可能的话，旨在调试模式进行字符串格式化，在发布模式下使用静态字符串。

### 避免错误

这说起来容易做起来难，不过像`twiggy`这样的工具可以手动检查LLVM以帮助你弄清楚哪个函数会出错。

错误并不总是表现为`panic!()`宏调用。
他们从很多结构体中隐式的出现，例如：

* 超出slice索引边界的索引： `my_slice[i]`

* 除数为0的除法：`dividend / divisor`

* 展开`Option`或者`Result`: `opt.unwrap()` or `res.unwrap()`

前两种情况可以被翻译为第三种情况。
索引可以用易错的`my_slice.get(i)`操作替换。
除法可以使用`checked_div`调用替换。
现在我们只剩下一种情况需要对付了。/

展开一个`Option`或`Result`而不产生错误有两种方法：安全的和不安全的。

安全的方法是在遇到`None`或者`Error`时`abort`而不是报错：

```rust
#[inline]
pub fn unwrap_abort<T>(o: Option<T>) -> T {
    use std::process;
    match o {
        Some(t) => t,
        None => process::abort(),
    }
}
```

最终，错误转换为了`wasm32-unknown-unknown`上的终止，这给了你相同的行为但是没有代码膨胀。

另外，[`unreachable` crate][unreachable]为`Option`和`Result`提供了
一个不安全的[`unchecked_unwrap`扩展方法][unchecked_unwrap]，
这个方法告诉Rust的编译器`Option`是`Some`，`Result`是`Ok`。
如果假设没有成立，这将是未定义的行为。
只有当你110%的*知道*假设会成立，只是编译器没有足够聪明到看见它时才使用这个不安全方法。
即使你已经这么做到了，你也应该有一个依然进行检查的调试版本的构建配置，只在发布构建中使用不检查操作。

[unreachable]: https://crates.io/crates/unreachable
[unchecked_unwrap]: https://docs.rs/unreachable/1.0.0/unreachable/trait.UncheckedOptionExt.html#tymethod.unchecked_unwrap

### 避免分配或者切换到`wee_alloc`

Rust的默认分配器时一个到Rust的`dlmalloc`端口。它大约有10千字节左右。
如果你可以完全避免动态分配，那么你应该能够丢掉这10千字节。

完全避免分配非常困难。
但是从热代码路径中移除分配通常很容易（并且通常也帮助那些热代码路径更快）。
在这些场景中，[使用`wee_alloc`][wee_alloc]替换默认的全局分配器应该能为你节省10千字节中的大部分（但不是全部）。
`wee_alloc`是为这样一种场景设计的一个分配器，即你需要一个分配器，
这个分配器不需要特别快，并且你很愿意用分配速度换取小的代码提价。

[wee_alloc]: https://github.com/rustwasm/wee_alloc

### 使用Trait对象而不是泛型类型参数

当你像这样使用类型参数创建泛型函数的时候：

```rust
fn whatever<T: MyTrait>(t: T) { ... }
```

`rustc`和LLVM会为每一个使用这个函数的类型`T`都创建一个新的这个函数的副本。
这使得编译器有很多机会针对每个特定的类型`T`来进行优化，但是就代码大小而言，这些副本增长的非常快。

如果你像这样使用trait对象代码类型参数：

```rust
fn whatever(t: Box<MyTrait>) { ... }
// or
fn whatever(t: &MyTrait) { ... }
// etc...
```

那么通过虚拟调用，动态调度被使用了，那么只有一个版本的函数被放到`.wasm`中。
缺点是编译器优化机会的丧失以及间接，动态调度的函数调用的额外成本。

### 使用`wasm-snip`工具

[`wasm-snip`使用`unreachable`指令替换WebAssembly函数体。][snip]
如果你仔细看的话，这实际上是一个看起来很小但实际很重要的功能。

可能你知道一些函数可能永远不会再运行时被调用，但是编译器不能在编译时证明？
砍掉它！然后，带上`--dce`标志再次运行`wasm-opt`，
所有被砍掉函数调用的函数也都会被移除（他们在运行时也将不会被调用）。

这个工具在移除错误基础设施时特别有用，因为错误最终会转换为陷阱。

[snip]: https://github.com/fitzgen/wasm-snip
