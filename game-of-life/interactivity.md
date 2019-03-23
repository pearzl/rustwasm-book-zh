*commit/57962c71c55510f5ed310f12a1957c593f7663cc*

# 增加交互性

通过添加一些交互特性到我们的生命游戏视线中，我们将继续探索JavaScript和WebAssembly接口。

## 暂停和恢复游戏

让我们添加一个按钮用于切换游戏运行或者暂停。
在`wasm-game-of-life/www/index.html`中，把按钮添加到`<canvas>`上方。

```html
<button id="play-pause"></button>
```

在`wasm-game-of-life/www/index.js` JavaScript中，我们将进行以下改变：

* 追踪由最近一次调用`requestAnimationFrame`返回的标识符，
  这样我们可以使用那个标识符调用`cancelAnimationFrame`来取消动画。

* 当暂停或运行按钮被点击，检查我们是否有一个派对的动画帧的标识符。
  如果有，那么游戏当前在运行，我们想要取消动画帧，所以`renderLoop`不会被再次调用，有效的暂停了游戏。
  如果没有，那么目前被暂停了，我们将会调用`requestAnimationFrame`来恢复游戏。

由于JavaScript驱动Rust and WebAssembly，这就是我们需要做的全部了，我们不需要修改Rust源代码。

我们引入了`animationId`变量来追踪`requestAnimationFrame`返回的标识符。
当没有派对的动画帧时，我们将这个变量设置为`null`。

```js
let animationId = null;

// 这个函数和以前一样，除了`requestAnimationFrame`的结果被分配给了`animationId`.
const renderLoop = () => {
  drawGrid();
  drawCells();

  universe.tick();

  animationId = requestAnimationFrame(renderLoop);
};
```

在任何时刻，我们都可以通过检查`animationId`的值来判断游戏是否暂停：

```js
const isPaused = () => {
  return animationId === null;
};
```

现在当继续/暂停按钮被电机的时候，我们检查游戏当前是运行还是暂停，
然后分别恢复`renderLoop`动画或者取消下一个动画帧。
此外，我们更新按钮的文本图标以反应按钮在下次单机时将执行的操作。

```js
const playPauseButton = document.getElementById("play-pause");

const play = () => {
  playPauseButton.textContent = "⏸";
  renderLoop();
};

const pause = () => {
  playPauseButton.textContent = "▶";
  cancelAnimationFrame(animationId);
  animationId = null;
};

playPauseButton.addEventListener("click", event => {
  if (isPaused()) {
    play();
  } else {
    pause();
  }
});
```

最后，我们之前时通过直接调用`requestAnimationFrame(renderLoop)`来启动游戏和动画的，
但是我们想用一个调用来替换他，一边按钮获得正确的初始文本图标。

```diff
// 这里原来是 `requestAnimationFrame(renderLoop)`.
play();
```

刷新[http://localhost:8080/](http://localhost:8080/)我们现在通过点击按钮应该能够暂停和恢复游戏了。

## 点击切换细胞状态

现在我们已经可以暂停游戏了，是时候添加点击使细胞变异的功能了。

使细胞变异是指从存活到死亡或者从死亡到存活的翻转它的状态。
在`wasm-game-of-life/src/lib.rs`中为`Cell`添加一个`toggle`方法：

```rust
impl Cell {
    fn toggle(&mut self) {
        *self = match *self {
            Cell::Dead => Cell::Alive,
            Cell::Alive => Cell::Dead,
        };
    }
}
```

为了改变指定行列的细胞状态，我们将行列对翻译为细胞向量的索引，然后调用那个索引位置上的细胞的翻转方法：

```rust
/// 公共方法，暴露给JavaScript
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn toggle_cell(&mut self, row: u32, column: u32) {
        let idx = self.get_index(row, column);
        self.cells[idx].toggle();
    }
}
```

这个方法定义再`impl`块中，它带有`#[wasm_bindgen]`注解，这样JavaScript才能调用它。

在`wasm-game-of-life/www/index.js`中，我们在`<canvas>`元素上监听点击事件，
将点击事件的页面相对坐标翻译为画布相对坐标，然后转换为行和列，
调用`toggle_cell`方法，最后重绘场景。

```js
canvas.addEventListener("click", event => {
  const boundingRect = canvas.getBoundingClientRect();

  const scaleX = canvas.width / boundingRect.width;
  const scaleY = canvas.height / boundingRect.height;

  const canvasLeft = (event.clientX - boundingRect.left) * scaleX;
  const canvasTop = (event.clientY - boundingRect.top) * scaleY;

  const row = Math.min(Math.floor(canvasTop / (CELL_SIZE + 1)), height - 1);
  const col = Math.min(Math.floor(canvasLeft / (CELL_SIZE + 1)), width - 1);

  universe.toggle_cell(row, col);

  drawGrid();
  drawCells();
});
```

在`wasm-game-of-life`中进行重构`wasm-pack build`，然后再次刷新
[http://localhost:8080/](http://localhost:8080/)，
我们现在可以通过点击细胞翻转他们的状态来绘制我们自己的图案了。

## 练习

* 引入一个[`<input type="range">`][input-range]不见来控制每个动画帧发生多少次tick。

* 添加一个按钮，在点击的时候将宇宙重置到随机的初始状态。另一个按钮将所有细胞设置为死亡。

* `Ctrl + Click`插入一个[滑翔机](https://en.wikipedia.org/wiki/Glider_(Conway%27s_Life))
  到目标单元格。`Shift + Click`插入一个脉冲。

[input-range]: https://developer.mozilla.org/en-US/docs/Web/HTML/Element/input/range
