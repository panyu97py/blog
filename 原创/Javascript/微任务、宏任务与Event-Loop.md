# event-loop 

> 本文大部分概念参照[JavaScript 运行机制详解：再谈Event Loop](https://www.ruanyifeng.com/blog/2014/10/event-loop.html)，并稍加理解，主要是为了记忆和理解相关概念。

## 什么是event-loop



首先我们都知道`javvascript` 是`单线程`的，那也就是所同一事件只能做一件事。那么为什么`JavaScript`是`单线程`的呢？



### 为什么JavaScript是单线程？



`JavaScrip`t的`单线程`，与它的用途有关。作为浏览器脚本语言，`JavaScript`的主要用途是与用户互动，以及操作DOM。这决定了它只能是单线程，否则会带来很复杂的同步问题。比如，假定`JavaScript`同时有两个线程，一个线程在某个`DOM`节点上添加内容，另一个线程删除了这个节点，这时浏览器应该以哪个线程为准？



所以，为了避免复杂性，从一诞生，`JavaScript`就是`单线程`，这已经成了这门语言的核心特征，将来也不会改变。



为了利用多核CPU的计算能力，`HTML5`提出`Web Worker`标准，允许`JavaScript`脚本创建多个线程，但是子线程完全受主线程控制，且不得操作`DOM`。所以，这个新标准并没有改变`JavaScript`单线程的本质。



### 同步任务和异步任务



由于`单线程`的特性，所有的任务都需要排队。不论前一个任务耗时多久后一个任务都需要等待直到前一个任务结束。



但假如前一个任务是`I/O`或者`Ajax`，受限于外部原因导致任务耗时很久就会导致进程阻塞，而此时cpu是空闲的完全可以不管`I/O`或者`Ajax`挂起等待中的任务，先运行排在后面的任务等到`I/O`或者`Ajax`返回了结果在回过头把挂起的任务继续下去。



于是所有的任务可以分成两种`同步任务`和`异步任务`

* 同步任务：在`主线程`上排队执行的任务，只有前一个任务执行完毕，才能执行后一个任务
* 异步任务：不进入主线程、而进入`任务队列（task queue）`的任务，只有`任务队列`通知`主线程`，某个`异步任务`可以执行了，该任务才会进入`主线程`执行



所以`javascript`的事件运行机制如下：

1. 所有同步任务都在`主线程`上执行，形成一个[执行栈](https://www.ruanyifeng.com/blog/2013/11/stack.html)（execution context stack）
2. `主线程`之外，还存在一个`任务队列（task queue）`。只要`异步任务`有了运行结果，就在`任务队列`之中放置一个事件
3. 一旦`执行栈`中的所有`同步任务`执行完毕，系统就会读取`任务队列`，看看里面有哪些事件。那些对应的`异步任务`，于是结束等待状态，进入执行栈，开始执行
4. 主线程不断重复上面的第三步

`主线程`从`任务队列`中读取事件，这个过程是循环不断的，所以整个的这种运行机制又称为`Event Loop（事件循环）`



# 什么是宏任务？什么是微任务

## 概念

1. 宏任务：当前调用栈中执行的代码成为`宏任务`。（主代码快，定时器等等）。

2. 微任务：**当前**（此次事件循环中）`宏任务`执行完，在**下一个**`宏任务`开始之前需要执行的任务,可以理解为回调事件。（promise.then，proness.nextTick等等）。

3. `宏任务`中的事件放在`callback queue`中，由`事件触发线程`维护；`微任务`的事件放在`微任务队列`中，由`js引擎线程`维护

## 执行机制

> 主任务(宏任务) ---> 所有微任务 ---> 宏任务

![执行机制](assets/%E5%BE%AE%E4%BB%BB%E5%8A%A1%E3%80%81%E5%AE%8F%E4%BB%BB%E5%8A%A1%E4%B8%8EEvent-Loop.assets/%E6%89%A7%E8%A1%8C%E6%9C%BA%E5%88%B6.png)

## 宏任务与微任务具体包含哪些方法

### 宏任务

| #                                                            | 浏览器 | Node |
| ------------------------------------------------------------ | ------ | ---- |
| `I/O`                                                        | ✅      | ✅    |
| `setTimeout`                                                 | ✅      | ✅    |
| `setInterval`                                                | ✅      | ✅    |
| [`setImmediate`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/setImmediate) | ❌      | ✅    |
| [`requestAnimationFrame`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame) | ✅      | ❌    |

### 微任务

| #                                                            | 浏览器 | Node |
| ------------------------------------------------------------ | ------ | ---- |
| `process.nextTick`                                           | ❌      | ✅    |
| [`MutationObserver`](https://developer.mozilla.org/zh-CN/docs/Web/API/MutationObserver) | ✅      | ❌    |
| `Promise.then catch finally`                                 | ✅      | ✅    |

# 参考

[JavaScript 运行机制详解：再谈Event Loop](https://www.ruanyifeng.com/blog/2014/10/event-loop.html)

[微任务、宏任务与Event-Loop](https://juejin.cn/post/6844903657264136200)

[为什么任务要分同步和异步?宏任务与微任务又有何区别? (结合面试题理解)](https://segmentfault.com/a/1190000040504943)
