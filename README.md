翻译自文章：https://github.com/acdlite/react-fiber-architecture

# react fiber 架构

## 介绍
 react fiber 重新实现了 react 的核心算法。他是 react 团队花了超过2年时间研究的成果。

 react fiber 的目标是为了更好的适配动画、布局、手势操作等领域。它最大的特征是**渐进式的渲染**：将渲染工作拆分成若干的小模块，并插入到不同的帧中。

 其他的特征包括：当有新的更新时，可以暂停、终止、重复使用工作模块；为不同的更新分配优先级；和并发的能力。

 ## 关于文档
 fiber 介绍了几个原则，如果只看代码很难去理解。这个文档是我随着 react 项目实现 fiber 的过程中收集的一些笔记。随着文档的扩充，我渐渐意识到这也许会帮助到其他人。

 我会尝试使用最朴素的语言，尽量避免行业术语。如果有需要我也会直接链接到外部的资源。

 我并不是 react 团队的组员，也不代表任何官方。**这也不是正式的文档**。当然，为了更加的准确，我也邀请了 react 团队的成员来帮忙检查这个文档。

 文档也还在进行当中。**fiber 是一个进行当中的项目，在它完成之前会经历一些大的重构**。我也尝试用文档记录这个过程。很欢迎大家的改进和建议。

 我希望大家读了这篇文档后，足够的理解 fiber ，[跟上它的实现](https://github.com/facebook/react/commits/master/src/renderers/shared/fiber)，最终甚至对 react 有所贡献。

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
  调和算法是 react 用来区分新、旧两棵树，并找出需要更新的部分。
  </dd>
  <dt>更新(update)</dt>
  <dd>
    数据发生变化，会重新渲染 react app 。通常是 `setState` 导致的。最终的结果是重新渲染。
  </dd>
</dl>

react 的 api 的中心思想是关系更新，好像他们会引起这个 app 的重新渲染。这允许开发人员以声明的方式进行推理，而不是担心如何有效率转换 app 从一个特定的状态到另一个。

事实上，每次更新时重新渲染整个应用只适用于大多数小型的 app。在实际中，这个很耗费性能。react 有很多的优化，在保持出色的性能的技术上实现整个应用的重新渲染。大部分的优化是调和算法的一部分。

调和算法建立在通常理解的 "虚拟dom" 的基础上。一个高层次的描述是这样的：当你渲染一个 react 应用时，用于描述 app 如何创建和存储的节点树。然后，节点树会更新到渲染环境。举个例子，一个浏览器应用，翻译成 DOM 操作的集合。当应用更新时，一颗新的树被创建。新的树和老的树进行对比，计算出更新应用需要哪些操作。

尽管 fiber 重写了调和，但是 react 文档中描述的高级算法大致上是相同的。下面是关键点：

- 不同的组件类型生成不同的树。react 将不会比对他们，而是完全替换这个老的树。
- 比对时会用到 keys。keys 必须是 "稳定的、可预言的、唯一的"。

### 调和(reconciliation) vs 渲染(rendering)

DOM 只是 react 能渲染的环境的一种，其他主要的包括能渲染 native 环境的 react native。(这就是为什么 "虚拟DOM" 有点用词不当)

react 将调度和渲染分成了2个独立的阶段，所以 react 能支持多种的环境。调和器用于计算那部分的树发生了变化；而渲染器用于将变化的信息更新到应用中。

这样的话，React DOM 和 React Native 可以共享调和器，而使用各自不同的渲染器。

fiber 重新实现了调和器。原则上和渲染无关，但是渲染器还是需要去适配新的架构。

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
> 在当前的实现中，react 在一个事件片期间递归遍历树，并调用整颗更新树的> 渲染函数。但是，在未来为了避免阻塞贞执行，会延迟执行部分的更新。
>
> 一些流行的库实现了 "push" 方法，即在新数据可用时执行计算。然而，
> React坚持 "pull" 方法，在这种方法中，计算可以延迟到必要时进行。
>
> react并不是数据处理库。它是用来创建用户界面。我们认为，它被独立的放置> 在一个应用中，用于区分那个计算是有意义哪个是没有意义的。
>
> 如果元素不在屏幕中，我们可以延迟和它相关的执行逻辑。如果数据到达的比帧> 率还快，我们可以合并数据批量执行。我们可以有效执行用户界面相关的工作
> (比如按钮点击动画)，再执行不太重要的后台工作(比如渲染网络请求的内容)。

关键点如下：
- 在一个 UI 中，不是每个更新都必须立即执行；事实上，这样做会很浪费，造成延时，降低用户体验。
- 不同类型的更新有不同的优先级 -- 动画必须比数据存储更快的执行。
- 一个基于 'push' 的方法需要应用(开发者)决定怎么调度工作。一个基于 'pull' 的方法允许框架帮你做这些决定。

现在我们准备好深入 fiber 的实现。下一部分会比目前讨论的更加的技术性。请确认你已经很好地理解了之前的内容。

---

## fiber 是什么?

### Structure of a fiber

#### `type` and `key`

#### `child` and `sibling`

#### `return`

#### `pendingProps` and `memoizedProps`

#### `pendingWorkPriority`

#### `alternate`

#### `output`

## Future sections

## Related Videos
- [What's Next for React (ReactNext 2016)](https://youtu.be/aV1271hd9ew)


