*commit/83404c7eb6ed0f9db524dd0b78538482a6990620*

# 发布到npm

现在我们有一个正常工作的，快速的，*并且*小体积的`wasm-game-of-life`包，我们可以将它发布到npm，
这样如果其他JavaScript开发者需要一个现成的生命游戏的实现时可以复用它。

## 先决条件

首先，[确保你有一个npm账号](https://www.npmjs.com/signup).

其次，确保你已经在本地登陆了你的账号，运行这个命令：

```
wasm-pack login
```

## 发布

通过在`wasm-game-of-life`目录中运行`wasm-pack`确保`wasm-game-of-life/pkg`构建已经更新：

```
wasm-pack build
```

现在花点时间检查一下`wasm-game-of-life/pkg`的内容，这是我们下一步发布到npm的内容。

如果你准备好了，运行`wasm-pack publish`将包上传到npm：

```
wasm-pack publish
```

这就是发布到npm的全部！

因为其他人也做了这个教程，所以`wasm-game-of-life`这个名字在npm上已经被占用了，因此最后一个命令可能不能生效。

打开`wasm-game-of-life/Cargo.toml`并把你的名字添加到`name`的后面，用一个独一无二的方法让这个包变得不同。

```toml
[package]
name = "wasm-game-of-life-my-username"
```

然后，重新构建并再次发布：

```
wasm-pack build
wasm-pack publish
```

这次应该生效了。
