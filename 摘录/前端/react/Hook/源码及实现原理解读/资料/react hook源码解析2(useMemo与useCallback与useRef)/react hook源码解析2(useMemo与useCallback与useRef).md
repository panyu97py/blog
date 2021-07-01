## hooks的两种调用阶段mount和update

> hooks的源码是放在https://github.com/facebook/react/blob/master/packages/react-reconciler/src/ReactFiberHooks.js这里

从源码可以看出，react把所有的hooks分成了两个阶段的hooks

1. mount阶段对应第一次渲染初始化时候调用的hooks方法,分别对应了`mountMemo`,`mountCallback`,`mountRef`,`以及其他hooks。
2. update阶段对应setXXX函数触发更新重新渲染的更新阶段,分别对应了`updateMemo`,`updateCallback`,`updateRef`, `updateLayoutEffect`以及其他hooks

```ts
// react-reconciler/src/ReactFiberHooks.js
// Mount 阶段Hooks的定义
const HooksDispatcherOnMount: Dispatcher = {
  useCallback: mountCallback,
  useMemo: mountMemo,
 // 其他Hooks
};

// Update阶段Hooks的定义
const HooksDispatcherOnUpdate: Dispatcher = {
  useCallback: updateCallback,
  useMemo: updateMemo,
  // 其他Hooks
};
```

hooks在mount阶段和update阶段所调用的逻辑是不一样的，在上一篇中我们了解了hook的一些通用化操作，接下来我们将直接探究useMemo，useCallback是如何做缓存的。

## useMemo

### useMemo用法

useMemo是用来做性能优化的，只要传入的依赖内部元素没有发生变化那么返回的值或者引用不会发生变化，只要val不发生变化，那么返回的state引用还是一样的，这样子Child组件就不会重复渲染。

```ts
const val = useState(1)
const state = useMemo(() => {val: 1}, [val])
return (
  <Child state={state}/>
)
```

### mountMemo

1. 跟其他hook一样，在mount阶段直接创建一个hook对象通过.next拼接在fiberNode的hook链表上
2. mount初始化的时候直接调用传入的函数获取需要缓存的值然后直接返回，这样子我们在`const state = useMemo(() => {val: 1}, [val])`就可以拿到相应的值。
3. 跟useState不一样的是：useState是直接保存值到hook的memoizedState属性上，useMemo保存的是一个长度为2的数组，值分别是上面调用后的值以及传入的依赖

```ts
// [react-reconciler/src/ReactFiberHooks.js](https://github.com/facebook/react/blob/be4c8b19c16a7558e2939a6665399f6f3202668e/packages/react-reconciler/src/ReactFiberHooks.js#L1389)
function mountMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  // 创建hook对象拼接在hook链表上
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  // 调用我们传入的函数 获取需要缓存的值
  const nextValue = nextCreate();
  //与useState直接保存值的不同 useMemo保存在memoizedState的是一个数组
  // 第一个值是需要缓存的值 第二个是传入的依赖
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}
```

### updateMemo

1. 跟useState一样通过`updateWorkInProgressHook`获取更新时当前的useMemo的hook对象。
2. 如果上一次的useMemo值(memoizedState)不为空并且这一次传入的依赖（nextDeps）不为空，那么两次依赖做浅比较。
3. 依赖没有发生变化，那么直接返回数组第一个值即上一次渲染的值。如果依赖发生变化重新调用函数生成新的值

```ts
function updateMemo<T>(
  nextCreate: () => T,
  deps: Array<mixed> | void | null,
): T {
  // 第一篇已经说过 获取相对应的hook对象
  const hook = updateWorkInProgressHook();
  // 获取更新时的依赖
  const nextDeps = deps === undefined ? null : deps;
  // 获取上一次的数组值
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    // Assume these are defined. If they're not, areHookInputsEqual will warn.
    // 如果这次传入的依赖不为空做浅比较 如果依赖没有发生变化那么直接返回上一次的值
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  // 依赖发生变化 重新调用生成新的值
  const nextValue = nextCreate();
  hook.memoizedState = [nextValue, nextDeps];
  return nextValue;
}

// 浅比较dep的函数
function areHookInputsEqual(
  nextDeps: Array<mixed>,
  prevDeps: Array<mixed> | null,
) {
  if (prevDeps === null) {
    return false;
  }

  // 浅比较
  for (let i = 0; i < prevDeps.length && i < nextDeps.length; i++) {
    if (is(nextDeps[i], prevDeps[i])) {
      continue;
    }
    return false;
  }
  return true;
}
```

### useMemo总结

其实useMemo跟普通的hook逻辑基本一致，差异是在useState的hook对象memoizedState保存的是每次执行updateAction直接返回的值。但是useMemo保存的是一个数组分别是需要缓存的值以及依赖。在更新阶段会比对上一次以及当前的依赖做出是否直接返回上一次渲染的值。

## useCallback

useCallback也是用来做性能优化的，只要传入的依赖内部元素没有发生变化，那么返回的函数引用相同。在下面中只要val不发生变化，那么返回的handleClick函数的引用就不会发生变化，这样子Child组件就不会重复渲染。

```ts
const val = useState(1)
const handleClick = useCallback(() => console.log(val), [val])
return (
  <Child onClick={handleClick}/>
)
```

### useCallback源码实现

useCallback源码实现跟useMemo基本上完全一模一样，不同的是useMemo会调用函数获取缓存的值，而useCallck保存的函数所以不需要调用。其余代码一模一样，这里就不做赘述了直接贴源码。

```ts
//https://github.com/facebook/react/blob/be4c8b19c16a7558e2939a6665399f6f3202668e/packages/react-reconciler/src/ReactFiberHooks.js#L1366
function mountCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = mountWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  //useCallback缓存的是函数 直接保存的就是函数引用
  hook.memoizedState = [callback, nextDeps];
  return callback;
}

function updateCallback<T>(callback: T, deps: Array<mixed> | void | null): T {
  const hook = updateWorkInProgressHook();
  const nextDeps = deps === undefined ? null : deps;
  const prevState = hook.memoizedState;
  if (prevState !== null) {
    if (nextDeps !== null) {
      const prevDeps: Array<mixed> | null = prevState[1];
      if (areHookInputsEqual(nextDeps, prevDeps)) {
        return prevState[0];
      }
    }
  }
  hook.memoizedState = [callback, nextDeps];
  return callback;
}
```

## useRef

useRef源码也是十分简洁，本质上就是保存的是一个对象，对象内部有一个current值。每次更新的时候直接把这个对象返回出来，注意的是useRef内的值的改变不会触发react的调度更新。

```ts
function mountRef<T>(initialValue: T): {|current: T|} {
  const hook = mountWorkInProgressHook();
  const ref = {current: initialValue};
  if (__DEV__) {
    Object.seal(ref);
  }
  hook.memoizedState = ref;
  return ref;
}

function updateRef<T>(initialValue: T): {|current: T|} {
  const hook = updateWorkInProgressHook();
  return hook.memoizedState;
}
```