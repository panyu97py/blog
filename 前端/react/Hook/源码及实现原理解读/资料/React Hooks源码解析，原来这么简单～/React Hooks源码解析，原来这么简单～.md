# React Hooks源码解析，原来这么简单～

### 前言

从React Hooks发布以来，整个社区都以积极的态度去拥抱它、学习它。期间也涌现了很多关于React Hooks 源码解析的文章。本文（基于v16.8.6）就以笔者自己的角度来写一篇属于自己的文章吧。希望可以**深入浅出、图文并茂**的帮助大家对React Hooks的实现原理进行学习与理解。本文将以文字、代码、图画的形式来呈现内容。主要对常用Hooks中的 **useState、useReducer、useEffect** 进行学习，尽可能的揭开Hooks的面纱。

### 使用Hooks时的疑惑

Hooks的面世让我们的Function Component逐步拥有了对标Class Component的特性，比如私有状态，生命周期函数等。**useState与useReducer**这两个Hooks让我们可以在 Function Component里使用到私有状态。而useState其实就是阉割版的useReducer，这也是我那它们两个放在一起讲的原因。应用一下官方的例子：

```
function PersionInfo ({initialAge,initialName}) {
  const [age, setAge] = useState(initialAge);
  const [name, setName] = useState(initialName);
  return (
    <>
      Age: {age}, Name: {name}
      <button onClick={() => setAge(age + 1)}>Growing up</button>
    </>
  );
}
复制代码
```

通过 **useState** 我们可以初始化一个私有状态，它会返回这个状态的最新值和一个用来更新状态的方法。而**useReducer**则是针对更复杂的状态管理场景：

```
const initialState = {age: 0, name: 'Dan'};

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return {...state, age: state.age + action.age};
    case 'decrement':
      return {...state, age: state.age - action.age};
    default:
      throw new Error();
  }
}
function PersionInfo() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      Age: {state.age}, Name: {state.name}
      <button onClick={() => dispatch({type: 'decrement', age: 1})}>-</button>
      <button onClick={() => dispatch({type: 'increment', age: 1})}>+</button>
    </>
  );
}
复制代码
```

同样也是返回当前最新的状态，并返回一个用来更新数据的方法。在使用这两个方法的时候也许我们会想过这样的问题：

```
  const [age, setAge] = useState(initialAge);
  const [name, setName] = useState(initialName);
复制代码
```

**React内部是怎么区分这两个状态的呢？** Function Component 不像 Class Component那样可以将私有状态挂载到类实例中并通过对应的key来指向对应的状态，而且每次的页面的刷新或者说组件的重新渲染都会使得 Function 重新执行一遍。所以React中必定有一种机制来区分这些Hooks。

```
 const [age, setAge] = useState(initialAge);
 // 或
 const [state, dispatch] = useReducer(reducer, initialState);
复制代码
```

**另一个问题就是React是如何在每次重新渲染之后都能返回最新的状态？** Class Component因为自身的特点可以将私有状态持久化的挂载到类实例上，每时每刻保存的都是最新的值。而 Function Component 由于本质就是一个函数，并且每次渲染都会重新执行。所以React必定拥有某种机制去记住每一次的更新操作，并最终得出最新的值返回。当然我们还会有其他的一些问题，比如**这些状态究竟存放在哪？为什么只能在函数顶层使用Hooks而不能在条件语句等里面使用Hooks？**

### 答案尽在源码之中

我们先来了解useState以及useReducer的源码实现，并从中解答我们在使用Hooks时的种种疑惑。首先我们从源头开始：

```
import React, { useState } from 'react';
复制代码
```

在项目中我们通常会以这种方式来引入**useState**方法，被我们引入的这个**useState**方法是什么样子的呢？其实这个方法就在源码 **packages/react/src/ReactHook.js** 中。

```
// packages/react/src/ReactHook.js
import ReactCurrentDispatcher from './ReactCurrentDispatcher';

function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current;
  // ... 
  return dispatcher;
}

// 我们代码中引入的useState方法
export function useState(initialState) {
  const dispatcher = resolveDispatcher();
  return dispatcher.useState(initialState)
}
复制代码
```

从源码中可以看到，我们调用的其实是 ReactCurrentDispatcher.js 中的dispatcher.useState()，那么我们继续前往ReactCurrentDispatcher.js文件：

```
import type {Dispacther} from 'react-reconciler/src/ReactFiberHooks';

const ReactCurrentDispatcher = {
  current: (null: null | Dispatcher),
};

export default ReactCurrentDispatcher;
复制代码
```

好吧，它继续将我们带向 **react-reconciler/src/ReactFiberHooks.js**这个文件。那么我们继续前往这个文件。

```
// react-reconciler/src/ReactFiberHooks.js
export type Dispatcher = {
  useState<S>(initialState: (() => S) | S): [S, Dispatch<BasicStateAction<S>>],
  useReducer<S, I, A>(
    reducer: (S, A) => S,
    initialArg: I,
    init?: (I) => S,
  ): [S, Dispatch<A>],
  useEffect(
    create: () => (() => void) | void,
    deps: Array<mixed> | void | null,
  ): void,
  // 其他hooks类型定义
}
复制代码
```

兜兜转转我们终于清楚了React Hooks 的源码就放 **react-reconciler/src/ReactFiberHooks.js** 目录下面。 在这里如上图所示我们可以看到有每个Hooks的类型定义。同时我们也可以看到Hooks的具体实现，大家可以多看看这个文件。首先我们注意到，我们大部分的Hooks都有两个定义：

```
// react-reconciler/src/ReactFiberHooks.js
// Mount 阶段Hooks的定义
const HooksDispatcherOnMount: Dispatcher = {
  useEffect: mountEffect,
  useReducer: mountReducer,
  useState: mountState,
 // 其他Hooks
};

// Update阶段Hooks的定义
const HooksDispatcherOnUpdate: Dispatcher = {
  useEffect: updateEffect,
  useReducer: updateReducer,
  useState: updateState,
  // 其他Hooks
};
复制代码
```

从这里可以看出，我们的Hooks在Mount阶段和Update阶段的逻辑是不一样的。在Mount阶段和Update阶段他们是两个不同的定义。我们先来看Mount阶段的逻辑。在看之前我们先思考一些问题。React Hooks需要在Mount阶段做什么呢？就拿我们的useState和useReducer来说：

1. 我们需要初始化状态，并返回修改状态的方法，这是最基本的。
2. 我们要区分管理每个Hooks。
3. 提供一个数据结构去存放更新逻辑，以便后续每次更新可以拿到最新的值。

我们一下React的实现，先来看mountState的实现。

```
// react-reconciler/src/ReactFiberHooks.js
function mountState (initialState) {
  // 获取当前的Hook节点，同时将当前Hook添加到Hook链表中
  const hook = mountWorkInProgressHook();
  if (typeof initialState === 'function') {
    initialState = initialState();
  }
  hook.memoizedState = hook.baseState = initialState;
  // 声明一个链表来存放更新
  const queue = (hook.queue = {
    last: null,
    dispatch: null,
    lastRenderedReducer,
    lastRenderedState,
  });
  // 返回一个dispatch方法用来修改状态，并将此次更新添加update链表中
  const dispatch = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  )));
  // 返回当前状态和修改状态的方法 
  return [hook.memoizedState, dispatch];
}
复制代码
```

#### 区分管理Hooks

关于第一件事，初始化状态并返回状态和更新状态的方法。这个没有问题，源码也很清晰利用**initialState**来初始化状态，并且返回了状态和对应更新方法 **return [hook.memoizedState, dispatch]**。那么我们来看看React是如何区分不同的Hooks的，这里我们可以从 mountState 里的 **mountWorkInProgressHook**方法和Hook的类型定义中找到答案。

```
// react-reconciler/src/ReactFiberHooks.js
export type Hook = {
  memoizedState: any,
  baseState: any,
  baseUpdate: Update<any, any> | null,
  queue: UpdateQueue<any, any> | null,
  next: Hook | null,  // 指向下一个Hook
};
复制代码
```

首先从Hook的类型定义中就可以看到，React 对Hooks的定义是链表。也就是说我们组件里使用到的Hooks是通过链表来联系的，上一个Hooks的next指向下一个Hooks。这些Hooks节点是怎么利用链表数据结构串联在一起的呢？相关逻辑就在每个具体mount 阶段 Hooks函数调用的 **mountWorkInProgressHook**方法里：

```
// react-reconciler/src/ReactFiberHooks.js
function mountWorkInProgressHook(): Hook {
  const hook: Hook = {
    memoizedState: null,
    baseState: null,
    queue: null,
    baseUpdate: null,
    next: null,
  };
  if (workInProgressHook === null) {
    // 当前workInProgressHook链表为空的话，
    // 将当前Hook作为第一个Hook
    firstWorkInProgressHook = workInProgressHook = hook;
  } else {
    // 否则将当前Hook添加到Hook链表的末尾
    workInProgressHook = workInProgressHook.next = hook;
  }
  return workInProgressHook;
}
复制代码
```

在mount阶段，每当我们调用Hooks方法，比如useState，mountState就会调用**mountWorkInProgressHook** 来创建一个Hook节点，并把它添加到Hooks链表上。比如我们的这个例子：

```
  const [age, setAge] = useState(initialAge);
  const [name, setName] = useState(initialName);
  useEffect(() => {})
复制代码
```

那么在mount阶段，就会生产如下图这样的单链表：

![img](React Hooks源码解析，原来这么简单～.assets/1-20210624144824774)



#### 返回最新的值

而关于第三件事，useState和useReducer都是使用了一个queue链表来存放每一次的更新。以便后面的update阶段可以返回最新的状态。每次我们调用dispatchAction方法的时候，就会形成一个新的updata对象，添加到queue链表上，而且这个是一个循环链表。可以看一下 **dispatchAction** 方法的实现：

```
// react-reconciler/src/ReactFiberHooks.js
// 去除特殊情况和与fiber相关的逻辑
function dispatchAction(fiber,queue,action,) {
    const update = {
      action,
      next: null,
    };
    // 将update对象添加到循环链表中
    const last = queue.last;
    if (last === null) {
      // 链表为空，将当前更新作为第一个，并保持循环
      update.next = update;
    } else {
      const first = last.next;
      if (first !== null) {
        // 在最新的update对象后面插入新的update对象
        update.next = first;
      }
      last.next = update;
    }
    // 将表头保持在最新的update对象上
    queue.last = update;
   // 进行调度工作
    scheduleWork();
}
复制代码
```

也就是我们每次执行dispatchAction方法，比如setAge或setName。就会创建一个保存着此次更新信息的update对象，添加到更新链表queue上。然后每个Hooks节点就会有自己的一个queque。比如假设我们执行了下面几个语句：

```
setAge(19);
setAge(20);
setAge(21);
复制代码
```

那么我们的Hooks链表就会变成这样：

![updateQueue](React Hooks源码解析，原来这么简单～.assets/1-20210624144824783)

在Hooks节点上面，会如上图那样，通过链表来存放所有的历史更新操作。以便在update阶段可以通过这些更新获取到最新的值返回给我们。这就是在第一次调用useState或useReducer之后，每次更新都能返回最新值的原因。再来看看mountReducer，你会发现和mountState几乎一摸一样，只是状态的初始化逻辑有那么一点区别。毕竟useState其实就是阉割版的useReducer。这里就不详细介绍mountReducer了。



```
// react-reconciler/src/ReactFiberHooks.js
function mountReducer(reducer, initialArg, init,) {
  // 获取当前的Hook节点，同时将当前Hook添加到Hook链表中
  const hook = mountWorkInProgressHook();
  let initialState;
  // 初始化
  if (init !== undefined) {
    initialState = init(initialArg);
  } else {
    initialState = initialArg ;
  }
  hook.memoizedState = hook.baseState = initialState;
  // 存放更新对象的链表
  const queue = (hook.queue = {
    last: null,
    dispatch: null,
    lastRenderedReducer: reducer,
    lastRenderedState: (initialState: any),
  });
  // 返回一个dispatch方法用来修改状态，并将此次更新添加update链表中
  const dispatch = (queue.dispatch = (dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue,
  )));
 // 返回状态和修改状态的方法
  return [hook.memoizedState, dispatch];
}
复制代码
```

然后我们来看看update阶段，也就是看一下我们的useState或useReducer是如何利用现有的信息，去给我们返回最新的最正确的值的。先来看一下useState在update阶段的代码也就是updateState：

```
// react-reconciler/src/ReactFiberHooks.js
function updateState(initialState) {
  return updateReducer(basicStateReducer, initialState);
}
复制代码
```

可以看到，updateState底层调用的其实就会死updateReducer，因为我们调用useState的时候，并不会传入reducer，所以这里会默认传递一个basicStateReducer进去。我们先看看这个**basicStateReducer**：

```
// react-reconciler/src/ReactFiberHooks.js
function basicStateReducer(state, action){
  return typeof action === 'function' ? action(state) : action;
} 
复制代码
```

在使用useState(action)的时候，action通常会是一个值，而不是一个方法。所以baseStateReducer要做的其实就是将这个action返回。来继续看一下updateReducer的逻辑：

```
// react-reconciler/src/ReactFiberHooks.js
// 去掉与fiber有关的逻辑

function updateReducer(reducer,initialArg,init) {
  const hook = updateWorkInProgressHook();
  const queue = hook.queue;

  // 拿到更新列表的表头
  const last = queue.last;

  // 获取最早的那个update对象
  first = last !== null ? last.next : null;

  if (first !== null) {
    let newState;
    let update = first;
    do {
      // 执行每一次更新，去更新状态
      const action = update.action;
      newState = reducer(newState, action);
      update = update.next;
    } while (update !== null && update !== first);

    hook.memoizedState = newState;
  }
  const dispatch = queue.dispatch;
  // 返回最新的状态和修改状态的方法
  return [hook.memoizedState, dispatch];
}
复制代码
```

在update阶段，也就是我们组件第二次第三次。。执行到useState或useReducer的时候，会遍历update对象循环链表，执行每一次更新去计算出最新的状态来返回，以保证我们每次刷新组件都能拿到当前最新的状态。**useState**的reducer是baseStateReducer，因为传入的update.action为值，所以会直接返回update.action，而**useReducer** 的reducer是用户定义的reducer，所以会根据传入的action和每次循环得到的newState逐步计算出最新的状态。

![updateReducer1](React Hooks源码解析，原来这么简单～.assets/1-20210624144824781)



### useState/useReducer 小总结

看到这里我们在回头看看最初的一些疑问：

1. React 如何管理区分Hooks？
   - React通过单链表来管理Hooks
   - 按Hooks的执行顺序依次将Hook节点添加到链表中
2. useState和useReducer如何在每次渲染时，返回最新的值？
   - 每个Hook节点通过循环链表记住所有的更新操作
   - 在update阶段会依次执行update循环链表中的所有更新操作，最终拿到最新的state返回
3. 为什么不能在条件语句等中使用Hooks？
   - 链表！



![img](React Hooks源码解析，原来这么简单～.assets/1-20210624144824773)

比如如图所示，我们在mount阶段调用了**useState('A'), useState('B'), useState('C')**，如果我们将**useState('B')** 放在条件语句内执行，并且在update阶段中因为不满足条件而没有执行的话，那么没法正确的重Hooks链表中获取信息。React也会给我们报错。



### Hooks链表放在哪？

好的，现在我们已经了解了React 通过链表来管理 Hooks，同时也是通过一个循环链表来存放每一次的更新操作，得以在每次组件更新的时候可以计算出最新的状态返回给我们。那么我们这个Hooks链表又存放在那里呢？理所当然的我们需要将它存放到一个跟当前组件相对于的地方。那么很明显这个与组件一一对应的地方就是我们的FiberNode。

![hooksInFiberNode](React Hooks源码解析，原来这么简单～.assets/1-20210624144824871)



如图所示，组件构建的Hooks链表会挂载到FiberNode节点的memoizedState上面去。

![hooksInFiberNode2](React Hooks源码解析，原来这么简单～.assets/1-20210624144824768)



### useEffect

看到这，相信你已经对Hooks的源码实现模式已经有一定的了解了，所以你尝试去看一下Effect的实现你会一下子就看懂。首先我们先回忆一下**useEffect**是怎么样工作的？

```
function PersionInfo () {
  const [age, setAge] = useState(18);
  useEffect(() =>{
      console.log(age)
  }, [age])

 const [name, setName] = useState('Dan');
 useEffect(() =>{
      console.log(name)
  }, [name])
  return (
    <>
      ...
    </>
  );
}
复制代码
```

PersionInfo组件第一次渲染的时候会在控制台输出age和name，在后面组件的每次update中，如果useEffect中的deps依赖的值发生了变化的话，也会在控制台中输出对应的状态，同时在unmount的时候就会执行清除函数（如果有）。React中是怎么实现的呢？其实很简单，在FiberNode中通过一个updateQueue来存放所有的effect，然后在每次渲染之后依次执行所有需要执行的effect。useEffect 也分为mountEffect和updateEffect

#### mountEffect

```
// react-reconciler/src/ReactFiberHooks.js
// 简化去掉特殊逻辑

function mountEffect( create,deps,) {
  return mountEffectImpl(
    create,
    deps,
  );
}

function mountEffectImpl(fiberEffectTag, hookEffectTag, create, deps) {
  // 获取当前Hook，并把当前Hook添加到Hook链表
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 将当前effect保存到Hook节点的memoizedState属性上，
  // 以及添加到fiberNode的updateQueue上
  hook.memoizedState = pushEffect(hookEffectTag, create, undefined, nextDeps);
}

function pushEffect(tag, create, destroy, deps) {
  const effect: Effect = {
    tag,
    create,
    destroy,
    deps,
    next: (null: any),
  };
  // componentUpdateQueue 会被挂载到fiberNode的updateQueue上
  if (componentUpdateQueue === null) {
    // 如果当前Queue为空，将当前effect作为第一个节点
    componentUpdateQueue = createFunctionComponentUpdateQueue();
   // 保持循环
    componentUpdateQueue.lastEffect = effect.next = effect;
  } else {
    // 否则，添加到当前的Queue链表中
    const lastEffect = componentUpdateQueue.lastEffect;
    if (lastEffect === null) {
      componentUpdateQueue.lastEffect = effect.next = effect;
    } else {
      const firstEffect = lastEffect.next;
      lastEffect.next = effect;
      effect.next = firstEffect;
      componentUpdateQueue.lastEffect = effect;
    }
  }
  return effect; 
}
复制代码
```

可以看到在mount阶段，useEffect做的事情就是将自己的effect添加到了componentUpdateQueue上。这个componentUpdateQueue会在**renderWithHooks**方法中赋值到fiberNode的updateQueue上。

```
// react-reconciler/src/ReactFiberHooks.js
// 简化去掉特殊逻辑
export function renderWithHooks() {
   const renderedWork = currentlyRenderingFiber;
   renderedWork.updateQueue = componentUpdateQueue;
}
复制代码
```

也就是在mount阶段我们所有的effect都以链表的形式被挂载到了fiberNode上。然后在组件渲染完毕之后，React就会执行updateQueue中的所有方法。

![useEffectInFiberNode1](React Hooks源码解析，原来这么简单～.assets/1-20210624144824871-4517304.)



#### updateEffect

```
// react-reconciler/src/ReactFiberHooks.js
// 简化去掉特殊逻辑

function updateEffect(create,deps){
  return updateEffectImpl(
    create,
    deps,
  );
}

function updateEffectImpl(fiberEffectTag, hookEffectTag, create, deps){
  // 获取当前Hook节点，并把它添加到Hook链表
  const hook = updateWorkInProgressHook();
  // 依赖 
  const nextDeps = deps === undefined ? null : deps;
 // 清除函数
  let destroy = undefined;

  if (currentHook !== null) {
    // 拿到前一次渲染该Hook节点的effect
    const prevEffect = currentHook.memoizedState;
    destroy = prevEffect.destroy;
    if (nextDeps !== null) {
      const prevDeps = prevEffect.deps;
      // 对比deps依赖
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        // 如果依赖没有变化，就会打上NoHookEffect tag，在commit阶段会跳过此
        // effect的执行
        pushEffect(NoHookEffect, create, destroy, nextDeps);
        return;
      }
    }
  }
  hook.memoizedState = pushEffect(hookEffectTag, create, destroy, nextDeps);
}
复制代码
```

update阶段和mount阶段类似，只不过这次会考虑effect 的依赖deps，如果此次更新effect的依赖没有变化的话，就会被打上NoHookEffect标签，最后会在commit阶段跳过改effect的执行。

```
function commitHookEffectList(unmountTag,mountTag,finishedWork) {
  const updateQueue = finishedWork.updateQueue;
  let lastEffect = updateQueue !== null ? updateQueue.lastEffect : null;
  if (lastEffect !== null) {
    const firstEffect = lastEffect.next;
    let effect = firstEffect;
    do {
      if ((effect.tag & unmountTag) !== NoHookEffect) {
        // Unmount 阶段执行tag !== NoHookEffect的effect的清除函数 （如果有的话）
        const destroy = effect.destroy;
        effect.destroy = undefined;
        if (destroy !== undefined) {
          destroy();
        }
      }
      if ((effect.tag & mountTag) !== NoHookEffect) {
        // Mount 阶段执行所有tag !== NoHookEffect的effect.create，
        // 我们的清除函数（如果有）会被返回给destroy属性，一遍unmount执行
        const create = effect.create;
        effect.destroy = create();
      }
      effect = effect.next;
    } while (effect !== firstEffect);
  }
}
复制代码
```

### useEffect 小总结



![img](https://user-gold-cdn.xitu.io/2020/3/15/170dc3955ec77444?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

useEffect做了什么？



1. FiberNdoe节点中会又一个updateQueue链表来存放所有的本次渲染需要执行的effect。
2. mountEffect阶段和updateEffect阶段会把effect 挂载到updateQueue上。
3. updateEffect阶段，deps没有改变的effect会被打上NoHookEffect tag，commit阶段会跳过该Effect。

到此为止，**useState/useReducer/useEffect**源码也阅读完毕了，相信有了这些基础，剩下的Hooks的源码阅读不会成问题，最后放上完整图示：



![img](React Hooks源码解析，原来这么简单～.assets/1-20210624144824883)