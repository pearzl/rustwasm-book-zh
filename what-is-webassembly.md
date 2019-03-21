*commit/089aa91cd71c127854898b794664b2ba7113a4f4*

# 什么是 WebAssembly?

WebAssembly (wasm) 是一种具有[广泛规范][extensive specification]的机器模型和可执行格式。
它被设计为轻便，紧凑并且以原生速度或接近原生速度运行。

作为一种程序语言，WebAssembly由两种表示相同结构的格式构成，尽管以不同的方式：

1. `.wat` 文本格式 (称为 `wat` 是 "**W**eb**A**ssembly **T**ext") 使用[符号表达式][S-expressions],
   并且与Lisp族的语言有一些相似，如Scheme 和 Clojure。

2. `.wasm` 二进制格式更低等级并且意图呗wasm虚拟机直接使用
   这在概念上很像ELF和Mach-O。

作为参考，这是一个 `wat`形式的阶乘函数：

```
(module
  (func $fac (param f64) (result f64)
    get_local 0
    f64.const 1
    f64.lt
    if (result f64)
      f64.const 1
    else
      get_local 0
      get_local 0
      f64.const 1
      f64.sub
      call $fac
      f64.mul
    end)
  (export "fac" (func $fac)))
```

如果你好奇一个 `wasm`看起来是怎样的你可以使用上面代码的[wat到wasm示例][wat2wasm demo]

## 线性内存

WebAssembly 有一个非常简单的[内存模型][memory model]。 
一个wasm模块访问单个“线性数组”，它本质上是一个平面的字节数组。
这个内存可以按照页大小（64K）成倍的增长[memory can be grown]。
它不能被缩小。

## WebAssembly只针对Web吗?

虽然它目前一般在JavaScript和Web社区获得关注，但是wasm并没有假定它的宿主环境。
因此可以推测wasm未来将成为一个用于多种环境下的“便携的可执行”格式。
不过截至目前，与wasm最相关的是JavaScript（JS），他是有很多形式(包括在Web上面的和[Node.js]).

[memory model]: https://webassembly.github.io/spec/core/syntax/modules.html#syntax-mem
[memory can be grown]: https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-instr-memory
[extensive specification]: https://webassembly.github.io/spec/
[value types]: https://webassembly.github.io/spec/core/syntax/types.html#value-types
[Node.js]: https://nodejs.org
[S-expressions]: https://en.wikipedia.org/wiki/S-expression
[wat2wasm demo]: https://cdn.rawgit.com/WebAssembly/wabt/aae5a4b7/demo/wat2wasm/
