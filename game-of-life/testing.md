*commit/76ff4c91e28b696fd11c4c4b31b63e0457d39f8a*

# 测试康威生命游戏

现在我们已经拥有了使用JavaScript渲染在浏览器中的生命游戏的Rust实现，
让我们讨论一下测试我们的Rust生成的WebAssembly函数。

我们将测试我们的`tick`函数以确保它给出了我们期望的输出。

接下来，我们想要在我们已经存在于`wasm_game_of_life/src/lib.rs`文件的`impl Universe`块里创建一些setter和getter函数。我们将创建一个`set_width`和一个`set_height`功能，所以我们可以创建不同尺寸的宇宙。

```rust
#[wasm_bindgen]
impl Universe { 
    // ...

    /// Set the width of the universe.
    ///
    /// Resets all cells to the dead state.
    pub fn set_width(&mut self, width: u32) {
        self.width = width;
        self.cells = (0..width * self.height).map(|_i| Cell::Dead).collect();
    }

    /// Set the height of the universe.
    ///
    /// Resets all cells to the dead state.
    pub fn set_height(&mut self, height: u32) {
        self.height = height;
        self.cells = (0..self.width * height).map(|_i| Cell::Dead).collect();
    }

}
```

我们将在`wasm_game_of_life/src/lib.rs`文件中不使用`#[wasm_bindgen]`属性来创造另一个`impl Universe`块。
有一些我们测试需要但不想暴露给JavaScript的函数。
Rust生成的WebAssembly函数不能返回被借用的引用。
尝试编译带有属性的Rust生成的WebAssembly并看一看你得到的错误。

我们将要编写`get_cells`的实现，它用于取得狱中中细胞的内容。
我们也写一个`set_cells`函数使得我们可以将宇宙中特定寒冽的细胞置为存活。

```rust
impl Universe {
    /// 获取整个宇宙中生存和死亡数
    pub fn get_cells(&self) -> &[Cell] {
        &self.cells
    }

    /// 通过将行和列以数组的形式传递给宇宙中每一个需要设为存活的细胞。
    pub fn set_cells(&mut self, cells: &[(u32, u32)]) {
        for (row, col) in cells.iter().cloned() {
            let idx = self.get_index(row, col);
            self.cells[idx] = Cell::Alive;
        }
    }

}
```

现在我们将在`wasm_game_of_life/tests/web.rs`中创建我们的测试。

在我们开始之前，文件中已经有一个工作中的测试了。
你可以通过在`wasm-game-of-life`目录下运行`wasm-pack test --chrome --headless`来确认
Rust生成的WebAssembly而是正在运行。

你也可以使用`--firefox`，`--safari`和`node`选项来在其他浏览器中测试你们的代码。


在`wasm_game_of_life/tests/web.rs`文件中，我们需要导出我们的`wasm_game_of_life`包和宇宙类型。

```rust
extern crate wasm_game_of_life;
use wasm_game_of_life::Universe;
```

在`wasm_game_of_life/tests/web.rs`文件中我们创造一些太空船构建函数。

我们希望得到这样一个函数，即调用输入的太空船上的`tick`函数之后，将在下一个tick得到我们希望的太空船。
在`input_spaceship`函数中，我选取了那些希望在初始化时被视为`Alive`的细胞来创建我们的的太空船。
在`input_spaceship`的tick之后的`expected_spaceship`函数中，太空船的被手动计算。
你可以自行确认，输入的太空船细胞在一个tick之后和期望的太空船一样。

```rust
#[cfg(test)]
pub fn input_spaceship() -> Universe {
    let mut universe = Universe::new();
    universe.set_width(6);
    universe.set_height(6);
    universe.set_cells(&[(1,2), (2,3), (3,1), (3,2), (3,3)]);
    universe
}

#[cfg(test)]
pub fn expected_spaceship() -> Universe {
    let mut universe = Universe::new();
    universe.set_width(6);
    universe.set_height(6);
    universe.set_cells(&[(2,1), (2,3), (3,2), (3,3), (4,2)]);
    universe
}
```

现在我们要写出`test_tick`函数的实现。
首先，我们创建一个`input_spaceship()`和`expected_spaceship()`的实例。
然后骂我们在`input_universe`上调用`tick`。
最后我们使用`assert_eq!`宏来调用`get_cells()`以确保`input_universe`
和`expected_universe`有相同的`Cell`序列值。
我们在代码块上添加了`#[wasm_bindgen_test]`属性，这样我们可以测试
Rust生成的WebAssembly代码并使用`wasm-build test`来测试WebAssembly代码。

```rust
#[wasm_bindgen_test]
pub fn test_tick() {
    // 让我们船舰一个小一些的宇宙和太空船来测试。
    let mut input_universe = input_spaceship();

    // 这是我们的太空船在一个tick之后的样子。
    let expected_universe = expected_spaceship();

    // 调用`tick`然后看看`Universe`中的细胞是否一样。.
    input_universe.tick();
    assert_eq!(&input_universe.get_cells(), &expected_universe.get_cells());
}
```

在`wasm-game-of-life`目录下执行`wasm-pack test --firefox --headless`开始测试。
