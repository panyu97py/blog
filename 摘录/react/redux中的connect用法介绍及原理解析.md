# redux中的connect用法介绍及原理解析

react-redux 提供了两个重要的对象， `Provider` 和 `connect` ，前者使 React 组件可被连接（connectable），后者把 React 组件和 Redux 的 store 真正连接起来。react-redux 的文档中，对 `connect` 的描述是一段晦涩难懂的英文，在初学 redux 的时候，我对着这段文档阅读了很久，都没有全部弄明白其中的意思（大概就是，单词我都认识，连起来啥意思就不明白了的感觉吧）。

在使用了一段时间 redux 后，本文尝试再次回到这里，给 [这段文档](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Freactjs%2Freact-redux%2Fblob%2Fmaster%2Fdocs%2Fapi.md%23connectmapstatetoprops-mapdispatchtoprops-mergeprops-options) （同时摘抄在附录中）一个靠谱的解读。



关于react-redux的一个流程图![img](redux中的connect用法介绍及原理解析.assets/89fcfd2752d0b0e031cb57f8494c79ef~tplv-t2oaga2asx-watermark.awebp)

流程图

connect用法介绍

connect方法声明：

```
connect([mapStateToProps], [mapDispatchToProps], [mergeProps],[options])
```

作用：连接React组件与 Redux store。

参数说明：

```
mapStateToProps(state, ownProps) : stateProps
```

这个函数允许我们将 store 中的数据作为 props 绑定到组件上。

```
const mapStateToProps = (state) => {
  return {
    count: state.count
  }
}
```

（1）这个函数的第一个参数就是 Redux 的 store，我们从中摘取了 count 属性。你不必将 state 中的数据原封不动地传入组件，可以根据 state 中的数据，动态地输出组件需要的（最小）属性。

（2）函数的第二个参数 ownProps，是组件自己的 props。有的时候，ownProps 也会对其产生影响。

当 state 变化，或者 ownProps 变化的时候，mapStateToProps 都会被调用，计算出一个新的 stateProps，（在与 ownProps merge 后）更新给组件。

```
mapDispatchToProps(dispatch, ownProps): dispatchProps
```

connect 的第二个参数是 mapDispatchToProps，它的功能是，将 action 作为 props 绑定到组件上，也会成为 MyComp 的 props。

```
[mergeProps],[options]
```

不管是 stateProps 还是 dispatchProps，都需要和 ownProps merge 之后才会被赋给组件。connect 的第三个参数就是用来做这件事。通常情况下，你可以不传这个参数，connect 就会使用 Object.assign 替代该方法。

[options] (Object) 如果指定这个参数，可以定制 connector 的行为。一般不用。

原理解析

首先connect之所以会成功，是因为Provider组件：

- 在原应用组件上包裹一层，使原来整个应用成为Provider的子组件
- 接收Redux的store作为props，通过context对象传递给子孙组件上的connect

那connect做了些什么呢？

它真正连接 Redux 和 React，它包在我们的容器组件的外一层，它接收上面 Provider 提供的 store 里面的 state 和 dispatch，传给一个构造函数，返回一个对象，以属性形式传给我们的容器组件。

关于它的源码

connect是一个高阶函数，首先传入mapStateToProps、mapDispatchToProps，然后返回一个生产Component的函数(wrapWithConnect)，然后再将真正的Component作为参数传入wrapWithConnect，这样就生产出一个经过包裹的Connect组件，该组件具有如下特点:

- 通过props.store获取祖先Component的store
- props包括stateProps、dispatchProps、parentProps,合并在一起得到nextState，作为props传给真正的Component
- componentDidMount时，添加事件this.store.subscribe(this.handleChange)，实现页面交互
- shouldComponentUpdate时判断是否有避免进行渲染，提升页面性能，并得到nextState
- componentWillUnmount时移除注册的事件this.handleChange

由于connect的源码过长，我们只看主要逻辑：

```
export default function connect(mapStateToProps, mapDispatchToProps, mergeProps, options = {}) {
  return function wrapWithConnect(WrappedComponent) {
    class Connect extends Component {
      constructor(props, context) {
        // 从祖先Component处获得store
        this.store = props.store || context.store
        this.stateProps = computeStateProps(this.store, props)
        this.dispatchProps = computeDispatchProps(this.store, props)
        this.state = { storeState: null }
        // 对stateProps、dispatchProps、parentProps进行合并
        this.updateState()
      }
      shouldComponentUpdate(nextProps, nextState) {
        // 进行判断，当数据发生改变时，Component重新渲染
        if (propsChanged || mapStateProducedChange || dispatchPropsChanged) {
          this.updateState(nextProps)
            return true
          }
        }
        componentDidMount() {
          // 改变Component的state
          this.store.subscribe(() = {
            this.setState({
              storeState: this.store.getState()
            })
          })
        }
        render() {
          // 生成包裹组件Connect
          return (
            <WrappedComponent {...this.nextState} />
          )
        }
      }
      Connect.contextTypes = {
        store: storeShape
      }
      return Connect;
    }
  }
```

connect使用实例

这里我们写一个关于计数器使用的实例：

Component/Counter.js

```
import React, {Component} from 'react'

class Counter extends Component {
    render() {
        //从组件的props属性中导入四个方法和一个变量
        const {increment, decrement, counter} = this.props;
        //渲染组件，包括一个数字，四个按钮
        return (
            <p>
                Clicked: {counter} times
                {' '}
                <button onClick={increment}>+</button>
                {' '}
                <button onClick={decrement}>-</button>
                {' '}
            </p>
        )
    }
}

export default Counter;
```

Container/App.js

```
import { connect } from 'react-redux'
import Counter from '../components/Counter'
import actions from '../actions/counter';

//将state.counter绑定到props的counter
const mapStateToProps = (state) => {
    return {
        counter: state.counter
    }
};
//将action的所有方法绑定到props上
const mapDispatchToProps = (dispatch, ownProps) => {
    return {
        increment: (...args) => dispatch(actions.increment(...args)),
        decrement: (...args) => dispatch(actions.decrement(...args))
    }
};

//通过react-redux提供的connect方法将我们需要的state中的数据和actions中的方法绑定到props上
export default connect(mapStateToProps, mapDispatchToProps)(Counter)
```





首先回顾一下 redux 的基本用法:

```
connect方法声明如下：
connect([mapStateToProps], [mapDispatchToProps], [mergeProps],[options])  

作用：连接 React 组件与 Redux store。 
连接操作不会改变原来的组件类，反而返回一个新的已与 Redux store 连接的组件类。 

返回值
根据配置信息，返回一个注入了 state 和 action creator 的 React 组件。

```



[mapStateToProps(state, [ownProps]): stateProps] (Function): 如果定义该参数，组件将会监听 Redux store 的变化。任何时候，只要 Redux store 发生改变，mapStateToProps 函数就会被调用。该回调函数必须返回一个纯对象，这个对象会与组件的 props 合并。如果你省略了这个参数，你的组件将不会监听 Redux store。如果指定了该回调函数中的第二个参数 ownProps，则该参数的值为传递到组件的 props，而且只要组件接收到新的 props，mapStateToProps 也会被调用。





[mapDispatchToProps(dispatch, [ownProps]): dispatchProps] (Object or Function): 如果传递的是一个对象，那么每个定义在该对象的函数都将被当作 Redux action creator，而且这个对象会与 Redux store 绑定在一起，其中所定义的方法名将作为属性名，合并到组件的 props 中。如果传递的是一个函数，该函数将接收一个 dispatch 函数，然后由你来决定如何返回一个对象，这个对象通过 dispatch 函数与 action creator 以某种方式绑定在一起（提示：你也许会用到 Redux 的辅助函数 bindActionCreators()）。如果你省略这个 mapDispatchToProps 参数，默认情况下，dispatch 会注入到你的组件 props 中。如果指定了该回调函数中第二个参数 ownProps，该参数的值为传递到组件的 props，而且只要组件接收到新 props，mapDispatchToProps 也会被调用。





[mergeProps(stateProps, dispatchProps, ownProps): props] (Function): 如果指定了这个参数，mapStateToProps() 与 mapDispatchToProps() 的执行结果和组件自身的 props 将传入到这个回调函数中。该回调函数返回的对象将作为 props 传递到被包装的组件中。你也许可以用这个回调函数，根据组件的 props 来筛选部分的 state 数据，或者把 props 中的某个特定变量与 action creator 绑定在一起。如果你省略这个参数，默认情况下返回 Object.assign({}, ownProps, stateProps, dispatchProps) 的结果。





[options] (Object) 如果指定这个参数，可以定制 connector 的行为。





[pure = true] (Boolean): 如果为 true，connector 将执行 shouldComponentUpdate 并且浅对比 mergeProps 的结果，避免不必要的更新，前提是当前组件是一个“纯”组件，它不依赖于任何的输入或 state 而只依赖于 props 和 Redux store 的 state。默认值为 true。[withRef = false] (Boolean): 如果为 true，connector 会保存一个对被包装组件实例的引用，该引用通过 getWrappedInstance() 方法获得。默认值为 false





> 静态属性
>
> WrappedComponent (Component): 传递到 connect() 函数的原始组件类。
>
> 静态方法
>
> 组件原来的静态方法都被提升到被包装的 React 组件。
>
> 实例方法
>
> getWrappedInstance(): ReactComponent
>
> 仅当 connect() 函数的第四个参数 options 设置了 { withRef: true } 才返回被包装的组件实例。

> 备注：
>
> 1、 函数将被调用两次。第一次是设置参数，第二次是组件与 Redux store 连接：connect(mapStateToProps, mapDispatchToProps, mergeProps)(MyComponent)。
>
> 2、 connect 函数不会修改传入的 React 组件，返回的是一个新的已与 Redux store 连接的组件，而且你应该使用这个新组件。
>
> 3、 mapStateToProps 函数接收整个 Redux store 的 state 作为 props，然后返回一个传入到组件 props 的对象。该函数被称之为 selector。





```
const reducer = (state = {count: 0}, action) => {
  switch (action.type){
    case 'INCREASE': return {count: state.count + 1};
    case 'DECREASE': return {count: state.count - 1};
    default: return state;
  }
}

const actions = {
  increase: () => ({type: 'INCREASE'}),
  decrease: () => ({type: 'DECREASE'})
}

const store = createStore(reducer);

store.subscribe(() =>
  console.log(store.getState())
);

store.dispatch(actions.increase()) // {count: 1}
store.dispatch(actions.increase()) // {count: 2}
store.dispatch(actions.increase()) // {count: 3}

```

通过 `reducer` 创建一个 `store` ，每当我们在 `store` 上 `dispatch` 一个 `action` ， `store` 内的数据就会相应地发生变化。

我们当然可以 **直接** 在 React 中使用 Redux：在最外层容器组件中初始化 `store` ，然后将 `state` 上的属性作为 `props` 层层传递下去。

```
class App extends Component{

  componentWillMount(){
    store.subscribe((state)=>this.setState(state))
  }

  render(){
    return <Comp state={this.state}
                 onIncrease={()=>store.dispatch(actions.increase())}
                 onDecrease={()=>store.dispatch(actions.decrease())}
    />
  }
}

```

但这并不是最佳的方式。最佳的方式是使用 react-redux 提供的 `Provider` 和 `connect` 方法。

## 使用 react-redux

首先在最外层容器中，把所有内容包裹在 `Provider` 组件中，将之前创建的 `store` 作为 `prop`传给 `Provider` 。

```
const App = () => {
  return (
    <Provider store={store}>
      <Comp/>
    </Provider>
  )
};

```

`Provider` 内的任何一个组件（比如这里的 `Comp` ），如果需要使用 `state` 中的数据，就必须是「被 connect 过的」组件——使用 `connect` 方法对「你编写的组件（ `MyComp` ）」进行包装后的产物。

```
class MyComp extends Component {
  // content...
}

const Comp = connect(...args)(MyComp);

```

可见， `connect` 方法是重中之重。

## connect 详解

究竟 `connect` 方法到底做了什么，我们来一探究竟。

首先看下函数的签名：

connect([mapStateToProps], [mapDispatchToProps], [mergeProps], [options])

`connect()` 接收四个参数，它们分别是 `mapStateToProps` ， `mapDispatchToProps` ， `mergeProps` 和 `options` 。

```
mapStateToProps(state, ownProps) : stateProps
```

这个函数允许我们将 `store` 中的数据作为 `props` 绑定到组件上。

```
const mapStateToProps = (state) => {
  return {
    count: state.count
  }
}

```

这个函数的第一个参数就是 Redux 的 `store` ，我们从中摘取了 `count` 属性。因为返回了具有 `count` 属性的对象，所以 `MyComp` 会有名为 `count` 的 `props` 字段。

```
class MyComp extends Component {
  render(){
    return <div>计数：{this.props.count}次</div>
  }
}

const Comp = connect(...args)(MyComp);

```

当然，你不必将 `state` 中的数据原封不动地传入组件，可以根据 `state` 中的数据，动态地输出组件需要的（最小）属性。

```
const mapStateToProps = (state) => {
  return {
    greaterThanFive: state.count > 5
  }
}

```

函数的第二个参数 `ownProps` ，是 `MyComp` 自己的 `props` 。有的时候， `ownProps` 也会对其产生影响。比如，当你在 `store` 中维护了一个用户列表，而你的组件 `MyComp` 只关心一个用户（通过 `props` 中的 `userId` 体现）。

```
const mapStateToProps = (state, ownProps) => {
  // state 是 {userList: [{id: 0, name: '王二'}]}
  return {
    user: _.find(state.userList, {id: ownProps.userId})
  }
}

class MyComp extends Component {
  
  static PropTypes = {
    userId: PropTypes.string.isRequired,
    user: PropTypes.object
  };
  
  render(){
    return <div>用户名：{this.props.user.name}</div>
  }
}

const Comp = connect(mapStateToProps)(MyComp);

```

当 `state` 变化，或者 `ownProps` 变化的时候， `mapStateToProps` 都会被调用，计算出一个新的 `stateProps` ，（在与 `ownProps` merge 后）更新给 `MyComp` 。

这就是将 Redux `store` 中的数据连接到组件的基本方式。

```
mapDispatchToProps(dispatch, ownProps): dispatchProps
```

`connect` 的第二个参数是 `mapDispatchToProps` ，它的功能是，将 action 作为 `props` 绑定到 `MyComp` 上。

```
const mapDispatchToProps = (dispatch, ownProps) => {
  return {
    increase: (...args) => dispatch(actions.increase(...args)),
    decrease: (...args) => dispatch(actions.decrease(...args))
  }
}

class MyComp extends Component {
  render(){
    const {count, increase, decrease} = this.props;
    return (<div>
      <div>计数：{this.props.count}次</div>
      <button onClick={increase}>增加</button>
      <button onClick={decrease}>减少</button>
    </div>)
  }
}

const Comp = connect(mapStateToProps， mapDispatchToProps)(MyComp);

```

由于 `mapDispatchToProps` 方法返回了具有 `increase` 属性和 `decrease` 属性的对象，这两个属性也会成为 `MyComp` 的 `props` 。

如上所示，调用 `actions.increase()` 只能得到一个 `action` 对象 `{type:'INCREASE'}` ，要触发这个 `action` 必须在 `store` 上调用 `dispatch` 方法。 `diapatch` 正是 `mapDispatchToProps` 的第一个参数。但是，为了不让 `MyComp` 组件感知到 `dispatch` 的存在，我们需要将 `increase` 和 `decrease` 两个函数包装一下，使之成为直接可被调用的函数（即，调用该方法就会触发 `dispatch`）。

Redux 本身提供了 `bindActionCreators` 函数，来将 action 包装成直接可被调用的函数。

```
import {bindActionCreators} from 'redux';

const mapDispatchToProps = (dispatch, ownProps) => {
  return bindActionCreators({
    increase: action.increase,
    decrease: action.decrease
  });
}

```

同样，当 `ownProps` 变化的时候，该函数也会被调用，生成一个新的 `dispatchProps` ，（在与 `statePrope` 和 `ownProps` merge 后）更新给 `MyComp` 。注意， `action` 的变化不会引起上述过程，默认 `action` 在组件的生命周期中是固定的。

```
[mergeProps(stateProps, dispatchProps, ownProps): props]
```

之前说过，不管是 `stateProps` 还是 `dispatchProps` ，都需要和 `ownProps` merge 之后才会被赋给 `MyComp` 。 `connect` 的第三个参数就是用来做这件事。通常情况下，你可以不传这个参数， `connect` 就会使用 `Object.assign` 替代该方法。

其他

最后还有一个 `options` 选项，比较简单，基本上也不大会用到（尤其是你遵循了其他的一些 React 的「最佳实践」的时候），本文就略过了。希望了解的同学可以直接看文档。



## 1. 前言

随着WEB应用变得越来越复杂，再加上node前后端分离越来越流行，那么对数据流动的控制就显得越发重要。redux是在flux的基础上产生的，基本思想是保证数据的单向流动，同时便于控制、使用、测试。

redux不依赖于任意框架(库)，只要subscribe相应框架(库)的内部方法，就可以使用该应用框架保证数据流动的一致性。

那么如何使用redux呢？下面一步步进行解析，并带有源码说明，不仅做到 知其然 ，还要做到 知其所以然 。

## 2. 主干逻辑介绍(createStore)

2.1 简单demo入门

先来一个直观的认识：

```
// 首先定义一个改变数据的plain函数，成为reducer
function count (state, action) {
  var defaultState = {
    year: 2015,
  };
  state = state || defaultState;
  switch (action.type) {
    case 'add':
      return {
        year: state.year + 1
      };
    case 'sub':
      return {
        year: state.year - 1
      }
    default :
      return state;
    }
  }

// store的创建
var createStore = require('redux').createStore;
var store = createStore(count);

// store里面的数据发生改变时，触发的回调函数
store.subscribe(function () {
  console.log('the year is: ', store.getState().year);
});

// action: 触发state改变的唯一方法(按照redux的设计思路)
var action1 = { type: 'add' };
var action2 = { type: 'add' };
var action3 = { type: 'sub' };

// 改变store里面的方法
store.dispatch(action1); // 'the year is: 2016
store.dispatch(action2); // 'the year is: 2017
store.dispatch(action3); // 'the year is: 2016
```

2.2 挖掘createStore实现

为了说明主要问题，仅列出其中的关键代码，全部代码，可以点击 [这里](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Frackt%2Fredux%2Fblob%2Fmaster%2Fsrc%2FcreateStore.js) 阅读。

a首先看createStore到底都返回的内容:

```
export default function createStore(reducer, initialState) {
  ...
  return {
    dispatch,
    subscribe,
    getState,
    replaceReducer
  }
}
```

每个属性的含义是: - dispatch: 用于action的分发，改变store里面的state - subscribe: 注册listener，store里面state发生改变后，执行该listener - getState: 读取store里面的state - replaceReducer: 替换reducer，改变state修改的逻辑

b关键代码解析

```
export default function createStore(reducer, initialState) {
  // 这些都是闭包变量
  var currentReducer = reducer
  var currentState = initialState
  var listeners = []
  var isDispatching = false;

  // 返回当前的state
  function getState() {
    return currentState
  }

  // 注册listener，同时返回一个取消事件注册的方法
  function subscribe(listener) {
    listeners.push(listener)
    var isSubscribed = true

    return function unsubscribe() {
    if (!isSubscribed) {
	   return
    }

    isSubscribed = false
    var index = listeners.indexOf(listener)
      listeners.splice(index, 1)
    }
  }
  // 通过action该改变state，然后执行subscribe注册的方法
  function dispatch(action) {
    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }
    listeners.slice().forEach(listener => listener())
    return action
  }
  
  // 替换reducer，修改state变化的逻辑
  function replaceReducer(nextReducer) {
    currentReducer = nextReducer
    dispatch({ type: ActionTypes.INIT })
  }
  // 初始化时，执行内部一个dispatch，得到初始state
  dispatch({ type: ActionTypes.INIT })
}
```

如果还按照2.1的方式进行开发，那跟flux没有什么大的区别，需要手动解决很多问题，那redux如何将整个流程模板化(Boilerplate)呢?

## 3. 保证store的唯一性

随着应用越来越大，一方面，不能把所有的数据都放到一个reducer里面，另一方面，为每个reducer创建一个store，后续store的维护就显得比较麻烦。如何将二者统一起来呢？

3.1 demo入手

通过combineReducers将多个reducer合并成一个rootReducer: // 创建两个reducer: count year function count (state, action) { state = state || {count: 1} switch (action.type) { default: return state; } } function year (state, action) { state = state || {year: 2015} switch (action.type) { default: return state; } }

```
// 将多个reducer合并成一个
var combineReducers = require('./').combineReducers;
var rootReducer = combineReducers({
  count: count,
  year: year,
});

// 创建store，跟2.1没有任何区别
var createStore = require('./').createStore;
var store = createStore(rootReducer);

var util = require('util');
console.log(util.inspect(store));
//输出的结果，跟2.1的store在结构上不存在区别
// { dispatch: [Function: dispatch],
//   subscribe: [Function: subscribe],
//   getState: [Function: getState],
//   replaceReducer: [Function: replaceReducer]
// }
```

3.2 源码解析combineReducers

```
// 高阶函数，最后返回一个reducer
export default function combineReducers(reducers) {
  // 提出不合法的reducers, finalReducers就是一个闭包变量
  var finalReducers = pick(reducers, (val) => typeof val === 'function')
  // 将各个reducer的初始state均设置为undefined
  var defaultState = mapValues(finalReducers, () => undefined)

  // 一个总reducer，内部包含子reducer
  return function combination(state = defaultState, action) {
    var finalState = mapValues(finalReducers, (reducer, key) => {
      var previousStateForKey = state[key]
      var nextStateForKey = reducer(previousStateForKey, action)
      hasChanged = hasChanged || nextStateForKey !== previousStateForKey
      return nextStateForKey
    );
    return hasChanged ? finalState : state
  }
  
}
```

## 4. 自动实现dispatch

4.1 demo介绍

在2.1中，要执行state的改变，需要手动dispatch:

```
var action = { type: '***', payload: '***'};
dispatch(action);
```

手动dispatch就显得啰嗦了，那么如何自动完成呢?

```
var bindActionCreators = require('redux').bindActionCreators;
// 可以在具体的应用框架隐式进行该过程(例如react-redux的connect组件中)
bindActionCreators(action)
```

4.2 源码解析

```
// 隐式实现dispatch
function bindActionCreator(actionCreator, dispatch) {
  return (...args) => dispatch(actionCreator(...args))
}

export default function bindActionCreators(actionCreators, dispatch) {
  if (typeof actionCreators === 'function') {
    return bindActionCreator(actionCreators, dispatch)
  }
  return mapValues(actionCreators, actionCreator =>
    bindAQctionCreator(actionCreator, dispatch)
  )
}
```

## 5. 支持插件 - 对dispatch的改造

5.1 插件使用demo

一个action可以是同步的，也可能是异步的，这是两种不同的情况， dispatch执行的时机是不一样的:

```
// 同步的action creator, store可以默认实现dispatch
function add() {
  return { tyle: 'add' }
}
dispatch(add());

// 异步的action creator，因为异步完成的时间不确定，只能手工dispatch
function fetchDataAsync() {
  return function (dispatch) {
    requst(url).end(function (err, res) {
      if (err) return dispatch({ type: 'SET_ERR', payload: err});
      if (res.status === 'success') {
        dispatch({ type: 'FETCH_SUCCESS', payload: res.data });
      }
    })
  }
}
```

下面的问题就变成了，如何根据实际情况实现不同的dispatch方法，也即是根据需要实现不同的moddleware:

```
// 普通的dispatch创建方法
var store = createStore(reducer, initialState);
console.log(store.dispatch);

// 定制化的dispatch
var applyMiddleware = require('redux').applyMiddleware;
// 实现action异步的middleware
var thunk = requre('redux-thunk');
var store = applyMiddleware([thunk])(createStore);
// 经过处理的dispatch方法
console.log(store.dispatch);
```

5.2 源码解析

```
// next: 其实就是createStore
export default function applyMiddleware(...middlewares) {
  return (next) => (reducer, initialState) => {
    var store = next(reducer, initialState)
    var dispatch = store.dispatch
    var chain = []

    var middlewareAPI = {
      getState: store.getState,
      dispatch: (action) => dispatch(action)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch // 实现新的dispatch方法
    }
  }
}
// 再看看redux-thunk的实现, next就是store里面的上一个dispatch
function thunkMiddleware({ dispatch, getState }) {
  return function(next) {
    return function(action) {
      typeof action === 'function' ? action(dispatch, getState) : next(action);
    }
  }
  
  return next => action =>
    typeof action === 'function' ?
      action(dispatch, getState) :
      next(action);
 }
```

## 6. 与react框架的结合

6.1 基本使用

目前已经有现成的工具 react-redux 来实现二者的结合:

```
var rootReducers = combineReducers(reducers);
var store = createStore(rootReducers);
var Provider = require('react-redux').Provider;
// App 为上层的Component
class App extend React.Component{
  render() {
    return (
      <Provier store={store}>
        <Container />
      </Provider>
    );
  }
}

// Container作用: 1. 获取store中的数据; 2.将dispatch与actionCreator结合起来
var connect = require('react-redux').connect;
var actionCreators = require('...');
// MyComponent是与redux无关的组件
var MyComponent = require('...');

function select(state) {
  return {
    count: state.count
  }
}
export default connect(select, actionCreators)(MyComponent)
```

6.2 Provider – 提供store

React通过Context属性，可以将属性(props)直接给子孙component，无须通过props层层传递, Provider仅仅起到获得store，然后将其传递给子孙元素而已:

```
export default class Provider extends Component {
  getChildContext() { // getChildContext: 将store传递给子孙component
    return { store: this.store }
  }

  constructor(props, context) {
    super(props, context)
    this.store = props.store
  }

  componentWillReceiveProps(nextProps) {
    const { store } = this
    const { store: nextStore } = nextProps

    if (store !== nextStore) {
      warnAboutReceivingStore()
    }
  }

  render() {
    let { children } = this.props
    return Children.only(children)
  }
}
```

6.3 connect – 获得store及dispatch(actionCreator)

connect是一个高阶函数，首先传入mapStateToProps、mapDispatchToProps，然后返回一个生产 Component 的函数(wrapWithConnect)，然后再将真正的Component作为参数传入wrapWithConnect(MyComponent)，这样就生产出一个经过包裹的Connect组件，该组件具有如下特点:

- 通过this.context获取祖先Component的store
- props包括stateProps、dispatchProps、parentProps,合并在一起得到 nextState ，作为props传给真正的Component
- componentDidMount时，添加事件this.store.subscribe(this.handleChange)，实现页面交互
- shouldComponentUpdate时判断是否有避免进行渲染，提升页面性能，并得到nextState
- componentWillUnmount时移除注册的事件this.handleChange
- 在非生产环境下，带有热重载功能

主要的代码逻辑:

```
export default function connect(mapStateToProps, mapDispatchToProps, mergeProps, options = {}) {
  return function wrapWithConnect(WrappedComponent) {
    class Connect extends Component {
      constructor(props, context) {
        // 从祖先Component处获得store
        this.store = props.store || context.store
        this.stateProps = computeStateProps(this.store, props)
        this.dispatchProps = computeDispatchProps(this.store, props)
        this.state = { storeState: null }
        // 对stateProps、dispatchProps、parentProps进行合并
        this.updateState()
      }
      shouldComponentUpdate(nextProps, nextState) {
        // 进行判断，当数据发生改变时，Component重新渲染
        if (propsChanged || mapStateProducedChange || dispatchPropsChanged) {
          this.updateState(nextProps)
            return true
          }
        }
        componentDidMount() {
          // 改变Component的state
          this.store.subscribe(() = {
            this.setState({
              storeState: this.store.getState()
            })
          })
        }
        render() {
          // 生成包裹组件Connect
          return (
            <WrappedComponent {...this.nextState} />
          )
        }
      }
      Connect.contextTypes = {
        store: storeShape
      }
      return Connect;
    }
  }
```

## 7. redux与react-redux关系图

![img](redux中的connect用法介绍及原理解析.assets/a8decf114c59f9180157bd50f7ed705e~tplv-t2oaga2asx-watermark.awebp)