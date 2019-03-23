*commit/a3f3f4b36a34b12580015063bd548eda03577dd5*

# 实现康威生命游戏

## 设计

在我们深入之前，我们有一些设计选择供考虑。

### 无限宇宙

生命游戏在一个无限宇宙中进行，但是我们没有无限的内存和计算能力。
解决这个烦人的限制通常使用三种方法之一：

1. 保持跟踪正在发生有趣事情的宇宙的子集，然后按需要扩展这个区域。
   最差的情况是扩展是没有边界的，并且实现将会越来越慢并最终耗尽内存。

2. 创建固定尺寸的宇宙，这样位于边缘的细胞会比在中间的细胞拥有更少的邻居。
   这个方法的缺点是像滑翔机那样的无穷模式在到达宇宙边界后就被消灭了。

3. 创建一个固定尺寸的周期行宇宙，这样位于边缘的细胞拥有环绕到宇宙另一侧的邻居。
   因为邻居环绕宇宙的边缘，滑翔机可以永远运行。

我们将实现第三个选项。

### Rust和JavaScript接口技术

> ⚡ 这是需要从理解并从本教程中学会的最重要的概念之一。

JavaScript的垃圾收集堆内存不同于WebAssembly的线性内存空间，
前者是分配`Object`，`Array`和DOM节点的地方，后者存放我们Rust的值。
WebAssembly目前没有直接访问垃圾回收堆内存
（截至2018年四月，期望这将随着["host bindings"提案][host-bindings]改变）。
换句话说JavaScript可以读写WebAssembly的线性内存空间，
但只能作为标量值（ `u8`, `i32`, `f64`等）的[`ArrayBuffer`][array-buf]。
WebAssembly函数也获取和返回标量值。
这些是构成WebAssembly和JavaScript通信的所有组件。

[host-bindings]: https://github.com/WebAssembly/host-bindings/blob/master/proposals/host-bindings/Overview.md
[array-buf]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer

`wasm_bindgen`定义了如何在这个边界上使用复合结构的共识。
它涉及包装Rust结构并将指针包装在JavaScript类中以便于使用，或者从Rust中索引JavaScript对象表。
`wasm_bindgen`非常方便，但是它没有移除考虑我们的数据表示以及什么值和结构被传递到这个边界的需求。
相反，将它视为一个用于实现你选择的界面设计的工具。

当你在WebAssembly和JavaScript之间设计接口的时候，我们想要优化下面的性能：

1. **最小化复制进出WebAssembly的线性内存。**
   不必要的复制会产生不必要的开销。

2. **最小化序列化和反序列化。** 
   类似于复制，序列化和反序列化也会产生开销，并且通常也进行复制。
   如果我们可以传递不透明的句柄到数据结构，
   而不用在一端序列化然后将其拷贝到WebAssembly线性内存的某个已知位置再从另一端反序列化，
   这样我们通常能减少大量开销。
   `wasm_bindgen`帮助我们定义和处理JavaScript的`Object`或封装的Rust结构的不透明句柄。

作为一般经验法则，一个好的JavaScript↔WebAssembly接口设计通常是将大型，长寿命的数据结构实现为存在
WebAssembly线性内存中的Rust类型，并作为不透明句柄暴露给JavaScript。 
JavaScript调用导出的WebAssembly函数，这个函数获取不透明句柄，
传递他们的数据，执行繁重的计算，查询数据，并最终返回小的，可复制的结果。 
只有通过返回计算的小结果，我们才能避免拷贝和序列化所有在
JavaScript垃圾收集堆内存和WebAssembly线性内存之间来回传递的所有内容。

### 我们的生命游戏中的Rust和JavaScript的接口技术

我们先来列举一些要避开的危险。我们不想要每一个tick都将整个宇宙拷贝进出到WebAssembly的线性内存。
我们不想要为宇宙中的每个单元分配对象，也不想强加一个跨边界调用来读写每个单元。

这给我们留下了什么？ 
我们可以将宇宙表示为一个平面数组，它存储在WebAssembly的线性内存中，每个单元格有一个字节。
`0`是死细胞，`1`是活细胞。

这是一个4*4宇宙在内存中的样子：

![Screenshot of a 4 by 4 universe](../images/game-of-life/universe.png)

为了找到宇宙中给定行列单元格的数组索引，我们可以使用这个公式：

```text
index(row, column, universe) = row * width(universe) + column
```

我们有几个方法将宇宙中的单元格暴露给JavaScript。
首先我们将为`Universe`实现[`std::fmt::Display`][`Display`]，
这样我们可以用来生成一个表现为文本字符的单元格的Rust`String`。
然后这个Rust String被从WebAssembly的线性内存拷贝到JavaScript的垃圾收集堆内存上的JavaScript字符串，
然后通过设置HTML的`textContent`来展示。
在本节的后面，我们将改变这个实现以避免在堆内存之间拷贝宇宙单元格和渲染到`<canvas>`.

*另一个可行的设计是Rust在每个tick后返回一个改变状态的单元格的列表，而不用将这个宇宙暴露给JavaScript。
 这样JavaScript在渲染的时候不需要迭代整个宇宙，只需要一个对应的子集。
 这种增量的设计实现起来稍微有些困难。*

## Rust实现

在上一个节中，我们克隆了一个初始的模板项目。我们现在将要修改那个模板项目。

我们首先从`wasm-game-of-life/src/lib.rs`中移除`alert`导入和`greet`函数，
然后用一个对单元格的类型定义取代他们：

```rust
#[wasm_bindgen]
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Cell {
    Dead = 0,
    Alive = 1,
}
```

这里有一个`#[repr(u8)]`是很重要的，这样每个单元格被表示为单个字节。
`Dead`表示为`0`而`Alive`表示为`1`也很重要，这样我们能够轻松的用加计算一个单元格的存活邻居数。

接下来让我们定义宇宙。宇宙有一个长度和宽度以及一个具有`width * height`长度的向量。

```rust
#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<Cell>,
}
```

为了访问给定行列的单元格，我们吧行列翻译为单元格向量的索引，正如前面描述的那样：

```rust
impl Universe {
    fn get_index(&self, row: u32, column: u32) -> usize {
        (row * self.width + column) as usize
    }

    // ...
}
```

为了计算单元格的下一个状态，我们需要得到一个有多少邻居存活的计数。
让我们写一个`live_neighbor_count`方法来做这个。

```rust
impl Universe {
    // ...

    fn live_neighbor_count(&self, row: u32, column: u32) -> u8 {
        let mut count = 0;
        for delta_row in [self.height - 1, 0, 1].iter().cloned() {
            for delta_col in [self.width - 1, 0, 1].iter().cloned() {
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }

                let neighbor_row = (row + delta_row) % self.height;
                let neighbor_col = (column + delta_col) % self.width;
                let idx = self.get_index(neighbor_row, neighbor_col);
                count += self.cells[idx] as u8;
            }
        }
        count
    }
}
```

`live_neighbor_count`方法使用增加和模来避免使用if对宇宙边缘进行特殊的覆盖。
当应用 `-1`的增量时，我们*加*`self.height - 1`并让模数做它的事而不是尝试去减`1`。
`row`和`column`可以是`0`，如果我们试图对他们减`1`，将会有一个无符号整数下溢。

现在我们有了从当前一代计算出下一代所需要的全部内容了。每一条游戏规则都被直接翻译为一个`match`表达式的条件。
另外，由于我们希望当ticks发生的时候JavaScript进行控制，所以我们将把这个方法放到一个`#[wasm_bindgen]`块中，
这样它就才能暴露给JavaScript。

```rust
/// 公共函数，暴露给JavaScript。
#[wasm_bindgen]
impl Universe {
    pub fn tick(&mut self) {
        let mut next = self.cells.clone();

        for row in 0..self.height {
            for col in 0..self.width {
                let idx = self.get_index(row, col);
                let cell = self.cells[idx];
                let live_neighbors = self.live_neighbor_count(row, col);

                let next_cell = match (cell, live_neighbors) {
                    // 规则 1:任何存活细胞周围少于两个存活邻居则死亡，仿佛是人口过少造成的。
                    (Cell::Alive, x) if x < 2 => Cell::Dead,
                    // 规则 2: 任何存活细胞周围有两个或三个存活邻居则活到下一代。
                    (Cell::Alive, 2) | (Cell::Alive, 3) => Cell::Alive,
                    // 规则 3: 任何存活细胞周围有超过三个存活邻居则死亡，仿佛是人口过剩造成的。
                    (Cell::Alive, x) if x > 3 => Cell::Dead,
                    // Rule 4: 任何死亡细胞周围有正好三个存活细胞则变成存活细胞，仿佛是生殖。
                    (Cell::Dead, 3) => Cell::Alive,
                    // 所有其他单元格保持同样的状态。
                    (otherwise, _) => otherwise,
                };

                next[idx] = next_cell;
            }
        }

        self.cells = next;
    }

    // ...
}
```

到目前为止，宇宙的状态被表示为一个单元格向量。为了是人可读，让我们实现一个基本的文本渲染器。
方法是将宇宙一行行以文本形式写下来，对于每个存活细胞，打印Unicode字符`◼`（“黑色中方框”)。
对于每个死细胞，我们打印`◻`(一个“白色中方框”)。

通过实现Rust标准库中的[`Display`] trait，我们可以添加一个方法来以面向用户的方式格式化一个结构。
这将会自动的给我们一个[`to_string`]方法。

[`Display`]: https://doc.rust-lang.org/1.25.0/std/fmt/trait.Display.html
[`to_string`]: https://doc.rust-lang.org/1.25.0/std/string/trait.ToString.html

```rust
use std::fmt;

impl fmt::Display for Universe {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        for line in self.cells.as_slice().chunks(self.width as usize) {
            for &cell in line {
                let symbol = if cell == Cell::Dead { '◻' } else { '◼' };
                write!(f, "{}", symbol)?;
            }
            write!(f, "\n")?;
        }

        Ok(())
    }
}
```

最后我们定义一个构造函数，它使用有意思的生死细胞模式来初始化宇宙，也有一个`render`方法：

```rust
/// 公共方法，暴露给JavaScript。
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn new() -> Universe {
        let width = 64;
        let height = 64;

        let cells = (0..width * height)
            .map(|i| {
                if i % 2 == 0 || i % 7 == 0 {
                    Cell::Alive
                } else {
                    Cell::Dead
                }
            })
            .collect();

        Universe {
            width,
            height,
            cells,
        }
    }

    pub fn render(&self) -> String {
        self.to_string()
    }
}
```

这样，我们的Rust版本的生命游戏已经完成一半了！

在`wasm-game-of-life`目录中运行`wasm-pack build`将它编译为WebAssembly。

## 使用JavaScript渲染

首先，我们天加一个`<pre>`元素到wasm-game-of-life/www/index.html`中，
宇宙就在其中渲染，添加的位置就在`<script>`标签上方：

```html
<body>
  <pre id="game-of-life-canvas"></pre>
  <script src="./bootstrap.js"></script>
</body>
```

另外，我们希望`<pre>`位于Web页面的中间。我们可以使用CSS的flex boxes来完成这个任务。
添加下面的 `<style>`标签到`wasm-game-of-life/www/index.html`的`<head>`中：

```html
<style>
  body {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
  }
</style>
```

在`wasm-game-of-life/www/index.js`的顶端，让我们修正一个我们的导入，
应该导入`Universe`而不是旧的`greet`函数。

```js
import { Universe } from "wasm-game-of-life";
```

然后，我们获取刚刚添加的`<pre>`元素并实例化一个新的宇宙：

```js
const pre = document.getElementById("game-of-life-canvas");
const universe = Universe.new();
```

JavaScrit运行在[一个`requestAnimationFrame`循环][requestAnimationFrame]中，
在每一次迭代时，它将当前宇宙画到`<pre>`中，然后调用`Universe::tick`。

[requestAnimationFrame]: https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame

```js
const renderLoop = () => {
  pre.textContent = universe.render();
  universe.tick();

  requestAnimationFrame(renderLoop);
};
```

为了启动渲染进程，我们需要做的所有事情就是为了第一期渲染循环进行初始化调用：

```js
requestAnimationFrame(renderLoop);
```

确定你的开发环境服务还在运行（`wasm-game-of-life/www`目录内运行`npm run start`）
[http://localhost:8080/](http://localhost:8080/)看起来应该是这样的：

[![Screenshot of the Game of Life implementation with text rendering](../images/game-of-life/initial-game-of-life-pre.png)](../images/game-of-life/initial-game-of-life-pre.png)

## 直接从内存渲染到画布

在Rust中生成（和分配）一个`String`字符串
然后用`wasm-bindgen`将其转换为一个有效的JavaScript字符串会产生不必要的宇宙单元格的副本。
因为JavaScript代码已经知道了宇宙的宽和高，并且可以读取直接构成单元格的WebAssembly的线性内存，
所以我们将修改`render`方法使其返回一个指向单元格数组起始位置的指针。

此外，我们将使用[Canvas API]来代替渲染Unicode文本。
在本教程的剩下部分，我们将使用这个设计。

[Canvas API]: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API

在`wasm-game-of-life/www/index.html`中，我们用一个`<canvas>`取代我们之前添加的`<pre>`，
我们将在其中渲染（它也应该在`<body>`内，在加载我们的JavaScript代码的`<script>`前）：

```html
<body>
  <canvas id="game-of-life-canvas"></canvas>
  <script src='./bootstrap.js'></script>
</body>
```

为了在Rust实现中得到必要的信息，我们需要添加一些额外的获取函数
用于取得宇宙的宽高以及指向它的单元格数组的指针。
所有的这些都被暴露给JavaScript。
将这些添加到`wasm-game-of-life/src/lib.rs`：

```rust
/// 公共方法，暴露给JavaScript。
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn width(&self) -> u32 {
        self.width
    }

    pub fn height(&self) -> u32 {
        self.height
    }

    pub fn cells(&self) -> *const Cell {
        self.cells.as_ptr()
    }
}
```

接下来，在`wasm-game-of-life/www/index.js`中我们也从`wasm-game-of-life`导入`Cell`，
并定义一些我们将在渲染到画布时使用的常量。

```js
import { Universe, Cell } from "wasm-game-of-life";

const CELL_SIZE = 5; // px
const GRID_COLOR = "#CCCCCC";
const DEAD_COLOR = "#FFFFFF";
const ALIVE_COLOR = "#000000";
```

现在让我们重写这个JavaScript代码的剩下部分，不再是输出到`<pre>`的`textContent`而是画到`<canvas>`：

```js
// 构建宇宙，并获取它的宽高。
const universe = Universe.new();
const width = universe.width();
const height = universe.height();

// 给我们的所有单元格提供画布空间并且在他们周围又1像素的边界。
const canvas = document.getElementById("game-of-life-canvas");
canvas.height = (CELL_SIZE + 1) * height + 1;
canvas.width = (CELL_SIZE + 1) * width + 1;

const ctx = canvas.getContext('2d');

const renderLoop = () => {
  universe.tick();

  drawGrid();
  drawCells();

  requestAnimationFrame(renderLoop);
};
```

为了在细胞之间绘制格子，我们画了一组等距的竖线和一组等距的横线。
这些线相互交错行程网格。

```js
const drawGrid = () => {
  ctx.beginPath();
  ctx.strokeStyle = GRID_COLOR;

  // 竖线。
  for (let i = 0; i <= width; i++) {
    ctx.moveTo(i * (CELL_SIZE + 1) + 1, 0);
    ctx.lineTo(i * (CELL_SIZE + 1) + 1, (CELL_SIZE + 1) * height + 1);
  }

  // 横线。
  for (let j = 0; j <= height; j++) {
    ctx.moveTo(0,                           j * (CELL_SIZE + 1) + 1);
    ctx.lineTo((CELL_SIZE + 1) * width + 1, j * (CELL_SIZE + 1) + 1);
  }

  ctx.stroke();
};
```

我们可以通过`memory`直接访问WebAssembly的线性内存，它定义在原生的wasm模块`wasm_game_of_life_bg`中。
为了绘制细胞，我们获取一个指向宇宙细胞的指针，构造一个`Uint8Array`来叠加细胞缓冲区，
在每个细胞上迭代，并绘制一个白的或黑的矩形，这取决于细胞是死的还是活的。
通过使用指针和叠加，我们避免在每个tick上跨越边界复制细胞。
tick.

```js
// 在文件的顶部导入WebAssembly内存。
import { memory } from "wasm-game-of-life/wasm_game_of_life_bg";

// ...

const getIndex = (row, column) => {
  return row * width + column;
};

const drawCells = () => {
  const cellsPtr = universe.cells();
  const cells = new Uint8Array(memory.buffer, cellsPtr, width * height);

  ctx.beginPath();

  for (let row = 0; row < height; row++) {
    for (let col = 0; col < width; col++) {
      const idx = getIndex(row, col);

      ctx.fillStyle = cells[idx] === Cell.Dead
        ? DEAD_COLOR
        : ALIVE_COLOR;

      ctx.fillRect(
        col * (CELL_SIZE + 1) + 1,
        row * (CELL_SIZE + 1) + 1,
        CELL_SIZE,
        CELL_SIZE
      );
    }
  }

  ctx.stroke();
};
```

为了启动渲染进程，我们将使用相同和上面启动渲染循环的第一次迭代的代码：

```js
drawGrid();
drawCells();
requestAnimationFrame(renderLoop);
```

注意这里我们在调用`requestAnimationFrame()`前调用了`drawGrid()`和`drawCells()`。
我们这么做的原因是初始状态的宇宙在我们做出修改*前*就被绘制了。
如果我们简单的调用`requestAnimationFrame(renderLoop)`，我们会陷入一个困境即，
被绘制的第一帧实际上是在第一次调用`universe.tick()`*后*的，这是这些细胞生命的第二个“tick”。

## 成功了！

通过在`wasm-game-of-life`根目录中运行这个命令来重新构建WebAssembly并绑定胶水代码：

```
wasm-pack build
```

确保你的开发服务还在运行。如果没有，在`wasm-game-of-life/www`目录中再次启动它：

```
npm run start
```

如果你刷新[http://localhost:8080/](http://localhost:8080/)，你应该会被一个令人激动的生命展示问候！

[![Screenshot of the Game of Life implementation](../images/game-of-life/initial-game-of-life.png)](../images/game-of-life/initial-game-of-life.png)

顺便说一句，还有一个非常简洁的称为[hashlife](https://en.wikipedia.org/wiki/Hashlife)的算法来实现生命游戏。
它使用积极的记忆并且实际上随着运行时间越来越长，它实际上可以得到*成倍的加快*计算下一代。 
鉴于此，你好奇为什么我们没有在这个教程中实现hashlife。
它超出了这篇文章的范畴，这里我们关注于Rust和WebAssembly的继承，但是我们非常鼓励你自己去学习hashlife。


## 练习

* 用单个空间船初始化宇宙

* 使用一个随机的初始化宇宙来代替硬编码的初始化宇宙，让每个细胞有是存活或死亡的几率各有50%。

  *提示: 使用[the `js-sys` crate](https://crates.io/crates/js-sys)导入[`Math.random` JavaScript函数]
  (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/random)。*

  <details>
    <summary>答案</summary>
    *首先在`wasm-game-of-life/Cargo.toml`中将`js-sys`添加为依赖：*
  
    ```toml
    # ...
    [dependencies]
    js-sys = "0.3"
    # ...
    ```
  
    *然后，使用`js_sys::Math::random`函数投掷一枚硬币：*
  
    ```rust
    extern crate js_sys;
  
    // ...
  
    if js_sys::Math::random() < 0.5 {
        // Alive...
    } else {
        // Dead...
    }
    ```
  </details>

* 用一个字节表示一个细胞使得在细胞上迭代很容易进行，但是它的代价是内存浪费。
  每个字节是八比特，但是我们只需要一个比特来表示每个细胞是存活或者死亡。
  重构代码表示，使得每个细胞仅使用一比特的空间。

  <details>
    <summary>答案</summary>

    I在Rust中，你可以使用[`fixedbitset` crate和它的`FixedBitSet`类型]
    (https://crates.io/crates/fixedbitset)来代替`Vec<Cell>`表示细胞：

    ```rust
    // Make sure you also added the dependency to Cargo.toml!
    extern crate fixedbitset;
    use fixedbitset::FixedBitSet;

    // ...

    #[wasm_bindgen]
    pub struct Universe {
        width: u32,
        height: u32,
        cells: FixedBitSet,
    }
    ```
    
    宇宙的构造器可以按如下方式调整：
    
    ```rust
    pub fn new() -> Universe {
        let width = 64;
        let height = 64;

        let size = (width * height) as usize;
        let mut cells = FixedBitSet::with_capacity(size);

        for i in 0..size {
            cells.set(i, i % 2 == 0 || i % 7 == 0);
        }

        Universe {
            width,
            height,
            cells,
        }
    }
    ```

    更新宇宙中一个细胞在下一个tick的状态，我们使用`FixedBitSet`的`set`方法：

    ```rust
    next.set(idx, match (cell, live_neighbors) {
        (true, x) if x < 2 => false,
        (true, 2) | (true, 3) => true,
        (true, x) if x > 3 => false,
        (false, 3) => true,
        (otherwise, _) => otherwise
    });
    ```

    给JavaScript传递一个指向比特开始处的指针，你可以将`FixedBitSet`转换为一个slice然后再转换成指针：

    ```rust
    #[wasm_bindgen]
    impl Universe {
        // ...

        pub fn cells(&self) -> *const u32 {
            self.cells.as_slice().as_ptr()
        }
    }
    ```

    在JavaScript中，从Wasm内存中构建一个`Uint8Array`与之前一样，
    除了数组长度不再是`width * height`而是`width * height / 8`，
    因为我们每个比特表示一个细胞不再是一个字节表示一个细胞。

    ```js
    const cells = new Uint8Array(memory.buffer, cellsPtr, width * height / 8);
    ```

    给定索引和`Uint8Array`你可以使用下面的函数确定是否将第n位比特置位：

    ```js
    const bitIsSet = (n, arr) => {
      const byte = Math.floor(n / 8);
      const mask = 1 << (n % 8);
      return (arr[byte] & mask) === mask;
    };
    ```

    综上所述，新版本的`drawCells`看起来是这样的：

    ```js
    const drawCells = () => {
      const cellsPtr = universe.cells();

      // This is updated!
      const cells = new Uint8Array(memory.buffer, cellsPtr, width * height / 8);

      ctx.beginPath();

      for (let row = 0; row < height; row++) {
        for (let col = 0; col < width; col++) {
          const idx = getIndex(row, col);

          // This is updated!
          ctx.fillStyle = bitIsSet(idx, cells)
            ? ALIVE_COLOR
            : DEAD_COLOR;

          ctx.fillRect(
            col * (CELL_SIZE + 1) + 1,
            row * (CELL_SIZE + 1) + 1,
            CELL_SIZE,
            CELL_SIZE
          );
        }
      }

      ctx.stroke();
    };
    ```

  </details>
