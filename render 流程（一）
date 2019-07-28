# 剖析 React 源码：render 流程（一）

这是我的剖析 React 源码的第二篇文章，如果你没有阅读过之前的文章，请务必先阅读一下 [第一篇文章](https://github.com/KieSun/Dream/issues/18) 中提到的一些注意事项，能帮助你更好地阅读源码。

## 文章相关资料

- [React 16.8.6 源码中文注释](https://github.com/KieSun/react-interpretation)，这个链接是文章的核心，文中的具体代码及代码行数都是依托于这个仓库
- [热身篇](https://github.com/KieSun/Dream/issues/18)
- [render 流程（二）](https://github.com/KieSun/Dream/issues/20)

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032238.png)

现在请大家打开 [我的代码](https://github.com/KieSun/react-interpretation) 并定位到 react-dom 文件夹下的 src 中的 ReactDOM.js 文件，今天的内容会从这里开始。

## render

想必大家在写 React 项目的时候都写过类似的代码 

```js
ReactDOM.render(<APP />, document.getElementById('root')
```

这句代码告诉了 React 应用我们想在容器中渲染出一个组件，这通常也是一个 React 应用的入口代码，接下来我们就来梳理整个 `render` 的流程，并且会分为几篇文章来讲解，因为流程实在太长了。

首先请大家先定位到 ReactDOM.js 文件的第 702 行代码，开始今天的旅程。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032240.png)

这部分代码其实没啥好说的，唯一需要注意的是在调用 `legacyRenderSubtreeIntoContainer` 函数时写死了第四个参数 `forceHydrate` 为 `false`。这个参数为 `true` 时表明了是服务端渲染，因为我们分析的是客户端渲染，因此后面有关这部分的内容也不会再展开。

接下来进入 `legacyRenderSubtreeIntoContainer` 函数中，这部分代码分为两块来讲。第一部分是没有 `root` 之前我们首先需要创建一个 `root`（对应这篇文章），第二部分是有 `root` 之后的渲染流程（对应接下来的文章）。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032241.png)

一开始进来函数的时候肯定是没有 `root` 的，因此我们需要去创建一个 `root`，大家可以发现这个 `root` 对象同样也被挂载在了 `container._reactRootContainer` 上，也就是我们的 DOM 容器上。
如果你手边有 React 项目的话，在控制台键入如下代码就可以看到这个 `root` 对象了。

```js
document.querySelector('#root')._reactRootContainer
```

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032244.png)

大家可以看到 `root` 是 `ReactRoot` 构造函数构造出来的，并且内部有一个 `_internalRoot` 对象，这个对象是本文接下来要重点介绍的 `fiber` 对象，接下来我们就来一窥究竟吧。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032245.png)

首先还是和上文中提到的 `forceHydrate` 属性相关的内容，不需要管这部分，反正 `shouldHydrate` 肯定为 `false`。

接下来是将容器内部的节点全部移除，一般来说我们都是这样写一个容器的的

```html
<div id='root'></div>
```

这样的形式肯定就不需要去移除子节点了，这也侧面说明了一点那就是容器内部不要含有任何的子节点。一是肯定会被移除掉，二来还要进行 DOM 操作，可能还会涉及到重绘回流等等。

最后就是创建了一个 `ReactRoot` 对象并返回。接下来的内容中我们会看到好几个 `root`，可能会有点绕。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032247.png)

在 `ReactRoot` 构造函数内部就进行了一步操作，那就是创建了一个 `FiberRoot` 对象，并挂载到了 `_internalRoot` 上。**和 DOM 树一样，`fiber` 也会构建出一个树结构（每个 DOM 节点一定对应着一个 `fiber` 对象），`FiberRoot` 就是整个 `fiber` 树的根节点**，接下来的内容里我们将学习到关于 `fiber` 相关的内容。这里提及一点，`fiber` 和 Fiber 是两个不一样的东西，前者代表着数据结构，后者代表着新的架构。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032249.png)

在 `createFiberRoot` 函数内部，分别创建了两个 `root`，一个 `root` 叫做 `FiberRoot`，另一个 `root` 叫做 `RootFiber`，并且它们两者还是相互引用的。

这两个对象内部拥有着数十个属性，现在我们没有必要一一去了解它们各自有什么用处，在当下只需要了解少部分属性即可，其他的属性我们会在以后的文章中了解到它们的用处。

对于 `FiberRoot` 对象来说，我们现在只需要了解两个属性，分别是 `containerInfo` 及 `current`。前者代表着容器信息，也就是我们的 `document.querySelector('#root')`；后者指向 `RootFiber`。

对于 `RootFiber` 对象来说，我们需要了解的属性稍微多点

```js
function FiberNode(
	tag: WorkTag,
	pendingProps: mixed,
	key: null | string,
	mode: TypeOfMode,
) {
	this.stateNode = null;
	this.return = null;
	this.child = null;
	this.sibling = null;
	this.effectTag = NoEffect;
	this.alternate = null;
}
```

`stateNode` 上文中已经讲过了，这里就不再赘述。

`return`、`child`、`sibling` 这三个属性很重要，它们是构成 `fiber` 树的主体数据结构。`fiber` 树其实是一个单链表树结构，`return` 及 `child` 分别对应着树的父子节点，并且父节点只有一个 `child` 指向它的第一个子节点，即便是父节点有好多个子节点。那么多个子节点如何连接起来呢？答案是 `sibling`，每个子节点都有一个 `sibling` 属性指向着下一个子节点，都有一个 `return` 属性指向着父节点。这么说可能有点绕，我们通过图来了解一下这个 `fiber` 树的结构。

```js
const APP = () => (
		<div>
				<span></span>
				<span></span>
		</div>
)
ReactDom.render(<APP/>, document.querySelector('#root'))
```

假如说我们需要渲染出以上组件，那么它们对应的 `fiber` 树应该长这样

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-32250.png)

从图中我们可以看到，每个组件或者 DOM 节点都会对应着一个 `fiber` 对象。另外你手边有 React 项目的话，也可以在控制台输入如下代码，查看 `fiber` 树的整个结构。

```js
// 对应着 FiberRoot
const fiber = document.querySelector('#root')._reactRootContainer._internalRoot
```

另外两个属性在本文中虽然用不上，但是看源码的时候笔者觉得很有意思，就打算拿出来说一下。

在说 `effectTag` 之前，我们先来了解下啥是 `effect`，简单来说就是 DOM 的一些操作，比如增删改，那么 `effectTag` 就是来记录所有的 effect 的，但是这个记录是通过位运算来实现的，[这里](https://github.com/facebook/react/blob/master/packages/shared/ReactSideEffectTags.js) 是 `effectTag` 相关的二进制内容。

如果我们想新增一个 `effect` 的话，可以这样写 `effectTag |= Update`；如果我们想删除一个 `effect` 的话，可以这样写 `effectTag &= ~Update`。

最后是 `alternate` 属性。其实在一个 React 应用中，通常来说都有两个 `fiebr` 树，一个叫做 old tree，另一个叫做 workInProgress tree。前者对应着已经渲染好的 DOM 树，后者是正在执行更新中的 fiber tree，还能便于中断后恢复。两棵树的节点互相引用，便于共享一些内部的属性，减少内存的开销。毕竟前文说过每个组件或 DOM 都会对应着一个 `fiber` 对象，应用很大的话组成的 `fiber` 树也会很大，如果两棵树都是各自把一些相同的属性创建一遍的话，会损失不少的内存空间及性能。

当更新结束以后，workInProgress tree 会将 old tree 替换掉，这种做法称之为 double buffering，这也是性能优化里的一种做法，有兴趣的同学可以自行查找资料。

## 总结

以上就是本文的全部内容了，最后通过一张流程图总结一下这篇文章的内容。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032252.png)

## 最后

阅读源码是一个很枯燥的过程，但是收益也是巨大的。如果你在阅读的过程中有任何的问题，都欢迎你在评论区与我交流。

另外写这系列是个很耗时的工程，需要维护代码注释，还得把文章写得尽量让读者看懂，最后还得配上画图，如果你觉得文章看着还行，就请不要吝啬你的点赞。

下一篇文章还是 render 流程相关的内容。

最后，觉得内容有帮助可以关注下我的公众号 「前端真好玩」咯，会有很多好东西等着你。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032253.jpg)
