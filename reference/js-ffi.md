*commit/042d7f0406d8c16809a6387fcc1f303c61e49a23*

# JavaScript互操作

## 导入和导出JS函数

### 从Rust这边

当在一个JS宿主中使用wasm时，从Rust这边导入导出函数都很直接：它和C的方式非常相似。

WebAssembly模块声明一格导入序列，每个带有一个*module name*和一个*import name*。
对于一个`extern { ... }`块的模块名可以使用[`#[link(wasm_import_module)]`][wasm_import_module]
来进行制定，当前它默认是“env”。

导出只有一个名字。另外对于任何`extern`函数，WebAssembly示例的默认线性内存被导出为“内存”。 functions the
WebAssembly instance's default linear memory is exported as "memory".

[wasm_import_module]: https://github.com/rust-lang/rust/issues/52090

```rust
// import a JS function called `foo` from the module `mod`
#[link(wasm_import_module = "mod")]
extern { fn foo(); }

// export a Rust function called `bar`
#[no_mangle]
pub extern fn bar() { /* ... */ }
```

因为wasm限制值的类型，这些函数必须只运行在原始数字类型上。

### 从JS一边

在JS中，一个wasm二进制文件转换为一个ES6模块。
他必须被使用线性内存*实例化*并且有一组与预期导入匹配的JS函数。
实例化的细节可在[MDN][instantiation]上查看。

[instantiation]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/instantiateStreaming

生成的ES6模块将包含从Rust导出的所有函数，现在可以作为JS函数使用了。

[这里][hello world]有一个全部设置的简单例子。

[hello world]: https://www.hellorust.com/demos/add/index.html

## 超越数字

当在JS中使用wasm时，在JS内存和wasm模块内存之间有一个严重的分离：

- 每个wasm模块有一个线性内存（在本文档的顶部描述过了），他在实例化的时候被初始化。
  **JS代码可以自由的读写这块内存。**

- 相比较，wasm代码则不能直接访问JS对象。

因此，复杂的互操作主要以两种方式发生：

- 将我们的二进制数据拷贝进（出）wasm内存。例如，这是一个提供一个所有权的`String`到rust一侧的方法。

- 设置一个显示的JS的对象的堆，这些对象之后被给出“地址”。
  这允许wasm代码间接地（使用整形）引用JS对象，然后通过调用其他被导入的JS函数来操作那些对象。

幸运的时，这个互操作的事情非常使用通过一个通用的"bindgen"风格的框架[wasm-bindgen]来解决。
这个框架使我们可以编写管用的Rust函数签名并自动映射到管用的JS函数。

[wasm-bindgen]: https://github.com/alexcrichton/wasm-bindgen

## 自定义段

自定义部分允许将命名的任意数据嵌入到wasm模块中。
段数据在编译时设置，直接从wasm模块读取，不能在运行时修改。

在Rust中，自定义段时静态数组（`[T; size]`），使用`#[link_section]`属性导出：

```rust
#[link_section = "hello"]
pub static SECTION: [u8; 24] = *b"This is a custom section";
```

这添加了一个名为`hello`的自定义段到wasm文件，Rust的变量名`SECTION`时随意的，
改变它并不会改变这个行为。在这里内容时文本字节，但是它实际可以是任何数据。

自定义段可以在JS这边通过使用[`WebAssembly.Module.customSections`]函数读取，
它获取以恶wasm模块和段名作为参数，返回一个[`ArrayBuffer`]的数组。
多个段可以使用相同的名字来指定，这时他们将同时出现在这个数组中。

```js
WebAssembly.compileStreaming(fetch("sections.wasm"))
.then(mod => {
  const sections = WebAssembly.Module.customSections(mod, "hello");

  const decoder = new TextDecoder();
  const text = decoder.decode(sections[0]);

  console.log(text); // -> "This is a custom section"
});
```

[`ArrayBuffer`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[`WebAssembly.Module.customSections`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Module/customSections
