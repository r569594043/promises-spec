# Promises/A+

**一个为健全可互操作的JavaScript Promise制定的开放标准。匠者为之，以惠匠者。**

*promise* 表示一个异步操作的最终结果。 与 promise 进行交互的主要方式是通过 `then` 方法，该方法注册了两个回调函数，用于接受 promise 的最终结果或者 promise 的据因。

本规范详细列出了 `then` 方法的行为，所有遵循 Promises/A+ 规范实现的 promise 均可参照标准以实施 then 方法，故而提供了一个的可互操作的基础。因此，本规范可以被认为是十分稳定的。尽管 Promises/A+ 组织可能有时会修订本规范，造成微小的、向下兼容的改变以解决新发现的一些边界值，但我们仅仅在慎重考虑、详细讨论和严格测试之后才会进行在大规模或向下不兼容的集成。

从历史上说， Promises/A+ 规范将之前 [Promises/A proposal](http://wiki.commonjs.org/wiki/Promises/A) 规范的建议明确为了行为标准。其扩展了原规范以覆盖一些*约定俗成*的行为，以及省略掉一些仅在特定情况下存在的或者有问题的部分。

最后，核心的 Promises/A+ 规范不设计如何建立、执行、拒绝 promises，而是专注于提供一个可互操作的 `then` 方法。上述对于 promises 的操作方法将来在其他规范中可能会触及。

## 术语

1. "promise" 是一个拥有 `then` 方法的对象或函数，其行为符合本规范。
1. "thenable" 是一个定义 `then` 方法的对象或函数。
1. "值"（value） 指任何 JavaScript 的合法值（包括 `undefined` , thenable 和 promise）。
1. "异常"（exception） 是使用 `throw` 语句抛出的一个值。
1. "据因"（reason） 是一个值以表明为何一个 promise 会被拒绝。

## 要求

### Promise的状态

一个 Promise 必须处于等待态（Pending）、执行态（Fulfilled）和拒绝态（Rejected）这三种状态中的一种之中。

1. 处于等待态时，promise :
    1. 可以迁移至执行态或拒绝态
1. 处于执行态时，promise :
    1. 不能迁移至其他任何状态
    1. 必须拥有一个不可变的值
1. 处于拒绝态时，promise:
    1. 不能迁移至其他任何状态
    1. 必须拥有一个不可变的据因

这里的"不可变"指的是恒等（即可用 `===` 判断相等），而不是意味着更深层次的不可变。

### `Then` 方法

一个 promise 必须提供一个 `then` 方法以访问其当前值、最终返回值和据因。

promise 的 `then` 方法接受两个参数：

```js
promise.then(onFulfilled, onRejected)
```

1. `onFulfilled` 和 `onRejected` 都是可选参数:
    1. 如果 `onFulfilled` 不是函数，其必须被忽略
    1. 如果 `onRejected` 不是函数，其必须被忽略
1. 如果 `onFulfilled` 是函数:
    1. 当 `promise` 执行结束后其必须被调用，其第一个参数为 `promise` 的值
    1. 在 `promise` 执行结束前其不可被调用
    1. 其调用次数不可超过一次
1. 如果 `onRejected` 是函数,
    1. 当 `promise` 被拒绝执行后其必须被调用，其第一个参数为 `promise` 的据因
    1. 在 `promise` 被拒绝执行前其不可被调用
    1. 其调用次数不可超过一次
1. `onFulfilled` 和 `onRejected` 直到[执行环境](http://es5.github.io/#x10.3)堆栈尽包含平台代码前不可被调用。[[3.1](#注释)].
1. `onFulfilled` 和 `onRejected` 必须被作为函数调用（即没有 `this` 值）。[[3.2](#注释)]
1. `then` 方法可以被同一个 promise 调用多次
    1. 当 `promise` 成功执行时，所有 `onFulfilled` 需按照其注册顺序依次回调
    1. 当 `promise` 被拒绝执行时，所有的 `onRejected` 需按照其注册顺序依次回调
1. `then` 方法必须返回一个 promise 对象[[3.3](#注释)]。

    ```js
    promise2 = promise1.then(onFulfilled, onRejected);
    ```

    1. 如果 `onFulfilled` 或者 `onRejected` 返回一个值 `x` ，则运行下面的 Promise 解决程序：`[[Resolve]](promise2, x)`
    1. 如果 `onFulfilled` 或者 `onRejected` 抛出一个异常 `e` ，则 `promise2` 必须拒绝执行，并返回据因 `e`
    1. 如果 `onFulfilled` 不是函数且 `promise1` 成功执行， `promise2` 必须成功执行并返回相同的值
    1. 如果 `onRejected` 不是函数且 `promise1` 拒绝执行， `promise2` 必须拒绝执行并返回相同的据因

### Promise 解决程序

**Promise解决程序**是一个抽象的操作，其需输入一个 promose 和一个值，我们表示为 `[[Resolve]](promise, x)`，如果 `x` 是 thenable 的，同时若 `x` 至少满足和 promise 类似（即鸭子类型， x 拥有部分或全部 promise 拥有的方法属性）的前提，解决程序即尝试使 `promise` 接受 `x` 的状态；否则其用 `x` 的值来执行 `promise` 。

这种对 thenable 的操作允许 promise 实现互操作，只要其暴露出一个遵循 Promise/A+ 协议的 `then` 方法。

运行 `[[Resolve]](promise, x)` 需遵循以下步骤：

1. 如果 `promise` 和 `x` 指向同一对象，以 `TypeError` 为据因拒绝执行 `promise`
1. 如果 `x` 为 promise ，接受其状态 [[3.4](#注释)]:
   1. 如果 `x` 处于等待态， `promise` 需保持为等待态直至 `x` 被执行或拒绝
   1. 如果 `x` 处于执行态，用相同的值执行 `promise`
   1. 如果 `x` 处于拒绝态，用相同的据因拒绝 `promise`
1. 抑或 `x` 为对象或者函数：
   1. 设置 `then` 方法为 `x.then` [[3.5](#注释)]
   1. 如果取 `x.then` 的返回值时抛出错误 `e` ，则以 `e` 为据因拒绝 `promise`
   1. 如果 `then` 是函数，将 `x` 作为函数的作用域 `this` 调用之。其第一个参数为 `resolvePromise` ，第二个参数为 `rejectPromise`:
      1. 如果 `resolvePromise` 以值 `y` 为参数被调用，则运行 `[[Resolve]](promise, y)`
      1. 如果 `rejectPromise` 以据因 `r` 为参数被调用，则以据因 `r` 拒绝 `promise`
      1. 如果 `resolvePromise` 和 `rejectPromise` 均被调用，或者被同一参数调用了多次，则优先采用首次调用和忽略剩下的调用
      1. If calling `then` throws an exception `e`,
      1. 如果调用 `then` 方法抛出了异常 `e`
         1. 如果 `resolvePromise` 或 `rejectPromise` 已经被调用，则忽略之
         1. 否则以 `e` 为据因拒绝 `promise`
   1. 如果 `then` 不是函数，以 `x` 为参数执行 `promise`
1. 如果 `x` 不为对象或者函数，以 `x` 为参数执行 `promise`

如果一个 promise 被一个循环的 thenable 链中的对象解决，而 `[[Resolve]](promise, thenable)` 的递归性质又使得其被再次调用，根据上述的算法将会陷入无限递归之中。算法不强制要求，但鼓励其实施者以检测这样的递归是否存在，若存在则以一个可识别的 `TypeError` 为据因来拒绝 `promise`[[3.6](#注释)]

## 注释

1. 这里的"平台代码"指的是引擎、环境以及 promise 的实施代码。实践中要确保 `onFulfilled` 和 `onRejected` 在 `then` 方法被调用后的事件循环中异步执行和一个全新的堆栈。这可以用一个"宏任务"机制（如 [`setTimeout`](http://www.whatwg.org/specs/web-apps/current-work/multipage/timers.html#timers) 或 [`setImmediate`](https://dvcs.w3.org/hg/webperf/raw-file/tip/specs/setImmediate/Overview.html#processingmodel) ）或者"微任务"机制（如 [`MutationObserver`](http://dom.spec.whatwg.org/#interface-mutationobserver) 或 [`process.nextTick`](http://nodejs.org/api/process.html#process_process_nexttick_callback) ）来实现。由于 promise 的实施代码本身就是平台代码，故其自身可以包含一个任务调度队列或者在处理程序被调用时的"蹦床"。

1. 也就是说在严格模式（strict）中，函数 `this` 的值为 `undefined` ；在非严格模式中其为全局对象。

1. 代码实现在满足所有要求的情况下可以允许 `promise2 === promise1` 。每个实现都要文档说明其是否允许以及在何种条件下允许 `promise2 === promise1` 。

1. 总体来说，其只会知道 `x` 是真正的 promise 如果其来自当前实施。此标准允许使用特殊的实现方式以接受符合已知要求的 promises 状态。

1. 这一步是存储一个指向 `x.then` 的引用，然后测试该引用，然后调用该引用。以避免多次访问 `x.then` 属性。这种预防措施在确保可访问属性的一致性上非常重要，因为其值可能在返回时被改变。

1. 实现*不*应该对 thenable 链的深度设限，并假定超出本限制的递归就是无限循环。只有真正的循环递归才应能导致 `TypeError` 异常；如果一条无限长的链上 thenable 均不相同，那么递归下去永远是正确的行为。


## 原文
  [http://www.ituring.com.cn/article/66566](http://www.ituring.com.cn/article/66566)

---

<p xmlns:dct="http://purl.org/dc/terms/" xmlns:vcard="http://www.w3.org/2001/vcard-rdf/3.0#">
  <a rel="license"
     href="http://creativecommons.org/publicdomain/zero/1.0/">
    <img src="http://i.creativecommons.org/p/zero/1.0/88x31.png" style="border-style: none;" alt="CC0" />
  </a>
  <br />
  To the extent possible under law,
  <a rel="dct:publisher"
     href="https://github.com/promises-aplus">
    <span property="dct:title">the Promises/A+ organization</span></a>
  has waived all copyright and related or neighboring rights to
  <span property="dct:title">Promises/A+ Promise Specification</span>.
This work is published from:
<span property="vcard:Country" datatype="dct:ISO3166"
      content="US" about="https://github.com/promises-aplus">
  United States</span>.
</p>
