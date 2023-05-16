> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [zhuanlan.zhihu.com](https://zhuanlan.zhihu.com/p/608959809)

现在写 react 组件基本都是 function + hooks 了，因为 hooks 很强大也很灵活。

比如 useState 可以声明和修改 state，useEffect 可以管理异步逻辑，useContext 可以读取 context 的值等等，还可以把它们进一步封装成自定义 hooks（自定义 hooks 其实就是普通的函数封装）。

虽然每天都在用 hooks，但依然有很多人不知道 hooks 的实现原理。

所以这篇文章我们就一起来探究下 hooks 的原理吧，主要是讲 useState 和 useEffect 这两个 hook。

首先，我们过一下 react 的渲染流程。

我们组件里用 jsx 描述页面：

![](https://pic4.zhimg.com/v2-8c61988e59864e66d89c2b1ff978de23_r.jpg)

jsx 会被编译成 render function，也就是类似 React.createElement 这种：

![](https://pic2.zhimg.com/v2-3151b9e859aa90290975c197625ab615_r.jpg)

所以之前写 React 组件都必须有一行 import * as React from 'react'，因为编译后会用到 React 的 api。

但后来改为了这种 render function：

![](https://pic1.zhimg.com/v2-70fa8ea32fef55f7cc44484f084198b0_r.jpg)

由 babel、tsc 等编译工具自动引入一个 react/jsx-runtime 的包，

![](https://pic2.zhimg.com/v2-5520897293a9a006401e3256ed846559_r.jpg)

然后 render function 执行后产生 React Element 对象，也就是常说的 vdom。

![](https://pic1.zhimg.com/v2-07009a905cbff30220421d0afc0c640c_r.jpg)

也就是这样的流程：

![](https://pic3.zhimg.com/v2-98f3d06887db6ff053eaf36a68a86242_r.jpg)

然后 vdom 会转换为 fiber 结构，它是一个链表：

![](https://pic1.zhimg.com/v2-d11cd657ba9221951791ed3434372b00_r.jpg)

vdom 只有 children 属性来链接父子节点，但是转为 fiber 结构之后就有了 child、sibling、return 属性来关联父子、兄弟节点。

vdom 转 fiber 的流程叫做 reconcile，我们常说的 diff 算法就是在 reconcile 这个过程中。

多个节点的 diff 也就是当老的 fiber 子节点列表需要更新的时候，要和新的 vdom 的 children 进行对比，找到哪些是可以复用的，直接移动过去，剩下的节点做增删，产生新的 fiber 节点列表。

经过 reconcile 之后，就有了新的 fiber 树了。

这时候还没处理副作用，也就是 useEffect、生命周期等函数，这些会在 reconcile 结束之后处理。

所以 react 渲染流程整体分为两个大阶段： render 阶段和 commit 阶段。

render 阶段也就是 reconcile 的 vdom 转 fiber 的过程，commit 阶段就是具体操作 dom，以及执行副作用函数的过程。

commit 阶段还分为了 3 个小阶段：before mutation、mutation、layout。

![](https://pic1.zhimg.com/v2-5bdedd86191f73018647e193db44cad0_r.jpg)

具体操作 dom 的阶段是 mutation，操作 dom 之前是 before mutation，而操作 dom 之后是 layout。

layout 阶段在操作 dom 之后，所以这个阶段是能拿到 dom 的，ref 更新是在这个阶段，useLayoutEffect 回调函数的执行也是在这个阶段。

理清了 react 的渲染流程 render + commit(before mutation、mutation、layout) 之后，我们来进入今天的主要内容 “hooks 实现原理” 部分吧。

hooks 的数据保存在哪里呢？比如 useState 的 state，useRef 的 ref 等。

很容易想到是在 fiber 节点上。

比如这样 3 个 hook：

![](https://pic4.zhimg.com/v2-41cdee8cd26e7528c1456a1aac759667_r.jpg)

你就可以在 fiber 节点上找到对应的 3 个 memoizedState 的链表节点。

![](https://pic4.zhimg.com/v2-e30dd32ec2e913b35567db9b3026c977_r.jpg)

hook 的 api 就是在 fiber 的 memoizedState 链表上存取数据的。

那是什么时候构造这个链表的呢？

在第一次调用 useXxx api 的时候。

比如 useRef 第一次调用会走到 mountRef：

![](https://pic3.zhimg.com/v2-65e4c5d4161ad3801c6dd619ef10ae26_r.jpg)

在 mountRef 里可以看到它创建了一个 hook 节点，然后设置了 memoizedState 属性为有 current 属性的对象，也就是 ref 对象。

![](https://pic2.zhimg.com/v2-cc097389fd428687d2c80dbf6950d73d_r.jpg)

具体创建 hook 链表的过程也很容易看懂：

![](https://pic4.zhimg.com/v2-e9267f94302fcac8a05d39f11f884ad3_r.jpg)

就是第一个节点挂在 fiber 节点的 memoizedState 属性上，后面的挂在上个节点的 next 属性上。

只有第一次 mountRef，那第二次呢？

第二次会走到 updateRef：

![](https://pic1.zhimg.com/v2-3a1e570eb027b1563097bc9ccd1a8394_r.jpg)

这里的 updateRef 就是取出 hook 的 momorizedState 的值直接返回了：

![](https://pic4.zhimg.com/v2-632ae77f876d5cc5ae027f0009d61e5b_r.jpg)

所以 useRef 的返回的 ref 对象始终是最开始那个。

再看几个别的 hook，比如 useMemo，它是当依赖不变的时候始终返回之前创建的对象，当依赖变了才重新创建。

一般是用在 props 上，因为组件只要 props 变了就会重新渲染，用 useMemo 可以避免没必要的 props 变化。

在 antd 源码里就用到很多：

![](https://pic4.zhimg.com/v2-f4f0f9c9873e459f96f1614dfb770ed3_r.jpg)

上面这个值就是作为组件的 props 的，如果不用 useMemo 包裹，那每次都会变成一个新对象，每次都会触发子组件重新渲染。

这就是 useMemo 的作用，useCallback 也是同理。

![](https://pic2.zhimg.com/v2-c2e5945e894dfcfc10f6361a39548909_r.jpg)

它们是怎么实现的呢？

useMemo 同样也是分为 mountMemo 和 updateMemo 两个阶段。

mount 的时候是这样的：

![](https://pic2.zhimg.com/v2-9b4d0529a7150a8afbc22e0a6bacdd05_r.jpg)

创建 hook，然后执行传入的 create 函数，把值设置到 hook.memoizedState 属性上。

update 的时候会判断依赖有没有变：

![](https://pic1.zhimg.com/v2-a8a929c5eb2508f0b7daffb139e423e8_r.jpg)

如果依赖数组都没变，那就返回之前的值，否则创建新的值更新到 hook.memoizedState。

很容易想到 useCallback 的实现是分为 mountCallback 和 updateCallback 的：

![](https://pic1.zhimg.com/v2-7cedaaa2a5ec8e385950ab5e6a154ba4_r.jpg)![](https://pic2.zhimg.com/v2-5f32051d845373ee8ab0f4901a8289f9_r.jpg)

和 useMemo 的实现大同小异。

至此，我们可以小结一下了：

**hook 的数据是存放在 fiber 的 memoizedState 属性的链表上的，每个 hook 对应一个节点，第一次执行 useXxx 的 hook 会走 mountXxx 的逻辑来创建 hook 链表，之后会走 updateXxx 的逻辑。**

当然，前面的 useRef、useCallback、useMemo 都比较简单，只是 mountXxx 和 updateXxx 里的那几行代码。

但 useState 和 useEffect 就没那么简单了，因为它们涉及到了渲染的流程。

我们先来看 useEffect，它是用来封装副作用逻辑的。

比如这样：

![](https://pic2.zhimg.com/v2-c47e0be899fd6b5987fb80468fd362a5_r.jpg)

它同样分了 mountEffect 和 updateEffect 两个阶段：

![](https://pic4.zhimg.com/v2-ade4032e1d2075b2704904180fabfe37_r.jpg)

mountEffect 里执行了一个 pushEffect 的函数：

![](https://pic4.zhimg.com/v2-1c252e6fe62c690b1d5affaa53b81ccf_r.jpg)

在 updateEffect 里也是，只是多了依赖数组变化的检测逻辑：

![](https://pic1.zhimg.com/v2-8059e5e8e902c0f5ce738a2964d6c168_r.jpg)

那这个 pusheEffect 做了什么呢？

这里面创建了 effect 对象并把它放到了 fiber.updateQueue 上：

![](https://pic4.zhimg.com/v2-904c0b94b62dbb0482cfb42dca5a6907_r.jpg)

updateQueue 是个环形链表，有个 lastEffect 来指向最后一个 effect。

为什么要这样设计呢？

因为这样新的 effect 好往后面插入呀，直接设置 lastEffect.next 就行。

也就是说我们执行完 useEffect 之后，就把 effect 串联起来放到了 fiber.updateQueue 上。

那什么时候执行 effect 呢？

这个前面说过了，就是 commit 阶段执行。

那是在 commit 阶段的 before mutation、mutation、layout 的哪个阶段执行呢？

![](https://pic2.zhimg.com/v2-571f655a362748f2165492917c82d7d5_r.jpg)

都不是。

是在 commit 最开始的时候，异步处理的 effect 列表：

![](https://pic1.zhimg.com/v2-24f9d4e0e520ec1ee4477e515ee07360_r.jpg)

具体处理的过程就是取出 fiber.updateQueue，然后从 lastEffect.next 开始循环处理

![](https://pic3.zhimg.com/v2-84b2415dd336a73fa3d8bc953b979e8e_r.jpg)

遍历完一遍 fiber 树，处理完每个 fiber.updateQueue 就处理完了所有的 useEffect 的回调：

![](https://pic4.zhimg.com/v2-f40a71ae16e51ee834157e09eaa74d4b_r.jpg)

那有的同学说了，不在 before mutation、mutation、layout 阶段执行有啥好处呢？

因为异步执行不阻塞渲染呀！

当然，还有个 useLayoutEffect 的 hook，它是在 layout 阶段同步调用的。

比如这样的代码：

![](https://pic1.zhimg.com/v2-264161a892167bb9e2a23355252c5c8c_r.jpg)

大家觉得打印顺序是什么呢？

![](https://pic4.zhimg.com/v2-6aec33b3a20e63858059c2c7a5f462e7_b.jpg)

结果是先 layout effect 再 effect。

因为 layout effect 是在 layout 阶段，也就是 dom 更新之后同步调用的，而 effect 是异步调用的。

一般不建议用 useLayoutEffect，因为同步逻辑会阻塞渲染。

layout effect 的执行就是在 layout 阶段遍历所有 fiber，取出 updateQueue 的每个 effect 执行。

![](https://pic4.zhimg.com/v2-18d6c391bf60ff1385fa29e93e25f0a7_r.jpg)

这就是 effect 的实现原理。

小结一下：

**useEffect 的 hook 在 render 阶段会把 effect 放到 fiber 的 updateQueue 中，这是一个 lastEffect.next 串联的环形链表，然后 commit 阶段会异步执行所有 fiber 节点的 updateQueue 中的 effect。**

**useLayoutEffect 和 useEffect 差不多，区别只是它是在 commit 阶段的 layout 阶段同步执行所有 fiber 节点的 updateQueue 中的 effect。**

最后，我们再来看下 useState 的实现：

同样要分为 mountState 和 updateState 来看：

它把 initialState 设置到了 hook.baseState 上，这是 state 最终保存的地方。

![](https://pic1.zhimg.com/v2-c31a343468cd1d73231d91e7404e5adc_r.jpg)

然后创建了一个 queue，这个是用于多个 setState 的时候记录每次更新的。

返回的第二个值是 dispatch 函数，给他绑定了当前的 fiber 还有那个 queue。

这样，当你再执行返回的 setXxx 函数的时候就会走到 dispatch 逻辑：

![](https://pic3.zhimg.com/v2-2e79f136e43952f7d083443ad4d8f076_r.jpg)

这时候前两个参数 fiber 和 queue 都是 bind 的值，只有第三个参数是传入的新 state，当然，现在还叫 action：

![](https://pic4.zhimg.com/v2-3cfc59aa14e359dae6231b75f707fb7b_r.jpg)

它会创建一个 update 对象，然后标记 fiber 节点，之后调度下次渲染：

![](https://pic4.zhimg.com/v2-5b839df79a60a21803ecc59385c2831b_r.jpg)

这里要简单介绍下优先级机制 lane。

假设有 30 多种优先级，怎么表示呢？

用数字么？

这样计算太慢了，而且如果同时有几种优先级计算起来就比较麻烦了。

所以 react 选择了用二进制的方式来表示：

![](https://pic3.zhimg.com/v2-fc33274438a63eb050702e90a82c86f6_r.jpg)

每个二进制位代表一种优先级，有多个优先级就是多个位为 1。

这样通过位运算就能轻松算出是啥优先级：

![](https://pic3.zhimg.com/v2-7916e00d3c0cb8530d0b7338631a8f62_r.jpg)

这种机制就叫做 lane，因为二进制的位就像一条条赛道一样，很形象：

![](https://pic4.zhimg.com/v2-03e3a625920cfe140bea03004e8e7ab3_r.jpg)

创建了 update 对象之后就要标记 fiber 节点有更新了，不只是要标记那个节点，还要标记它的父节点直到跟节点：

所以这个方法名字就叫做 markUpdateFromFiberToRoot，也就是从当前 fiber 一直到 root 的意思：

![](https://pic1.zhimg.com/v2-59604e661aa7ef73623ae8ff5abcba68_r.jpg)

做的事情就是循环往上一层层 merge lane。

不过当前节点是 fiber.lanes，而父节点是 fiber.childLanes，用来区分是当前节点的更新还是子节点的更新。

标记完更新就是调度下次渲染了。

也就是 scheduleUpdateOnFiber 这个方法：

![](https://pic1.zhimg.com/v2-230295d6abc914ccac6061be6fb28bd0_r.jpg)

它里面最终会调用到 renderRootSync，也就是从跟节点开启新的 vdom 转 fiber 的循环：

![](https://pic4.zhimg.com/v2-155693f00c71d27891fe9c4690f13bd7_r.jpg)![](https://pic4.zhimg.com/v2-84d91ad8f73272d889b0d0d7ca6f73cb_r.jpg)

这样就触发了新一次渲染。

然后再渲染到这个函数的时候就会执行到 updateState：

![](https://pic3.zhimg.com/v2-2b7599de1e60ddd3bc98fde7ed155ba2_r.jpg)![](https://pic4.zhimg.com/v2-ca62c660d212512280d50e77f62aee5f_r.jpg)

updateState 会调用 updateReducer，选出最终的 state 来返回做渲染：

怎么决定 state 要更新成啥呢？

自然也是根据优先级，这里会根据 lane 来比较，然后做 state 的合并，最后返回一个新的 state：

![](https://pic2.zhimg.com/v2-a9503148223584e175a58c77c8513461_r.jpg)

这样组件里拿到的就是新 state，然后根据它做渲染。

这就是 useState 的实现原理。

小结一下：

**useState 同样分为 mountState 和 updateState 两个阶段：**

**mountState 会返回 state 和 dispatch 函数，dispatch 函数里会记录更新到 hook.queue，然后标记当前 fiber 到根 fiber 的 lane 需要更新，之后调度下次渲染。**

**再次渲染的时候会执行 updateState，会取出 hook.queue，根据优先级确定最终的 state 返回，这样渲染出的就是最新的结果。**

**总结**
------

react 渲染流程分为 render 和 commit 阶段。

render 阶段执行 vdom 转 fiber 的 reconcile，commit 阶段更新 dom，执行 effect 等副作用逻辑。

commit 阶段分为 before mutation、mutation、layout 3 个小阶段。

hook 的数据就是保存在 fiber.memoizedState 的链表上的，每个 hook 对应一个链表节点。

hook 的执行分为 mountXxx 和 updateXxx 两个阶段，第一次会走 mountXxx，创建 hook 链表，之后执行 updateXxx。

我们看了 useRef、useMemo、useCallback 的实现原理，这几个 hook 都比较简单。其中后两个 hook 是作为 props 时为了减少不必要渲染的时候用的。

useState 和 useEffect 就和渲染流程有关了：

useEffect 在 render 阶段会把 effect 放到 fiber.updateQueue 的环形链表上，然后在 commit 阶段遍历所有 fiber 的 updateQueue，取出 effect 异步执行。

useLayoutEffect 和 useEffect 差不多，只是 effect 链表是在 layout 阶段同步执行的。

useState 的 mountState 阶段返回的 setXxx 是绑定了几个参数的 dispatch 函数。执行它会创建 hook.queue 记录更新，然后标记从当前到根节点的 fiber 的 lanes 和 childLanes 需要更新，然后调度下次渲染。

下次渲染执行到 updateState 阶段会取出 hook.queue，根据优先级确定最终的 state，最后返回来渲染。

这样就实现了 state 的更新和重新渲染。

这就是 react hooks 特别是 useState 和 useEffect 的实现原理。理解它们不单单要理解 hook 的存储结构，还要理解 react 的整个渲染流程。