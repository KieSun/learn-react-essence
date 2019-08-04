这是我的剖析 React 源码的第六篇文章。这篇文章连接上篇，将会带着大家学习组件更新过程相关的内容，并且尽可能的脱离源码来了解原理，降低大家的学习难度。

## 文章相关资料

- [React 16.8.6 源码中文注释](https://github.com/KieSun/react-interpretation)，如果你想读源码但是又怕看不懂的话，可以通过我这个仓库来学习
- [之前的所有文章合集](https://github.com/KieSun/learn-react-essence)

## 这篇文章你能学到什么？

文章分为三部分，在这部分的文章中你可以学习到如下内容：
- 调和的过程

三篇文章并没有强相关性，当然还是推荐阅读下 [前一篇文章](https://github.com/KieSun/learn-react-essence/blob/master/%E7%BB%84%E4%BB%B6%E6%9B%B4%E6%96%B0%E6%B5%81%E7%A8%8B%EF%BC%88%E4%B8%80%EF%BC%89.md)。

## 调和的过程

组件更新归结到底还是 DOM 的更新。对于 React 来说，这部分的内容会分为两个阶段：

1. 调和阶段，基本上也就是大家熟知的虚拟 DOM 的 diff 算法
2. 提交阶段，也就是将上一个阶段中 diff 出来的内容体现到 DOM 上

这一小节的内容将会集中在调和阶段，提交阶段这部分的内容将会在下一篇文章中写到。另外大家所熟知的虚拟 DOM 的 diff 算法在新版本中其实已经完全被重写了。

```!
这一小节的内容会有点难度，如果你觉得难以读懂我的文章或者是别的问题，欢迎在下方评论区与我互动！
```

有个例子能更好地帮助理解，我们就通过以下组件的更新来了解整个调和的过程。

```jsx
class Test extends React.Component {
  state = {
    data: [{ key: 1, value: 1 }, { key: 2, value: 2 }]
  };
  componentDidMount() {
    setTimeout(() => {
      const data = [{ key: 0, value: 0 }, { key: 2, value: 2 }]
      this.setState({
        data
      })
    }, 3000);
  }
  render() {
    const { data } = this.state;
    return (
      <>
        { data.map(item => <p key={item.key}>{item.value}</p>) }
      </>
    )
  }
}
```

在前一篇文章中我们了解到了整个更新过程（不包括渲染）就是在反复寻找工作单元并运行它们，那么具体体现到代码中是怎么样的呢？

```js
while (nextUnitOfWork !== null && !shouldYield()) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
}
```

上述代码的 `while` 循环只有当找不到工作单元或者应该打断的时候才会终止。找不到工作单元的情况只有当循环完所有工作单元才会触发，打断的情况是调度器触发的。

当更新任务开始时，`root` **永远**是第一个工作单元，无论之前有没有被打断过工作。

循环寻找工作单元的这个流程其实很简单，就是自顶向下再向上的一个循环。这个循环的规则如下：

1. `root` 永远是第一个工作单元，不管之前有没有被打断过任务
2. 首先判断当前节点是否存在第一个子节点，存在的话它就是下一个工作单元，并让下一个工作节点继续执行该条规则，不存在的话就跳到规则 3
3. 判断当前节点是否存在兄弟节点。如果存在兄弟节点，就回到规则 2，否则跳到规则 4
4. 回到父节点并判断父节点是否存在。如果存在则执行规则 3，否则跳到规则 5
5. 当前工作单元为 null，即为完成整个循环

以下动画是例子代码的工作循环过程的一个示例：

![](https://user-gold-cdn.xitu.io/2019/7/25/16c283e7e105479d?w=480&h=360&f=gif&s=790305)

了解了工作循环流程以后，我们就来深入学习一下工作单元是如何工作的。为了精简流程，我们就直接认为当前的工作单元为 `Test` 组件实例。

在工作单元工作的这一阶段中其实是分为很多分支的，因为涉及到不同类型组件及 DOM 的处理。`Test` 是 `class` 组件，另外这也是最常用的组件类型，因此接下来的内容会着重介绍 `class` 组件的调和过程。

`class` 组件的调和过程大致分为两个部分：

1. 生命周期函数的处理
2. 调和子组件，也就是 diff 算法的过程

### 处理 `class` 组件生命周期函数

最先被处理的生命周期函数是 `componentWillReceiveProps`。

但是触发这个函数的条件有两个：

1. `props` 前后有差别
2. 没有使用 `getDerivedStateFromProps` 或者 `getSnapshotBeforeUpdate` 这两个新的生命周期函数。**使用其一则 `componentWillReceiveProps` 不会被触发**

满足以上条件该函数就会被调用。因此该函数在 React 16 中已经不被建议使用。因为调和阶段是有可能会打断的，因此该函数会重复调用。

```!
凡是在调和阶段被调用的函数基本是不被建议使用的。
```

接下来需要处理 `getDerivedStateFromProps` 函数来获取最新的 `state`。

然后就是判断是否需要更新组件了，这一块的判断逻辑分为两块：

1. 判断是否存在 `shouldComponentUpdate` 函数，存在就调用
2. 不存在上述函数的话，就判断当前组件是否继承自 `PureComponent`。如果是的话，就浅比较前后的 `props` 及 `state` 得出结果

如果得出结论需要更新组件的话，那么就会先调用 `componentWillUpdate` 函数，然后处理 `componentDidUpdate` 及 `getSnapshotBeforeUpdate` 函数。

这里需要注意的是：调和阶段并不会调用以上两个函数，而是打上 tag 以便将来使用位运算知晓是否需要使用它们。`effectTag` 这个属性在整个更新的流程中都是至关重要的一员，凡是涉及到函数的延迟调用、devTool 的处理、DOM 的更新都可能会使用到它。

```js
if (typeof instance.componentDidUpdate === 'function') {
    workInProgress.effectTag |= Update;
}
if (typeof instance.getSnapshotBeforeUpdate === 'function') {
    workInProgress.effectTag |= Snapshot;
}
```

### 调和子组件

处理完生命周期后，就会调用 `render` 函数获取新的 `child`，用于在之后与老的 `child` 进行对比。

在继续学习之前我们先来熟悉三个对象，因为它们在后续的内容中会反复出现：

- returnFiber：父组件。
- currentFirstChild：父组件的第一个 `child`。如果你还记得 fiber 的数据结构的话，应该知道每个 fiber 都有一个 `sibling` 属性指向它的兄弟节点。因此知道第一个子节点就能知道所有的同级节点。
- newChild：也就是我们刚刚 `render` 出来的内容。

首先我们会判断 `newChild` 的类型，知道类型就可以进行相应的 diff 策略了。它可能会是一个 Fragment 类型，也可能是 `object`、`number` 或者 `string` 类型。这几个类型都会有相应的处理，但这不是我们的重点，并且它们的处理也相当简单。

我们的重点会放在可迭代类型上，也就是 `Array` 或者 `Iterator` 类型。这两者的核心逻辑是一致的，因此我们就只讲对 `Array` 类型的处理了。

以下内容是对于 diff 算法的详解，虽然有三次 `for` 循环，但是本质上只是遍历了一次整个 `newChild`。

#### 正餐开始，第一轮遍历

第一轮遍历的核心逻辑是复用和当前节点索引一致的老节点，一旦出现不能复用的情况就跳出遍历。

那么如何复用之前的节点呢？规则如下：

- 新旧节点都为文本节点，可以直接复用，因为文本节点不需要 key
- 其他类型节点一律通过判断 key 是否相同来复用或创建节点（可能类型不同但 key 相同）

以下是我简化后的第一轮遍历代码：

```js
for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
  // 找到下一个老的子节点
  nextOldFiber = oldFiber.sibling;
  // 通过 oldFiber 和 newChildren[newIdx] 判断是否可以复用
  // 并给复用出来的节点的 return 属性赋值 returnFiber
  const newFiber = reuse(
    returnFiber,
    oldFiber,
    newChildren[newIdx]
  );
  // 不能复用，跳出
  if (newFiber === null) {
    break;
  }
}
```

那么回到上文中的例子中，我们老的第一个节点的 key 为 1，新的节点的 key 为 0。key 不相同不能复用，因此直接跳出循环，此时 `newIdx` 仍 为 0。

#### 第二轮遍历

当第一轮遍历结束后，会出现两种情况：

- `newChild` 已经遍历完
- 老的节点已经遍历完了

当出现 `newChild` 已经遍历完的情况时只需要把所有剩余的老节点都删除即可。删除的逻辑也就是设置 `effectTag` 为 `Deletion`，另外还有几个 fiber 节点属性需要提及下。

当出现需要在渲染阶段进行处理的节点时，会把这些节点放入父节点的 `effect` 链表中，比如需要被删除的节点就会把加入进链表。这个链表的作用是可以帮助我们在渲染阶段迅速找到需要更新的节点。

当出现老的节点已经遍历完了的情况时，就会开始第二轮遍历。这轮遍历的逻辑很简单，只需要把剩余新的节点全部创建完毕即可。

这轮遍历在我们的例子中是不会执行的，因为我们以上两种情况都不符合。

#### 第三轮遍历

第三轮遍历的核心逻辑是找出可以复用的老节点并移动位置，不能复用的话就只能创建一个新的了。

那么问题又再次回到了如何复用节点并移动位置上。首先我们会把所有剩余的老节点都丢到一个 `map` 中。

我们例子中的代码剩余的老节点为：

```html
<p key={1}>1</p>
<p key={2}>2</p>
```

那么这个 `map` 的结构就会是这样的：

```js
// 节点的 key 作为 map 的 key
// 如果节点不存在 key，那么 index 为 key
const map = {
    1: {},
    2: {}
}
```

在遍历的过程中会寻找新的节点的 key 是否存在于这个 `map` 中，存在即可复用，不存在就只能创建一个新的了。其实这部分的复用及创建的逻辑和第一轮中的是一模一样的，所以也就不再赘述了。

那么如果复用成功，就应该把复用的 `key` 从 `map` 中删掉，并且给复用的节点移动位置。这里的移动依旧不涉及 DOM 操作，而是给 `effectTag` 赋值为 `Placement`。

此轮遍历结束后，就把还存在于 `map` 中的所有老节点删除。

#### 小结

以上就是 diff 子节点的全部逻辑，对比 React 15 的 diff 策略而言个人认为代码好懂了许多。

## 最后

阅读源码是一个很枯燥的过程，但是收益也是巨大的。如果你在阅读的过程中有任何的问题，都欢迎你在评论区与我交流。

另外写这系列是个很耗时的工程，需要维护代码注释，还得把文章写得尽量让读者看懂，最后还得配上画图，如果你觉得文章看着还行，就请不要吝啬你的点赞。

最后，觉得内容有帮助可以加群一同交流与学习。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-034140.png)
