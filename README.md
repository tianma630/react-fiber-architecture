翻译文章：https://github.com/acdlite/react-fiber-architecture

# react fiber 架构

## 介绍
 `react fiber` 重新实现了 `react` 的核心算法。他是 `react` 团队花了超过2年时间研究的成果。

 `react fiber` 的目标是为了更好的适配动画、布局、手势操作等领域。它最大的特征是**渐进式的渲染**：将渲染工作拆分成若干的小模块，并插入到不同的帧中。

 其他的特征包括：当有新的更新时，可以暂停、终止、重复使用工作模块；为不同的更新分配优先级；和并发的能力。

 ## 关于文档
 `fiber` 介绍了几个原则，如果只看代码很难去理解。这个文档是我随着 react 项目实现 `fiber` 的过程中收集的一些笔记。随着文档的扩充，我渐渐意识到这也许会帮助到其他人。

 我会尝试使用最朴素的语言，尽量避免行业术语。如果有需要我也会直接链接到外部的资源。

 我并不是 `react` 团队的组员，也不代表任何官方。**这也不是正式的文档**。当然，为了更加的准确，我也邀请了 `react` 团队的成员来帮忙检查这个文档。

 文档也还在进行当中。**fiber 是一个进行当中的项目，在它完成之前会经历一些大的重构**。我也尝试用文档记录这个过程。很欢迎大家的改进和建议。

 我希望大家读了这篇文档后，足够的理解 `fiber` ，[跟上它的实现](https://github.com/facebook/react/commits/master/src/renderers/shared/fiber)，最终甚至对 react 有所贡献。

 ### 预备知识
 我强烈建议你在继续之前先熟悉下面的内容：
 - [React Components, Elements, and instances](https://facebook.github.io/react/blog/2015/12/18/react-components-elements-and-instances.html) - 组件常常有多种含义。深刻掌握这几个概念非常重要。
 - [Reconciliation](https://facebook.github.io/react/docs/reconciliation.html) - 对 react 调度算法高层次的描述。
 - [React Basic Theoretical Concepts](https://github.com/reactjs/react-basic) - 描述了 react 概念模型。第一次阅读可能理解不了其中的一些原则。没有关系，随着时间的进行会有更好的理解。
 - [React Design Principles](https://facebook.github.io/react/contributing/design-principles.html) - 特别关注调度的部分。它解释了为什么是 react fiber 。

 ## 检查
 如果还没有的话，请确认下上面的预备知识。

 在深入之前，我们检查下一些原则。

 ### 调和算法(reconciliation)是什么

<dl>
  <dt>调和(reconciliation)</dt>
  <dd>
  调和算法是 `react` 用来区分新、旧两棵树，并找出需要更新的部分。
  </dd>
  <dt>更新(update)</dt>
  <dd>
    数据发生变化，会重新渲染 react app 。通常是 `setState` 导致的。最终的结果是重新渲染。
  </dd>
</dl>

`react` 的 `api` 的中心思想是关系更新，好像他们会引起这个应用的重新渲染。这允许开发人员以声明的方式进行推理，而不是担心如何有效率转换 app 从一个特定的状态到另一个。

事实上，每次更新时重新渲染整个应用只适用于大多数小型的应用。在实际中，这个很耗费性能。`react` 有很多的优化，在保持出色的性能的技术上实现整个应用的重新渲染。大部分的优化是调和算法的一部分。

调和算法建立在通常理解的 "虚拟dom" 的基础上。一个高层次的描述是这样的：当你渲染一个 react 应用时，用于描述 app 如何创建和存储的节点树。然后，节点树会更新到渲染环境。举个例子，一个浏览器应用，翻译成 DOM 操作的集合。当应用更新时，一颗新的树被创建。新的树和老的树进行对比，计算出更新应用需要哪些操作。

尽管 `fiber` 重写了调和，但是 `react` 文档中描述的高级算法大致上是相同的。下面是关键点：

- 不同的组件类型生成不同的树。react 将不会比对他们，而是完全替换这个老的树。
- 比对时会用到 keys。keys 必须是 "稳定的、可预言的、唯一的"。

### 调和(reconciliation) vs 渲染(rendering)

`DOM` 只是 `react` 能渲染的环境的一种，其他主要的包括能渲染 `native` 环境的 `react native`。(这就是为什么 "虚拟DOM" 有点用词不当)

`react` 将调度和渲染分成了2个独立的阶段，所以 `react` 能支持多种的环境。调和器用于计算那部分的树发生了变化；而渲染器用于将变化的信息更新到应用中。

这样的话，`React DOM` 和 `React Native` 可以共享调和器，而使用各自不同的渲染器。

`fiber` 重新实现了调和器。原则上和渲染无关，但是渲染器还是需要去适配新的架构。

### 调度(scheduling)

<dl>
  <dt>调度(scheduling)</dt>
  <dd>
  决定了工作什么时候被执行的工程。
  </dd>
  <dt>工作(work)</dt>
  <dd>
    任何的计算都必须被执行。工作通常是一个更新的结果。(setState)
  </dd>
</dl>

react 的[设计原则](https://facebook.github.io/react/contributing/design-principles.html#scheduling)文档很适合这个主题：
> 在当前的实现中，`react` 在一个事件片期间递归遍历树，并调用整颗更新树的渲染函数。但是，在未来为了避免阻塞贞执行，会延迟执行部分的更新。
>
> 一些流行的库实现了 "push" 方法，即在新数据可用时执行计算。然而，
`React`坚持 "pull" 方法，在这种方法中，计算可以延迟到必要时进行。
>
> `react` 并不是数据处理库。它是用来创建用户界面。我们认为，它被独立的放置在一个应用中，用于区分那个计算是有意义哪个是没有意义的。
>
> 如果元素不在屏幕中，我们可以延迟和它相关的执行逻辑。如果数据到达的比帧率还快，我们可以合并数据批量执行。我们可以有效执行用户界面相关的工作
(比如按钮点击动画)，再执行不太重要的后台工作(比如渲染网络请求的内容)。

关键点如下：
- 在一个 UI 中，不是每个更新都必须立即执行；事实上，这样做会很浪费，造成延时，降低用户体验。
- 不同类型的更新有不同的优先级 -- 动画必须比数据存储更快的执行。
- 一个基于 'push' 的方法需要应用(开发者)决定怎么调度工作。一个基于 'pull' 的方法允许框架帮你做这些决定。

现在我们准备好深入 `fiber` 的实现。下一部分会比目前讨论的更加的技术性。请确认你已经很好地理解了之前的内容。

---

## fiber 是什么?

我们马上要讨论的 `react fiber` 架构的核心。`fiber` 要比普通的应用开发者想象的更加低层级的抽象。如果你再尝试理解它的时候遇到了麻烦，别灰心。继续尝试，最终一定会有所理解。

来我们开始吧！

---

我们创建了 fiber 的一个基础目标，利用调度。特别我们需要能够做到：

- 暂停工作，稍后重启
- 给不同类型的工作设置优先级
- 复用之前的已经完成的工作
- 中止不需要的工作

为了实现任意一个能力，我们首先需要把工作拆分成小的单元。在某种意义上，这就是 fiber。一个 fiber 代表一个工作单元。

为了更进一步，我们回顾一下 [React components as functions of data](https://github.com/reactjs/react-basic#transformation) 的概念，通常表达为：
```
v = f(d)
```
也就是说，渲染一个 react 应用类似调用一个函数，函数体包含了调用其他函数，等等。这个例子有利于理解 `fiber`。

通常，计算机跟踪程序运行过程的方法是使用调用栈(call stack)。一个函数执行时，一个新的**栈帧**(stack frame)被添加到栈中。栈帧表示函数执行的内容。

当处理 UI 的时候，当有太多的工作同时需要处理，它会导致动画阻塞帧(frame)，看起来断断续续。而且，如果他被一个最近的更新代替时，有些工作可能不太需要。结果当 UI 组件和函数比对时会发生故障，因为组件通常比函数有更多的关注点。

一些新的浏览器实现了新的 api 可以帮助解决这个问题：`requestIdleCallback` 调度一些低优先级的函数在空闲时间执行，`requestAnimationFrame` 调度一些高优先级的函数在下一次动画帧时执行。问题是，为了使用这些 api，你需要把把渲染工作拆分成增量的单元。如果你仅依赖于调用栈，它会保持执行知道调用栈为空。

如果我们能通过定制调用栈的行为优化 UI 渲染；如果我们能够完全中断调用栈，人工操作栈帧；那就太棒了？

这是 `react fiber` 的目标。`fiber` 是栈的重新实现，特别为 `react` 组件。你可以把一个 `fiber` 当做一个虚拟的栈帧。

重新实现栈的优势是，你可以[把栈帧进行缓存](https://www.facebook.com/groups/2003630259862046/permalink/2054053404819731/)，在任何时候执行。实现整个目标对调度来说非常重要。

抛开调度，人工处理栈帧解锁了很多特性，比如并发和错误边界。我会在未来的章节覆盖这些主题。

下一章节，我们会更加关注与 `fiber` 的结构。

### fiber 的结构(Structure of a fiber)

*注意：因为我们获取更多关于实现细节的信息，有些内容变化的可能性会变大。如果你注意到任何错误或过期的信息，请创建一个PR。*

具体来说，一个 `fiber` 是一个包含了组件信息的 `JavaScript` 对象，它的输入、输出。

一个 `fiber` 对应一个栈帧，但是它也对应一个组件的实例。

下面是一些 `fiber` 的重要字段。(列表并不详细)

#### `type` and `key`

`fiber` 的 `type` 和 `key` 的作用和他们在 `react element` 中起到的作用是一样的。(事实上，当 fiber 从 element 被创建时，这2个字段就直接被拷贝过去了)

一个 `fiber` 的 `type` 描述了对应的组件。对合成组件，`type` 是函数或类组件本身。对宿主组件(div span 等)，`type` 是字符串。

概念上，`type` 是一个函数，执行过程中会被栈帧追踪。

连同 `type`，`key` 会在调和过程中用来决定 `fiber` 是否可以复用。

#### `child` and `sibling`

这2个字段指向其他的 `fiber`，描述了 `fiber` 的递归树结构。

孩子 `fiber` 对应了组件的 `render` 方法返回的值。在下面的例子中

```
function Parent() {
  return <Child />
}
```
`Parent` 的孩子 `fiber` 指向 `Child`。

`sibling` 字段以下面 `render` 方法返回多个孩子为例(fiber 中新的特性):

```
function Parent() {
  return [<Child1 />, <Child2 />]
}
```

孩子 fiber 指向列表的第一个孩子。在这个例子中，`Parent` 的孩子 `fiber` 就是 `Child1`，`Child1` 的 `sibling` 就是 `Child2`。

返回到我们的函数示例，你可以把孩子 `fiber` 认为是一个[最后调用的函数](https://en.wikipedia.org/wiki/Tail_call)

#### `return`

`return fiber` 是指当前 `fiber` 处理完成后返回的 `fiber`。概念上，它相当于一个栈帧的返回地址。也可以认为是父亲 `fiber`。

如果一个 `fiber` 有多个孩子 `fiber`，各个 `fiber` 的 `return fiber` 就是父亲 `fiber`。因此在之前的例子中，`Child1` 和 `Child2` 的 `return fiber` 就是 `Parent`。

#### `pendingProps` and `memoizedProps`

概念上，`props` 就是函数的参数。`fiber` 的 `pendingProps` 是在执行开始时被设置，`memoizedProps` 实在执行结束时被设置。

当输入的 `pendingProps` 和 `memoizedProps` 相同时，就表示之前 `fiber` 的输出可以被复用，避免不需要的工作。

#### `pendingWorkPriority`

标识了一个 fiber 工作的优先级。[ReactPriorityLevel](https://github.com/facebook/react/blob/master/src/renderers/shared/fiber/ReactPriorityLevel.js) 模块枚举了不同的优先级，以及他们代表的意思。

除了 `NoWork` 的优先级是0以外，一个更大的数字标识了更低的优先级。比如，你可以用下面的函数检查一个 fiber 的优先级是不是比给定的优先级高。

```
function matchesPriority(fiber, priority) {
  return fiber.pendingWorkPriority !== 0 &&
         fiber.pendingWorkPriority <= priority
}
```

这个函数只是个示例，并不是 `react fiber` 代码的一部分。

调度器使用优先级字段去查询下一个执行的工作单元。这个算法会在未来的章节进行讨论。


#### `alternate`

<dl>
  <dt>flush</dt>
  <dd>刷新一个 fiber 表示渲染 fiber 的输出到屏幕上</dd>

  <dt>work-in-progress</dt>
  <dd>一个还没被完成的 fiber；概念上，一个还没返回的栈帧。</dd>
</dl>

任何时候，一个组件实例有至多2个 `fiber` 对应它：当前的刷新后的 `fiber`，工作中的 `fiber`。

当前 `fiber` 的 `alternate` 就是 `work-in-progress`，而 `work-in-progress` 的 `alternate` 就是当前 fiber。

`fiber` 的 `alternate` 有一个叫 `cloneFiber` 的函数创建。并不是每次都创建一个新的对象，如果 `fiber` 的 `alternate` 存在的话，`cloneFiber` 会尝试复用它，最小化内存消耗。

你可以把 `alternate` 认为是一个实现详情，但是它经常出现在代码中，所有有必要在这里讨论一下。

#### `output`

<dl>
  <dt>宿主组件(host component)</dt>
  <dd>react 应用的叶子节点。他们对应渲染环境(比如在浏览器应用中，他们就是 div span 等。)</dd>
</dl>

概念上，一个 `fiber` 的输出是一个函数的返回值。

每个 `fiber` 都有输出，但是输出只会在叶子节点被宿主组件创建。然后输出转移到树上。

输出最终给到渲染器，它会刷新变化到渲染环境。渲染器需要定义输出的创建和更新。

## 未来章节
这就是全部，但是文档并没有完成。未来的章节会描述整个更新生命周期中的算法。主题包括：

- 调度器如何找到下一个工作单元。
- 优先级如何在 `fiber` 树种被追踪和传播。
- 调度器如何知道什么时候暂停和恢复工作。
- 工作如何被刷新和标记结束。
- 副作用(比如生命周期方法)如何工作。
- 什么是协同，它是如何被用来实现 `context` 和 `layout` 等特性。

## 关联适配
- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)

