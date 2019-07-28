# 剖析 React 源码：render 流程（二）

这是我的剖析 React 源码的第三篇文章，如果你没有阅读过之前的文章，请务必先阅读一下 [第一篇文章](https://github.com/KieSun/Dream/issues/18) 中提到的一些注意事项，能帮助你更好地阅读源码。

## 文章相关资料

- [React 16.8.6 源码中文注释](https://github.com/KieSun/react-interpretation)，这个链接是文章的核心，文中的具体代码及代码行数都是依托于这个仓库
- [热身篇](https://github.com/KieSun/Dream/issues/18)
- [render 流程（一）](https://github.com/KieSun/Dream/issues/19)

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-031952.png)

此篇文章内容衔接 [render 流程（一）](https://github.com/KieSun/Dream/issues/19)，当然不看上一篇文章也没什么问题，因为内容并没有强相关。

现在请大家打开 [我的代码](https://github.com/KieSun/react-interpretation) 并定位到 react-dom 文件夹下的 src 中的 ReactDOM.js 文件，今天的内容会从这里开始。

## ReactRoot.prototype.render

在上一篇文章中，我们介绍了当 `ReactDom.render` 执行时，内部会首先判断是否已经存在 `root`，没有的话会去创建一个 `root`。在今天的文章中，我们将会了解到存在 `root` 以后会发生什么事情。

大家可以先定位到代码的第 592 行。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-031954.png)

大家可以看到，在上述的代码中调用了 `unbatchedUpdates` 函数，这个函数涉及到的知识其实在 React 中相当重要。

大家都知道多个 `setState` 一起执行，并不会触发 React 的多次渲染。

```js
// 虽然 age 会变成 3，但不会触发 3 次渲染
this.setState({ age: 1 })
this.setState({ age: 2 })
this.setState({ age: 3 })
```

这是因为内部会将这个三次 `setState` 优化为一次更新，术语是批量更新（batchedUpdate），我们在后续的内容中也能看到内部是如何处理批量更新的。

对于 root 来说其实没必要去批量更新，所以这里调用了 `unbatchedUpdates` 函数来告知内部不需要批量更新。

然后在 `unbatchedUpdates` 回调内部判断是否存在 `parentComponent`。这一步我们可以假定不会存在 `parentComponent`，因为很少有人会在 `root` 外部加上 `context` 组件。不存在 `parentComponent` 的话就会执行 `root.render(children, callback)`，这里的 `render` 指的是 `ReactRoot.prototype.render`。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-031956.png)

在 `render` 函数内部我们首先取出 `root`，这里的 `root` 指的是 FiberRoot，如果你想了解 FiberRoot 相关的内容可以阅读 [上一篇文章](https://github.com/KieSun/Dream/issues/19)。然后创建了 `ReactWork` 的实例，这块内容我们没有必要深究，功能就是为了在组件渲染或更新后把所有传入 `ReactDom.render` 中的回调函数全部执行一遍。

接下来我们来看 `updateContainer` 内部是怎么样的。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-031958.png)

我们先从 FiberRoot 的 `current` 属性中取出它的 fiber 对象，然后计算了两个时间。这两个时间在 React 中相当重要，因此我们需要单独用一小节去学习它们。

## 时间

首先是 `currentTime`，在 `requestCurrentTime` 函数内部计算时间的最核心函数是 `recomputeCurrentRendererTime`。

```js
function recomputeCurrentRendererTime() {
	const currentTimeMs = now() - originalStartTimeMs;
	currentRendererTime = msToExpirationTime(currentTimeMs);
}
```

`now()` 就是 `performance.now()`，如果你不了解这个 API 的话可以阅读下 [相关文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Performance/now)，`originalStartTimeMs` 是 React 应用初始化时就会生成的一个变量，值也是 `performance.now()`，并且这个值不会在后期再被改变。那么这两个值相减以后，得到的结果也就是现在离 React 应用初始化时经过了多少时间。

然后我们需要把计算出来的值再通过一个公式算一遍，这里的 `| 0` 作用是取整数，也就是说 `11 / 10 | 0 = 1`

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-031959.png)

接下来我们来假定一些变量值，代入公式来算的话会更方便大家理解。

假如 `originalStartTimeMs` 为 `2500`，当前时间为 `5000`，那么算出来的差值就是 `2500`，也就是说当前距离 React 应用初始化已经过去了 2500 毫秒，最后通过公式得出的结果为：

```js
currentTime = 1073741822 - ((2500 / 10) | 0) = 1073741572
```

接下来是计算 `expirationTime`，**这个时间和优先级有关，值越大，优先级越高**。并且同步是优先级最高的，它的值为 `1073741823`，也就是之前我们看到的常量 `MAGIC_NUMBER_OFFSET` 加一。

在 `computeExpirationForFiber` 函数中存在很多分支，但是计算的核心就只有三行代码，分别是：

```js
// 同步
expirationTime = Sync
// 交互事件，优先级较高
expirationTime = computeInteractiveExpiration(currentTime)
// 异步，优先级较低
expirationTime = computeAsyncExpiration(currentTime)
```

接下来我们就来分析 `computeInteractiveExpiration` 函数内部是如何计算时间的，当然 `computeAsyncExpiration` 计算时间的方式也是相同的，无非更换了两个变量。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032001.png)

以上这些代码其实就是公式，我们把具体的值代入就能算出结果了。

```js
time = 1073741822 - ((((1073741822 - 1073741572 + 15) / 10) | 0) + 1) * 10 = 1073741552
```

另外在 `ceiling` 函数中的 `1 * bucketSizeMs / UNIT_SIZE` 是为了抹平一段时间内的时间差，在抹平的时间差内不管有多少个任务需要执行，他们的过期时间都是同一个，这也算是一个性能优化，帮助渲染页面行为节流。

最后其实我们这个计算出来的 `expirationTime` 是可以反推出另外一个时间的：

```js
export function expirationTimeToMs(expirationTime: ExpirationTime): number {
	return (MAGIC_NUMBER_OFFSET - expirationTime) * UNIT_SIZE;
}
```

如果我们将之前计算出来的 `expirationTime` 代入以上代码，得出的结果如下：

```js
(1073741822 - 1073741552) * 10 = 2700
```

这个时间其实和我们之前在上文中计算出来的 `2500` 毫秒差值很接近。因为 `expirationTime` 指的就是一个任务的过期时间，React 根据任务的优先级和当前时间来计算出一个任务的执行截止时间。只要这个值比当前时间大就可以一直让 React 延后这个任务的执行，以便让更高优先级的任务执行，但是一旦过了任务的截止时间，就必须让这个任务马上执行。

这部分的内容一直在算来算去，看起来可能有点头疼。当然如果你嫌麻烦，只需要记住任务的过期时间是通过当前时间加上一个常量（任务优先级不同常量不同）计算出来的。

另外其实你还可以在后面的代码中看到更加直观且简单的计算过期时间的方式，但是目前那部分代码还没有被使用起来。

## scheduleRootUpdate

当我们计算出时间以后就会调用 `updateContainerAtExpirationTime`，这个函数其实没有什么好解析的，我们直接进入 `scheduleRootUpdate` 函数就好。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032002.png)

首先我们会创建一个 `update`，**这个对象和 `setState` 息息相关**

```js
// update 对象的内部属性
expirationTime: expirationTime,
tag: UpdateState,
// setState 的第一二个参数
payload: null,
callback: null,
// 用于在队列中找到下一个节点
next: null,
nextEffect: null,
```

对于 `update` 对象内部的属性来说，我们需要重点关注的是 `next` 属性。因为 `update` 其实就是一个队列中的节点，这个属性可以用于帮助我们寻找下一个 `update`。对于批量更新来说，我们可能会创建多个 `update`，因此我们需要将这些 `update` 串联并存储起来，在必要的时候拿出来用于更新 `state`。

在 `render` 的过程中其实也是一次更新的操作，但是我们并没有 `setState`，因此就把 `payload` 赋值为 `{element}` 了。

接下来我们将 `callback` 赋值给 `update` 的属性，这里的 `callback` 还是 `ReactDom.render` 的第三个参数。

然后我们将刚才创建出来的 `update` 对象插入队列中，`enqueueUpdate` 函数内部分支较多且代码简单，这里就不再贴出代码了，有兴趣的可以自行阅读。函数核心作用就是创建或者获取一个队列，然后把 `update` 对象入队。

最后调用 `scheduleWork` 函数，这里开始就是调度相关的内容，这部分内容我们将在下一篇文章中来详细解析。

## 总结

以上就是本文的全部内容了，这篇文章其实核心还是放在了计算时间上，因为这个时间和后面的调度息息相关，最后通过一张流程图总结一下 render 流程两篇文章的内容。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032003.png)

## 最后

阅读源码是一个很枯燥的过程，但是收益也是巨大的。如果你在阅读的过程中有任何的问题，都欢迎你在评论区与我交流。

另外写这系列是个很耗时的工程，需要维护代码注释，还得把文章写得尽量让读者看懂，最后还得配上画图，如果你觉得文章看着还行，就请不要吝啬你的点赞。

下一篇文章还是 render 流程相关的内容。

最后，觉得内容有帮助可以关注下我的公众号 「前端真好玩」咯，会有很多好东西等着你。

![](https://yck-1254263422.cos.ap-shanghai.myqcloud.com/blog/2019-06-01-032004.jpg)
