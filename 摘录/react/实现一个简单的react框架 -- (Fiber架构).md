# 实现一个简单的react框架 -- (Fiber架构)

**[react](https://codeantenna.com/tag/react)** **[reactjs](https://codeantenna.com/tag/reactjs)**

------

## 简介

本文将从头开始编写一个简单的类 `react` 框架。用于理解 `fiber` 原理和 `hooks` 的实现，轻松地深入React代码库。

## React.createElement

我们从编写`createElement`开始，这个函数主要用于把`JSX`转换成虚拟DOM（`js对象`）。这里我们使用`@babel/plugin-transform-react-jsx`这个插件自动转换。

```jsx
// jsx
const element = (
  <div id="name">
    <a>name</a>
    <b />
  </div>
)
// 编译时插件会把 jsx 转换为
const element = React.createElement(
  "div",
  { id: "name" },
  React.createElement("a", null, "name"),
  React.createElement("b")
)
```

在这里我们将属性和子节点，都放入`props`这个参数中。因为文本、数字类型不是一个对象，这里我们需要对文本对象进行包装，这样做是为了代码更加简洁（`react`为了性能用的其他方式）。

```jsx
// ----------------- React -----------------
/**
 * 创建文本节点
 * @param {*} text 
 */
function createTextElement(text) {
  return {
    type: "TEXT",
    props: {
      nodeValue: text,
      children: [],
    },
  };
}

const React = {};
// type DOM节点的标签名
// attrs 节点上的所有属性
// children 子节点
React.createElement = function (type, props, ...children) {
  return {
    type,
    props: {
      ...props,
      children: children.map((child) =>
        typeof child === "object" ? child : createTextElement(child)
      ),
    },
  };
};
```

[搭建项目详细](https://juejin.cn/post/6914902070135488520#heading-2)

## ReactDOM.render

接下来，就是实现`react`的入口`ReactDOM.render(element, container)`函数。接收一个虚拟DOM对象和一个真实DOM（容器）。主要作用就是把，虚拟DOM转换为真实DOM后放入容器内。

1. 判断虚拟DOM类型，根据类型创建真实DOM节点。
2. 把虚拟DOM的props分配给真实DOM节点。
3. 把真实DOM节点附加到容器中。
4. 有子虚拟DOM，循环子节点，递归处理每个节点。

```jsx
// ----------------- ReactDOM -----------------
const ReactDOM = {};
/**
 *
 * @param {*} vDom 虚拟DOM
 * @param {*} container 容器
 */
ReactDOM.render = function (vDom, container) {
  // 创建 真实DOM
  const dom = vDom.type == "TEXT"
      ? document.createTextNode("")
      : document.createElement(vDom.type);

  // 获取除children 外的所有属性
  const isProperty = (key) => key !== "children";
  Object.keys(vDom.props)
    .filter(isProperty)
    .forEach((name) => {
      dom[name] = vDom.props[name];
    });

  // 递归子节点
  vDom.props.children.forEach((child) => ReactDOM.render(child, dom));

  // 真实DOM 放入容器中
  container.appendChild(dom);
};


// ----------------- 使用 -----------------
ReactDOM.render(<div id="name">1111</div>,document.getElementById('root'));
```

## 使用Fiber架构

什么是Fiber架构：Fiber架构 = Fiber节点 + Fiber调度算法

1. [React fiber 架构浅析](https://blog.csdn.net/yy729851376/article/details/111819890)
2. [React Fiber详解](https://juejin.cn/post/6844903975112671239)
   这里就不详解内容，需详细了解请看以上文章。

**1. 实现浏览器在空闲的时候执行**
前面的实现中递归子节点是不能中断的。如果DOM树很大，可能会阻塞主线程，造成页面出现卡顿。
所以我们把任务分解成多个工作单元，每个单元完成后，都让浏览器判断是否有时间继续执行，时间不够就中断执行。
这里使用的是 [requestIdleCallback](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback) 浏览器每一帧都会调用这个方法，并返回剩余多少时间。当然 `react` 不是使用的这个方法，而是自己实现了功能更强大的 [Scheduler](https://github.com/facebook/react/blob/1fb18e22ae66fdb1dc127347e169e73948778e5a/packages/scheduler/README.md)。

```javascript
// 要执行的工作单元
let nextUnitOfWork = null;
/**
 * 判断是否有时间继续执行
 * @param {*} deadline
 */
function workLoop(deadline) {
  // 剩余时间判断该
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    // 返回下一个工作单元
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }

  if (nextUnitOfWork) {
    // 工作单元未执行玩 继续交给浏览器
    requestIdleCallback(workLoop);
  }
}

/**
 * 操作节点
 * @param {*} fiber
 */
function performUnitOfWork(nextUnitOfWork) {

}
```

**2. Fiber节点**
为了实现任务可中断、恢复的功能。我们需要一个数据结构：`一棵Fiber链表树`。
每一个`Fiber树节点`都能找到，下一个要执行的`节点`是什么。所以每一个节点中都有 **return指向其父Fiber节点，child指向其子Fiber节点，sibling指向其兄弟Fiber节点** 如图:
![img](assets/%E5%AE%9E%E7%8E%B0%E4%B8%80%E4%B8%AA%E7%AE%80%E5%8D%95%E7%9A%84react%E6%A1%86%E6%9E%B6%20--%20(Fiber%E6%9E%B6%E6%9E%84).assets/c7ccc9a94961ef802bd9b4765e0893b8-20220328115012480.jpeg)

那么这个**数据结构的根**就是在 `render` 中创建，然后我们将其赋值给 `nextUnitOfWork` ，就能进入循环中调用 `performUnitOfWork` 处理当前节点。

**在`performUnitOfWork`中我们就需要实现**

1. 创建真实DOM节点。
2. 真实节点放入父容器中。
3. 为每个孩子节点，创建Fiber节点。
4. 查找下一个工作单元

查找顺序是，先找子节点，没有子节点，找子节点的兄节点，都没有了找父级的兄弟节点，一直重复这个操作。 `div#root` -> `APP` -> `div.app` -> `p` -> `text` -> `span` -> `text`。

我们先修改 `render` 函数，并把创建真实DOM的功能封装为公用函数。

```javascript
// 进入
ReactDOM.render = function (element, container) {
  nextUnitOfWork = {
    dom: container,
    props: {
      children: [element],
    },
  };

  requestIdleCallback(workLoop);
}

/**
 * 创建节点
 * @param {*} fiber
 */
function createDom(fiber) {
  const dom =
    fiber.type == "TEXT"
      ? document.createTextNode("")
      : document.createElement(fiber.type);
  // 获取除children 外的所有属性
  const isProperty = (key) => key !== "children";
  Object.keys(fiber.props)
    .filter(isProperty)
    .forEach((name) => {
      dom[name] = fiber.props[name];
    });

  return dom;
}
```

然后在 `performUnitOfWork` 添加对应功能。

```javascript
/**
 * 操作节点
 * @param {*} fiber
 */
function performUnitOfWork(fiber) {
  // 创建 真实节点
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }

  // 放入父容器中
  if (fiber.parent) {
    fiber.parent.dom.appendChild(fiber.dom);
  }

  // 为每个孩子节点创建DOM
  const elements = fiber.props.children;

  let index = 0;
  // 保存兄弟节点
  let prevSibling = null;
  /**
   * 为每一个孩子节点创建fiber节点
   */
  while (index < elements.length) {
    const element = elements[index];
    // 子节点
    const newFiber = {
      type: element.type,
      props: element.props,
      parent: fiber,
      dom: null,
    };

    if (index === 0) {
      // 如果是第一个元素 就设置为 子节点
      fiber.child = newFiber;
    } else {
      // 不是第一个元素 设置为 前一个的兄弟节点
      // 给上一个节点设置兄弟节点
      prevSibling.sibling = newFiber;
    }
    // 缓存上一次 节点
    prevSibling = newFiber;
    index++;
  }

  // 查找下一个工作单元
  if (fiber.child) {
    return fiber.child;
  }
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      // 有兄弟节点就 返回兄弟节点
      return nextFiber.sibling;
    }
    // 没兄弟节点 就返回父节点  --继续循环找父节点的兄弟节点.
    nextFiber = nextFiber.parent;
  }
}
```

## 渲染和提交阶段

上面是每次处理节点，都会把创建的新节点放入DOM中。因为这个过程中是可中断的，会导致UI展示不完全。所以我们删除 `performUnitOfWork` 中真实DOM放入容器中的代码。

```js
 // if (fiber.parent) {
 //   fiber.parent.dom.appendChild(fiber.dom)
 // }
123
```

提交阶段是提交整个fiber树，所以我们需要创建一个变量保存fiber树。

```jsx
// fiber 树
let wipRoot = null
/**
 *
 * @param {*} vDom 虚拟DOM
 * @param {*} container 容器
 */
ReactDOM.render = function (element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
  };

  nextUnitOfWork = wipRoot

  requestIdleCallback(workLoop);
};
```

需要在工作单元执行完成后立即提交，在`workLoop`循环中添加判断，执行完后提交整个`fiber树`，渲染到页面上。

```javascript
/**
 * 判断是否有时间继续执行
 * @param {*} deadline
 */
function workLoop(deadline) {
  // 剩余时间判断该
  let shouldYield = false;
  while (nextUnitOfWork && !shouldYield) {
    // 返回下一个工作单元
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
    shouldYield = deadline.timeRemaining() < 1;
  }
  if (nextUnitOfWork) {
    // 工作单元未执行玩 继续交给浏览器
    requestIdleCallback(workLoop);
  }
  if (!nextUnitOfWork && wipRoot) {
    // 执行完成 提交渲染
    commitRoot()
  }
}

/**
 * 提交渲染
 */
function commitRoot() {
  commitWork(wipRoot.child)
  wipRoot = null
}
/**
 * 节点真实DOM提交到父级DOM中
 * @param {*} fiber 
 */
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  domParent.appendChild(fiber.dom)
  // 操作子节点 和 兄弟节点
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}
```

## 加入 diff 对比

要实现对比，就需要保存上一次的`fiber数据`，添加`currentRoot`变量保存。
在对比阶段是不修改节点的。所以要定义 `deletions` 收集要删除的节点，在渲染阶段删除。

```javascript
// 上一次提交的树
let currentRoot = null
// 对比后要删除的节点
let deletions = null
/**
 *
 * @param {*} element 虚拟DOM
 * @param {*} container 容器
 */
ReactDOM.render = function (element, container) {
  wipRoot = {
    dom: container,
    props: {
      children: [element],
    },
    alternate: currentRoot, // 在根节点保存上一次 提交的树
  };
  nextUnitOfWork = wipRoot;
  deletions = [];
  requestIdleCallback(workLoop);
}

/**
 * 提交渲染
 */
function commitRoot() {
  // 处理要删除的节点
  deletions.forEach(commitWork)
  // 处理节点
  commitWork(wipRoot.child)
  currentRoot = wipRoot // 保存 提交的树
  wipRoot = null
}
```

现在开始修改`performUnitOfWork`函数，添加 `reconcileChildren` 函数，实现`fiber节点`对比的操作，并在该阶段对节点打上标签，以判断节点的操作类型。

```javascript
/**
 * 操作节点
 * @param {*} fiber
 */
function performUnitOfWork(fiber) {
  // 创建 真实节点
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }

  // // 放入父容器中
  // if (fiber.parent) {
  //   fiber.parent.dom.appendChild(fiber.dom);
  // }

  // 为每个孩子节点创建DOM
  const elements = fiber.props.children;
  // 子节点对比
  reconcileChildren(fiber, elements);
  // let index = 0;
  // // 保存兄弟节点
  // let prevSibling = null;
  // /**
  //  * 为每一个孩子节点创建fiber节点
  //  */
  // while (index < elements.length) {
  //   const element = elements[index];
  //   // 子节点
  //   const newFiber = {
  //     type: element.type,
  //     props: element.props,
  //     parent: fiber,
  //     dom: null,
  //   };

  //   if (index === 0) {
  //     // 如果是第一个元素 就设置为 子节点
  //     fiber.child = newFiber;
  //   } else {
  //     // 不是第一个元素 设置为 前一个的兄弟节点
  //     // 给上一个节点设置兄弟节点
  //     prevSibling.sibling = newFiber;
  //   }
  //   // 缓存上一次 节点
  //   prevSibling = newFiber;
  //   index++;
  // }

  // 查找下一个工作单元
  if (fiber.child) {
    return fiber.child;
  }
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      // 有兄弟节点就 返回兄弟节点
      return nextFiber.sibling;
    }
    // 没兄弟节点 就返回父节点  --继续循环找父节点的兄弟节点.
    nextFiber = nextFiber.parent;
  }
}

/**
 * 对比子节点
 * @param {*} wipFiber 当前操作的节点
 * @param {*} elements 子节点
 */
function reconcileChildren(wipFiber, elements) {
  let index = 0;

  // 上一次提交的 DOM
  let oldFiber = wipFiber.alternate && wipFiber.alternate.child;

  // 保存兄弟节点
  let prevSibling = null;
  /**
   * 为每一个孩子节点创建fiber节点
   */
  // oldFiber != null 当新子节点变少  旧节点还有 持续循环 为节点打上删除标签
  while (index < elements.length || oldFiber != null) {
    const element = elements[index];
    const newFiber = null
    // 类型是否相同
    let sameType = element && oldFiber && element.type === oldFiber.type;

    // 类型相同 复用原来的节点 打上修改标签 修改props
    if (sameType) {
      newFiber = {
        type: oldFiber.type,
        props: element.props,
        dom: oldFiber.dom,
        parent: wipFiber,
        alternate: oldFiber,
        effectTag: "UPDATE",
      };
    }

    // 类型不同 有新的节点 打上新建标签 新建节点
    if (element && !sameType) {
      newFiber = {
        type: element.type,
        props: element.props,
        dom: null,
        parent: wipFiber,
        alternate: null,
        effectTag: "PLACEMENT",
      };
    }

    // 类型不同 旧节点存在 打上删除标签 
    if (oldFiber && !sameType) {
      oldFiber.effectTag = "DELETION";
      deletions.push(oldFiber);// 收集要删除的节点
    }

    // 当没有兄弟节点 赋值 为空 表示当前操作的节点 旧子节点操作完毕
    if (oldFiber) {
      oldFiber = oldFiber.sibling;
    }

    if (index === 0) {
      // 如果是第一个元素 就设置为 子节点
      wipFiber.child = newFiber;
    } else {
      // 不是第一个元素 设置为 前一个的兄弟节点
      // 给上一个节点设置兄弟节点
      prevSibling.sibling = newFiber;
    }
    // 缓存上一次 节点
    prevSibling = newFiber;
    index++;
  }
}
```

最后就是修改提交操作，通过前面定义的标签，来判断如何处理节点。

```javascript
/**
 * 修改DOM节点
 * @param {*} fiber 
 */
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  const domParent = fiber.parent.dom
  // domParent.appendChild(fiber.dom)

  if(fiber.effectTag === "PLACEMENT" &&  fiber.dom != null){
    // 新增操作
    domParent.appendChild(fiber.dom)
  }else if(fiber.effectTag === "DELETION" &&  fiber.dom != null){
    // 删除节点
    domParent.removeChild(fiber.dom);
  }else if(fiber.effectTag === "UPDATE" && fiber.dom != null){
    // 节点 修改操作
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  }
  
  // 处理兄弟节点 和子节点
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}

// 前缀是 on 返回是 true
const isEvent = (key) => key.startsWith("on");
// 去掉 children 和 on 开头的
const isProperty = (key) => key !== "children" && !isEvent(key);
// 前一次 和 本次 不同
const isNew = (prev, next) => (key) => prev[key] !== next[key];
// 过滤 匹配 下一次 中没有的值
const isGone = (prev, next) => (key) => !(key in next);
/**
 * 修改节点属性
 * @param {*} dom 当前节点的真实dom
 * @param {*} prevProps 上一次的 Props
 * @param {*} nextProps 本次Props
 */
function updateDom(dom, prevProps, nextProps) {
  // 清空 旧 事件
  Object.keys(prevProps)
    .filter(isEvent)
    .filter((key) => !(key in nextProps) || isNew(prevProps, nextProps)(key))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.removeEventListener(eventType, prevProps[name]);
    });

  // 清空 旧 的值
  Object.keys(prevProps)
    .filter(isProperty)
    .filter(isGone(prevProps, nextProps))
    .forEach((name) => {
      dom[name] = "";
    });

  // 设置 新 事件
  Object.keys(nextProps)
    .filter(isEvent)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      const eventType = name.toLowerCase().substring(2);
      dom.addEventListener(eventType, nextProps[name]);
    });

  // 设置 新 的值
  Object.keys(nextProps)
    .filter(isProperty)
    .filter(isNew(prevProps, nextProps))
    .forEach((name) => {
      if(dom instanceof Object &&  dom.setAttribute){
        dom.setAttribute(name, nextProps[name]);
      }else{
        dom[name] = nextProps[name];
      }
    });
}
```

我们实现了一个全新的属性对比方法，修改`createDom`函数，使用`updateDom`。

```javascript
/**
 * 创建节点
 * @param {*} fiber
 */
function createDom(fiber) {
  const dom =
    fiber.type == "TEXT"
      ? document.createTextNode("")
      : document.createElement(fiber.type);
  // 设置 属性
  updateDom(dom, {}, fiber.props)
  // 获取除children 外的所有属性
  // const isProperty = (key) => key !== "children";
  // Object.keys(fiber.props)
  //   .filter(isProperty)
  //   .forEach((name) => {
  //     dom[name] = fiber.props[name];
  //   });
  return dom;
}
```

## 加入组件

组件和一般的DOM有两点不同，**组件中的`fiber节点`是没有真实DOM的**，**组件的子节点是通过执行后返回的**。所以在`performUnitOfWork`函数中，添加根据`fiber节点`类型来分别处理，真实DOM创建和子DOM获取的方式。

```javascript
/**
 * 操作节点
 * @param {*} fiber
 */
function performUnitOfWork(fiber) {
  // // 创建 真实节点
  // if (!fiber.dom) {
  //   fiber.dom = createDom(fiber);
  // }

  // 是否是组件
  const isFunctionComponent = fiber.type instanceof Function;
  if (isFunctionComponent) {
    // 组件
    updateFunctionComponent(fiber);
  } else {
    updateHostComponent(fiber);
  }

  // // 放入父容器中
  // if (fiber.parent) {
  //   fiber.parent.dom.appendChild(fiber.dom);
  // }

  // // 为每个孩子节点创建DOM
  // const elements = fiber.props.children;
  // // 子节点对比
  // reconcileChildren(fiber, elements);
  // let index = 0;
  // // 保存兄弟节点
  // let prevSibling = null;
  // /**
  //  * 为每一个孩子节点创建fiber节点
  //  */
  // while (index < elements.length) {
  //   const element = elements[index];
  //   // 子节点
  //   const newFiber = {
  //     type: element.type,
  //     props: element.props,
  //     parent: fiber,
  //     dom: null,
  //   };

  //   if (index === 0) {
  //     // 如果是第一个元素 就设置为 子节点
  //     fiber.child = newFiber;
  //   } else {
  //     // 不是第一个元素 设置为 前一个的兄弟节点
  //     // 给上一个节点设置兄弟节点
  //     prevSibling.sibling = newFiber;
  //   }
  //   // 缓存上一次 节点
  //   prevSibling = newFiber;
  //   index++;
  // }

  // 查找下一个工作单元
  if (fiber.child) {
    return fiber.child;
  }
  let nextFiber = fiber;
  while (nextFiber) {
    if (nextFiber.sibling) {
      // 有兄弟节点就 返回兄弟节点
      return nextFiber.sibling;
    }
    // 没兄弟节点 就返回父节点  --继续循环找父节点的兄弟节点.
    nextFiber = nextFiber.parent;
  }
}

/**
 * 非组件 创建
 * @param {*} fiber
 */
function updateHostComponent(fiber) {
  if (!fiber.dom) {
    fiber.dom = createDom(fiber);
  }
  // 子节点对比
  reconcileChildren(fiber, fiber.props.children);
}

/**
 * 组件创建
 * @param {*} fiber
 */
function updateFunctionComponent(fiber) {
  // 运行函数 获取 子节点
  const children = [fiber.type(fiber.props)];
  // 子节点对比
  reconcileChildren(fiber, children);
}
```

接下来就是提交阶段，因为组件没有真实DOM，那么子节点就需要放入组件节点，父节点真实DOM中。
删除是同样的道理，没有真实DOM，就删除子节点的真实DOM。

```javascript 
/**
 * 修改DOM节点
 * @param {*} fiber 
 */
function commitWork(fiber) {
  if (!fiber) {
    return
  }
  // const domParent = fiber.parent.dom
  // domParent.appendChild(fiber.dom)
  // 获取父节点
  let domParentFiber = fiber.parent;
  // 判断父级 是否有 真实dom 没有继续向上找
  while (!domParentFiber.dom) {
    domParentFiber = domParentFiber.parent;
  }
  // 获取 最近 上级的 真实dom
  const domParent = domParentFiber.dom;

  if(fiber.effectTag === "ADD" &&  fiber.dom != null){
    // 新增操作
    domParent.appendChild(fiber.dom)
  }else if(fiber.effectTag === "DELETION" &&  fiber.dom != null){
    // 删除节点
    // domParent.removeChild(fiber.dom);
    commitDeletion(fiber, domParent);
  }else if(fiber.effectTag === "UPDATE" && fiber.dom != null){
    // 节点 修改操作
    updateDom(fiber.dom, fiber.alternate.props, fiber.props);
  }

  // 处理兄弟节点 和子节点
  commitWork(fiber.child)
  commitWork(fiber.sibling)
}

/**
 * 删除节点
 * @param {*} fiber 
 * @param {*} domParent 
 */
function commitDeletion(fiber, domParent){
  if(fiber.dom){
    domParent.removeChild(fiber.dom)
  }else{
    commitDeletion(fiber.child, domParent)
  }
}
```

## 加入useState

我们知道`useState`是在函数运行的时候调用的。为了实现`state`一直保存状态，我们就需要在调用之前初始化一些全局变量，还需要在`fiber节点`上初始化`hooks`数据以保存上一次的状态。

```javascript
let wipFiber = null;// 本次操作的节点
let hookIndex = null;// state的索引
/**
 * 组件创建
 * @param {*} fiber
 */
function updateFunctionComponent(fiber) {
  wipFiber = fiber
  hookIndex = 0;// hook的索引位置
  wipFiber.hooks = []

  // 运行函数 获取 子节点 并执行函数中的 useState
  const children = [fiber.type(fiber.props)];
  reconcileChildren(fiber, children);
}
```

要完整的实现 `useState` 需要
**1. 状态数据的获取和保存。**
当函数调用`useState`时，先判断之前的`fiber节点`是否有值。如果有值就使用之前的值，然后把值放入当前节点。（–注意一个函数可以有多个钩子，通过执行顺序来添加索引）

```javascript
/**
 * 钩子函数
 * @param {*} initial
 */
ReactDOM.useState = function(initial) {
  // 通过执行顺序获取 对应索引的值
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  // 之前有值 获取 之前保存的值 没值 使用默认值
  const hook = {
    state: oldHook ? oldHook.state : initial,
  }
  // 保存数据进入队列
  wipFiber.hooks.push(hook)
  // 索引增加 函数是从上向下执行的。 这就是在react useState 不能在判断中添加的原因
  hookIndex++ 
  // 返回对应的值
  return [hook.state]
}
```

**2. 修改状态动作的接收和执行。**
为了实现更新状态，我们先定义一个`setState` **接收修改状态的动作**。并把这个动作加入`hooks`的队列中。然后，我们执行与`render`函数中类似的操作，将本次操作的`fiber节点`设置为下一个工作单元，进入循环。当再次执行`useState`时，获取之前 **接收修改状态的动作** 并全部执行，修改当前状态的值，最后返回最新的状态数据。

```javascript
/**
 * 钩子函数
 * @param {*} initial
 */
ReactDOM.useState = function(initial) {
  // 通过执行顺序获取 对应索引的值
  const oldHook =
    wipFiber.alternate &&
    wipFiber.alternate.hooks &&
    wipFiber.alternate.hooks[hookIndex]
  // 之前有值 获取 之前保存的值 没值 使用默认值
  // 初始化 修改状态的数据
  const hook = {
    state: oldHook ? oldHook.state : initial,
    queue: [],
  }

  // 获取上一次 的 修改操作
  const actions = oldHook ? oldHook.queue : []
  // 执行操作 修改state
  actions.forEach(action => {
    if(action instanceof Function){
      hook.state = action(hook.state)
    }else{
      hook.state = action
    }
  })

  // 修改状态
  const setState = action => {
    // 保存操作
    hook.queue.push(action)

    // 修改后 更新下一个工作单元
    wipRoot = {
      dom: currentRoot.dom,
      props: currentRoot.props,
      alternate: currentRoot,
    }
    nextUnitOfWork = wipRoot;
    deletions = [];
    // 加入循环
    requestIdleCallback(workLoop);
  }

  // 保存数据进入队列
  wipFiber.hooks.push(hook)
  hookIndex++ // 索引增加
  // 返回最新值
  return [hook.state,setState]
}
```

测试使用

```jsx
// ----------------- 使用 -----------------
const APPS = ()=>{
  const [state, setState] = ReactDOM.useState(1)
  return (
    <h1 class="bububu" onClick={() => {setState(c => c + 1)}}>
      Count: {state}
    </h1>
  )
}
const APPP = ()=>{
    const [state, setState] = ReactDOM.useState(1)
    return (
      <h1 class={state===2 ? "sss":""} onClick={() => {setState(c => c + 1)}}>
        Countsss: {state}
      </h1>
    )
  }
  const APP = ()=>{
    return (
      <div>
          <APPS />
          <APPP />
      </div>
    )
  }
ReactDOM.render( <APP />,document.getElementById('root'));
```

源码地址: https://github.com/nie-ny/react-simple/tree/main/react-simulation-fiber