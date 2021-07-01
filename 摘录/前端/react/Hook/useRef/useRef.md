# useRef

>跨越渲染周期存储数据，而且对它修改也不会引起组件渲染



## 使用

> `useRef` 只会回传一个值，这个值是一个有 `current`属性 的 Object 。**当更新 current 值并不会触发 re-render**

```
const renderCount = useRef(0);  
// renderCount = { current: 0 }
```



跟 `useState`不同， `useState` 会回传一个包含两个值的array，第一个值是state、第二個值是用來更新 state 的函式。**每当 renderCount 值改变，就会触发 re-render**

```
const [renderCount, setRenderCount] = useState(0);
```

`useRef` 只会回传一个值，这个值是一个有 `current`属性 的 Object 。**当更新 current 值并不会触发 re-render**

```
const renderCount = useRef(0);  
// renderCount = { current: 0 }
```

[资料](https://medium.com/hannah-lin/react-hook-%E7%AD%86%E8%A8%98-useref-c628cbf0d7fb)

