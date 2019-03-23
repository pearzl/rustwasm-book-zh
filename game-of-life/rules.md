*commit/22a35861cc68589a2c02d498ee827c68264c73e2*

# 康威生命游戏的规则

*注意：如果你已经熟悉康威生命游戏和它的规则，你可以直接跳过进入下一节。*

[维基百科给康威生命游戏一个很好的描述][wikipedia]

> 生命游戏的宇宙是一个无限二维正交方格单元格
>（*译注：也翻译为细胞，后文中将视情况使用两个词汇，他们表示相同的意思*），
> 每个细胞都有两种状态，生或者死，或者是“人口稠密”或“无人居住”。
> 每个细胞与它的八个相邻细胞相互作用，这八个细胞是水平，垂直或者对角线相邻。
> 在每个步骤中，发生下列转换：
>
> 1. 任何存活细胞周围少于两个存活邻居则死亡，仿佛是人口过少造成的。
>
> 2. 任何存活细胞周围有两个或三个存活邻居则活到下一代。
>
> 3. 任何存活细胞周围有超过三个存活邻居则死亡，仿佛是人口过剩造成的。
>
> 4. 任何死亡细胞周围有正好三个存活细胞则变成存活细胞，仿佛是生殖。
>
> 初始模式构成了系统的种子。
> 第一代是由每个种子细胞应用上述规则同时产生的——生和死同时发生，
> 并且每一次发生这个的离散时刻称为一个tick。
> （换句话说，每一代都是前一代的纯函数）。
> 规则继续重复应用以进一步创造下一代。

[wikipedia]: https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life

考虑下面的初始宇宙：

<img src='../images/game-of-life/initial-universe.png' alt='Initial Universe' width=80 />

我们可以通过考虑每个细胞计算出下一代。左上角的细胞死了。规则（4）是适用于死亡细胞的唯一规则。
但是因为左上角的细胞没有正好三个存活邻居，所以转换规则没有生效，并且在下一代继续保持死亡。
第一行的其他细胞也是一样的。

当我们考虑在第二行第三列的顶上的存活细胞时事情变得有意思了。
对于存活细胞，三条规则中的任何一条都可能被应用。
这个细胞的情况是它只有一个存活的邻居，因此规则（1）被应用：这个细胞将在下一代死亡。
同样的命运也等待着底部的那个存活细胞。

中间的那个存活细胞有两个存活邻居：顶部和底部的存活细胞。
这意味着规则（2）被应用，他将在下一代保持存活。

最后有趣的情况是在中间活细胞左右两侧的死细胞。
三个活细胞都是他们的邻居，这意味着规则（4）被应用，细胞将在下一代变为活细胞。

将他们放到一起，我们得到这个宇宙在下一个tick之后的样子：

<img src='../images/game-of-life/next-universe.png' alt='Next Universe' width=80 />

从这些简单，确定的规则出现，奇怪和令人兴奋的行为出现了：

| Gosper的滑翔机枪 | 脉冲 | 太空船 |
|---|---|---|
| ![Gosper's glider gun](https://upload.wikimedia.org/wikipedia/commons/e/e5/Gospers_glider_gun.gif) | ![Pulsar](https://upload.wikimedia.org/wikipedia/commons/0/07/Game_of_life_pulsar.gif) | ![Lighweight space ship](https://upload.wikimedia.org/wikipedia/commons/3/37/Game_of_life_animated_LWSS.gif) |

<center>
<iframe width="560" height="315" src="https://www.youtube.com/embed/C2vgICfQawE?rel=0&amp;start=65" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
</center>

## 练习

* 手动计算我们示例宇宙的下一个tick。注意到有些熟悉了吗？

  <details>
    <summary>答案</summary>

    它应该是我们示例宇宙的初始状态：

    <img src='../images/game-of-life/initial-universe.png' alt='Initial Universe' width=80 />

    这个模式是*周期性的*：他在每两个tick之后回到初始状态。

  </details>

* 你能发现一个稳定的初始宇宙吗？即，一个每一代都相同的宇宙。

  <details>
    <summar>答案</summary>

    有无穷数量的稳定宇宙！最平凡的穩定宇宙是一个空宇宙。
    一个二乘二的活细胞方格也是一个稳定宇宙。

  </details>
