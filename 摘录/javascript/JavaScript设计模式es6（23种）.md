每一种模式都是查阅各资料, 代码测试及思考总结而出，本文较长，希望对你有所帮助，如果对你有用，请点赞支持一把，也是给予我写作的动力

## 设计模式简介

设计模式代表了最佳的实践，通常被有经验的面向对象的软件开发人员所采用。设计模式是软件开发人员在软件开发过程中面临的一般问题的解决方案。这些解决方案是众多软件开发人员经过相当长的一段时间的试验和错误总结出来的。

设计模式是一套被反复使用的、多数人知晓的、经过分类编目的、代码设计经验的总结。使用设计模式是为了重用代码、让代码更容易被他人理解、保证代码可靠性。 毫无疑问，设计模式于己于他人于系统都是多赢的，设计模式使代码编制真正工程化，设计模式是软件工程的基石，如同大厦的一块块砖石一样。

### 设计模式原则

- S – Single Responsibility Principle 单一职责原则
  - 一个程序只做好一件事
  - 如果功能过于复杂就拆分开，每个部分保持独立
- O – OpenClosed Principle 开放/封闭原则
  - 对扩展开放，对修改封闭
  - 增加需求时，扩展新代码，而非修改已有代码
- L – Liskov Substitution Principle 里氏替换原则
  - 子类能覆盖父类
  - 父类能出现的地方子类就能出现
- I – Interface Segregation Principle 接口隔离原则
  - 保持接口的单一独立
  - 类似单一职责原则，这里更关注接口
- D – Dependency Inversion Principle 依赖倒转原则
  - 面向接口编程，依赖于抽象而不依赖于具体
  - 使用方只关注接口而不关注具体类的实现

##### SO体现较多，举个栗子：（比如Promise）

- 单一职责原则：每个then中的逻辑只做好一件事
- 开放封闭原则（对扩展开放，对修改封闭）：如果新增需求，扩展then

##### 再举个栗子：(此例来源-[守候-改善代码的各方面问题](https://juejin.im/post/6844903597092651015#comment))

```
//checkType('165226226326','mobile')
//result：false
let checkType=function(str, type) {
    switch (type) {
        case 'email':
            return /^[\w-]+(\.[\w-]+)*@[\w-]+(\.[\w-]+)+$/.test(str)
        case 'mobile':
            return /^1[3|4|5|7|8][0-9]{9}$/.test(str);
        case 'tel':
            return /^(0\d{2,3}-\d{7,8})(-\d{1,4})?$/.test(str);
        default:
            return true;
    }
}

复制代码
```

有以下两个问题：

- 如果想添加其他规则就得在函数里面增加 case 。添加一个规则就修改一次！这样违反了开放-封闭原则（对扩展开放，对修改关闭）。而且这样也会导致整个 API 变得臃肿，难维护。
- 比如A页面需要添加一个金额的校验，B页面需要一个日期的校验，但是金额的校验只在A页面需要，日期的校验只在B页面需要。如果一直添加 case 。就是导致A页面把只在B页面需要的校验规则也添加进去，造成不必要的开销。B页面也同理。

建议的方式是给这个 API 增加一个扩展的接口:

```
let checkType=(function(){
    let rules={
        email(str){
            return /^[\w-]+(\.[\w-]+)*@[\w-]+(\.[\w-]+)+$/.test(str);
        },
        mobile(str){
            return /^1[3|4|5|7|8][0-9]{9}$/.test(str);
        }
    };
    //暴露接口
    return {
        //校验
        check(str, type){
            return rules[type]?rules[type](str):false;
        },
        //添加规则
        addRule(type,fn){
            rules[type]=fn;
        }
    }
})();

//调用方式
//使用mobile校验规则
console.log(checkType.check('188170239','mobile'));
//添加金额校验规则
checkType.addRule('money',function (str) {
    return /^[0-9]+(.[0-9]{2})?$/.test(str)
});
//使用金额校验规则
console.log(checkType.check('18.36','money'));

复制代码
```

此例更详细内容请查看-> [守候i-重构-改善代码的各方面问题](https://juejin.im/post/6844903597092651015#comment)

## 设计模式分类（23种设计模式）

- 创建型
  - 单例模式
  - 原型模式
  - 工厂模式
  - 抽象工厂模式
  - 建造者模式
- 结构型
  - 适配器模式
  - 装饰器模式
  - 代理模式
  - 外观模式
  - 桥接模式
  - 组合模式
  - 享元模式
- 行为型
  - 观察者模式
  - 迭代器模式
  - 策略模式
  - 模板方法模式
  - 职责链模式
  - 命令模式
  - 备忘录模式
  - 状态模式
  - 访问者模式
  - 中介者模式
  - 解释器模式

### 工厂模式

工厂模式定义一个用于创建对象的接口，这个接口由子类决定实例化哪一个类。该模式使一个类的实例化延迟到了子类。而子类可以重写接口方法以便创建的时候指定自己的对象类型。

```
class Product {
    constructor(name) {
        this.name = name
    }
    init() {
        console.log('init')
    }
    fun() {
        console.log('fun')
    }
}

class Factory {
    create(name) {
        return new Product(name)
    }
}

// use
let factory = new Factory()
let p = factory.create('p1')
p.init()
p.fun()

复制代码
```

##### 适用场景

- 如果你不想让某个子系统与较大的那个对象之间形成强耦合，而是想运行时从许多子系统中进行挑选的话，那么工厂模式是一个理想的选择
- 将new操作简单封装，遇到new的时候就应该考虑是否用工厂模式；
- 需要依赖具体环境创建不同实例，这些实例都有相同的行为,这时候我们可以使用工厂模式，简化实现的过程，同时也可以减少每种对象所需的代码量，有利于消除对象间的耦合，提供更大的灵活性
- 

##### 优点

- 创建对象的过程可能很复杂，但我们只需要关心创建结果。
- 构造函数和创建者分离, 符合“开闭原则”
- 一个调用者想创建一个对象，只要知道其名称就可以了。
- 扩展性高，如果想增加一个产品，只要扩展一个工厂类就可以。

##### 缺点

- 添加新产品时，需要编写新的具体产品类,一定程度上增加了系统的复杂度
- 考虑到系统的可扩展性，需要引入抽象层，在客户端代码中均使用抽象层进行定义，增加了系统的抽象性和理解难度

##### 什么时候不用

当被应用到错误的问题类型上时,这一模式会给应用程序引入大量不必要的复杂性.除非为创建对象提供一个接口是我们编写的库或者框架的一个设计上目标,否则我会建议使用明确的构造器,以避免不必要的开销。

由于对象的创建过程被高效的抽象在一个接口后面的事实,这也会给依赖于这个过程可能会有多复杂的单元测试带来问题。

##### 例子

- 曾经我们熟悉的JQuery的$()就是一个工厂函数，它根据传入参数的不同创建元素或者去寻找上下文中的元素，创建成相应的jQuery对象

```
class jQuery {
    constructor(selector) {
        super(selector)
    }
    add() {
        
    }
  // 此处省略若干API
}

window.$ = function(selector) {
    return new jQuery(selector)
}


复制代码
```

- vue 的异步组件

在大型应用中，我们可能需要将应用分割成小一些的代码块，并且只在需要的时候才从服务器加载一个模块。为了简化，Vue 允许你以一个工厂函数的方式定义你的组件，这个工厂函数会异步解析你的组件定义。Vue 只有在这个组件需要被渲染的时候才会触发该工厂函数，且会把结果缓存起来供未来重渲染。例如：

```
Vue.component('async-example', function (resolve, reject) {
  setTimeout(function () {
    // 向 `resolve` 回调传递组件定义
    resolve({
      template: '<div>I am async!</div>'
    })
  }, 1000)
})

复制代码
```

------

### 单例模式

一个类只有一个实例，并提供一个访问它的全局访问点。

```
 class LoginForm {
    constructor() {
        this.state = 'hide'
    }
    show() {
        if (this.state === 'show') {
            alert('已经显示')
            return
        }
        this.state = 'show'
        console.log('登录框显示成功')
    }
    hide() {
        if (this.state === 'hide') {
            alert('已经隐藏')
            return
        }
        this.state = 'hide'
        console.log('登录框隐藏成功')
    }
 }
 LoginForm.getInstance = (function () {
     let instance
     return function () {
        if (!instance) {
            instance = new LoginForm()
        }
        return instance
     }
 })()

let obj1 = LoginForm.getInstance()
obj1.show()

let obj2 = LoginForm.getInstance()
obj2.hide()

console.log(obj1 === obj2)
复制代码
```

#### 优点

- 划分命名空间，减少全局变量
- 增强模块性，把自己的代码组织在一个全局变量名下，放在单一位置，便于维护
- 且只会实例化一次。简化了代码的调试和维护

#### 缺点

- 由于单例模式提供的是一种单点访问，所以它有可能导致模块间的强耦合 从而不利于单元测试。无法单独测试一个调用了来自单例的方法的类，而只能把它与那个单例作为一个单元一起测试。

#### 场景例子

- 定义命名空间和实现分支型方法
- 登录框
- vuex 和 redux中的store

------

### 适配器模式

将一个类的接口转化为另外一个接口，以满足用户需求，使类之间接口不兼容问题通过适配器得以解决。

```
class Plug {
  getName() {
    return 'iphone充电头';
  }
}

class Target {
  constructor() {
    this.plug = new Plug();
  }
  getName() {
    return this.plug.getName() + ' 适配器Type-c充电头';
  }
}

let target = new Target();
target.getName(); // iphone充电头 适配器转Type-c充电头
复制代码
```

#### 优点

- 可以让任何两个没有关联的类一起运行。
- 提高了类的复用。
- 适配对象，适配库，适配数据

#### 缺点

- 额外对象的创建，非直接调用，存在一定的开销（且不像代理模式在某些功能点上可实现性能优化)
- 如果没必要使用适配器模式的话，可以考虑重构，如果使用的话，尽量把文档完善

#### 场景

- 整合第三方SDK
- 封装旧接口

```
// 自己封装的ajax， 使用方式如下
ajax({
    url: '/getData',
    type: 'Post',
    dataType: 'json',
    data: {
        test: 111
    }
}).done(function() {})
// 因为历史原因，代码中全都是：
// $.ajax({....})

// 做一层适配器
var $ = {
    ajax: function (options) {
        return ajax(options)
    }
}
复制代码
```

- vue的computed

```
<template>
    <div id="example">
        <p>Original message: "{{ message }}"</p>  <!-- Hello -->
        <p>Computed reversed message: "{{ reversedMessage }}"</p>  <!-- olleH -->
    </div>
</template>
<script type='text/javascript'>
    export default {
        name: 'demo',
        data() {
            return {
                message: 'Hello'
            }
        },
        computed: {
            reversedMessage: function() {
                return this.message.split('').reverse().join('')
            }
        }
    }
</script>

复制代码
```

##### 原有data 中的数据不满足当前的要求，通过计算属性的规则来适配成我们需要的格式，对原有数据并没有改变，只改变了原有数据的表现形式

#### 不同点

适配器与代理模式相似

- 适配器模式： 提供一个不同的接口（如不同版本的插头）
- 代理模式： 提供一模一样的接口

------

### 装饰者模式

- 动态地给某个对象添加一些额外的职责，，是一种实现继承的替代方案
- 在不改变原对象的基础上，通过对其进行包装扩展，使原有对象可以满足用户的更复杂需求，而不会影响从这个类中派生的其他对象

```
class Cellphone {
    create() {
        console.log('生成一个手机')
    }
}
class Decorator {
    constructor(cellphone) {
        this.cellphone = cellphone
    }
    create() {
        this.cellphone.create()
        this.createShell(cellphone)
    }
    createShell() {
        console.log('生成手机壳')
    }
}
// 测试代码
let cellphone = new Cellphone()
cellphone.create()

console.log('------------')
let dec = new Decorator(cellphone)
dec.create()
复制代码
```

#### 场景例子

- 比如现在有4 种型号的自行车，我们为每种自行车都定义了一个单

独的类。现在要给每种自行车都装上前灯、尾 灯和铃铛这3 种配件。如果使用继承的方式来给 每种自行车创建子类，则需要 4×3 = 12 个子类。 但是如果把前灯、尾灯、铃铛这些对象动态组 合到自行车上面，则只需要额外增加3 个类

- ES7 Decorator [阮一峰](https://link.juejin.cn?target=http%3A%2F%2Fes6.ruanyifeng.com%2F%23docs%2Fdecorator)
- core-decorators

#### 优点

- 装饰类和被装饰类都只关心自身的核心业务，实现了解耦。
- 方便动态的扩展功能，且提供了比继承更多的灵活性。

#### 缺点

- 多层装饰比较复杂。
- 常常会引入许多小对象，看起来比较相似，实际功能大相径庭，从而使得我们的应用程序架构变得复杂起来

------

### 代理模式

是为一个对象提供一个代用品或占位符，以便控制对它的访问

> 假设当A 在心情好的时候收到花，小明表白成功的几率有

60%，而当A 在心情差的时候收到花，小明表白的成功率无限趋近于0。 小明跟A 刚刚认识两天，还无法辨别A 什么时候心情好。如果不合时宜地把花送给A，花 被直接扔掉的可能性很大，这束花可是小明吃了7 天泡面换来的。 但是A 的朋友B 却很了解A，所以小明只管把花交给B，B 会监听A 的心情变化，然后选 择A 心情好的时候把花转交给A，代码如下：

```
let Flower = function() {}
let xiaoming = {
  sendFlower: function(target) {
    let flower = new Flower()
    target.receiveFlower(flower)
  }
}
let B = {
  receiveFlower: function(flower) {
    A.listenGoodMood(function() {
      A.receiveFlower(flower)
    })
  }
}
let A = {
  receiveFlower: function(flower) {
    console.log('收到花'+ flower)
  },
  listenGoodMood: function(fn) {
    setTimeout(function() {
      fn()
    }, 1000)
  }
}
xiaoming.sendFlower(B)
复制代码
```

#### 场景

- HTML元 素事件代理

```
<ul id="ul">
  <li>1</li>
  <li>2</li>
  <li>3</li>
</ul>
<script>
  let ul = document.querySelector('#ul');
  ul.addEventListener('click', event => {
    console.log(event.target);
  });
</script>
复制代码
```

- ES6 的 proxy [阮一峰Proxy](https://link.juejin.cn?target=http%3A%2F%2Fes6.ruanyifeng.com%2F%23docs%2Fproxy)
- jQuery.proxy()方法

#### 优点

- 代理模式能将代理对象与被调用对象分离，降低了系统的耦合度。代理模式在客户端和目标对象之间起到一个中介作用，这样可以起到保护目标对象的作用
- 代理对象可以扩展目标对象的功能；通过修改代理对象就可以了，符合开闭原则；

#### 缺点

处理请求速度可能有差别，非直接访问存在开销

#### 不同点

装饰者模式实现上和代理模式类似

- 装饰者模式：  扩展功能，原有功能不变且可直接使用
- 代理模式： 显示原有功能，但是经过限制之后的

------

### 外观模式

为子系统的一组接口提供一个一致的界面，定义了一个高层接口，这个接口使子系统更加容易使用

1. 兼容浏览器事件绑定

```
let addMyEvent = function (el, ev, fn) {
    if (el.addEventListener) {
        el.addEventListener(ev, fn, false)
    } else if (el.attachEvent) {
        el.attachEvent('on' + ev, fn)
    } else {
        el['on' + ev] = fn
    }
}; 
复制代码
```

1. 封装接口

```
let myEvent = {
    // ...
    stop: e => {
        e.stopPropagation();
        e.preventDefault();
    }
};
复制代码
```

#### 场景

- 设计初期，应该要有意识地将不同的两个层分离，比如经典的三层结构，在数据访问层和业务逻辑层、业务逻辑层和表示层之间建立外观Facade
- 在开发阶段，子系统往往因为不断的重构演化而变得越来越复杂，增加外观Facade可以提供一个简单的接口，减少他们之间的依赖。
- 在维护一个遗留的大型系统时，可能这个系统已经很难维护了，这时候使用外观Facade也是非常合适的，为系系统开发一个外观Facade类，为设计粗糙和高度复杂的遗留代码提供比较清晰的接口，让新系统和Facade对象交互，Facade与遗留代码交互所有的复杂工作。

参考： 大话设计模式

#### 优点

- 减少系统相互依赖。
- 提高灵活性。
- 提高了安全性

#### 缺点

- 不符合开闭原则，如果要改东西很麻烦，继承重写都不合适。

------

### 观察者模式

定义了一种一对多的关系，让多个观察者对象同时监听某一个主题对象，这个主题对象的状态发生变化时就会通知所有的观察者对象，使它们能够自动更新自己，当一个对象的改变需要同时改变其它对象，并且它不知道具体有多少对象需要改变的时候，就应该考虑使用观察者模式。

- 发布 & 订阅
- 一对多

```
// 主题 保存状态，状态变化之后触发所有观察者对象
class Subject {
  constructor() {
    this.state = 0
    this.observers = []
  }
  getState() {
    return this.state
  }
  setState(state) {
    this.state = state
    this.notifyAllObservers()
  }
  notifyAllObservers() {
    this.observers.forEach(observer => {
      observer.update()
    })
  }
  attach(observer) {
    this.observers.push(observer)
  }
}

// 观察者
class Observer {
  constructor(name, subject) {
    this.name = name
    this.subject = subject
    this.subject.attach(this)
  }
  update() {
    console.log(`${this.name} update, state: ${this.subject.getState()}`)
  }
}

// 测试
let s = new Subject()
let o1 = new Observer('o1', s)
let o2 = new Observer('02', s)

s.setState(12)
复制代码
```

#### 场景

- DOM事件

```
document.body.addEventListener('click', function() {
    console.log('hello world!');
});
document.body.click()
复制代码
```

- vue 响应式

#### 优点

- 支持简单的广播通信，自动通知所有已经订阅过的对象
- 目标对象与观察者之间的抽象耦合关系能单独扩展以及重用
- 增加了灵活性
- 观察者模式所做的工作就是在解耦，让耦合的双方都依赖于抽象，而不是依赖于具体。从而使得各自的变化都不会影响到另一边的变化。

#### 缺点

过度使用会导致对象与对象之间的联系弱化，会导致程序难以跟踪维护和理解

------

### 状态模式

允许一个对象在其内部状态改变的时候改变它的行为，对象看起来似乎修改了它的类

```
// 状态 （弱光、强光、关灯）
class State {
    constructor(state) {
        this.state = state
    }
    handle(context) {
        console.log(`this is ${this.state} light`)
        context.setState(this)
    }
}
class Context {
    constructor() {
        this.state = null
    }
    getState() {
        return this.state
    }
    setState(state) {
        this.state = state
    }
}
// test 
let context = new Context()
let weak = new State('weak')
let strong = new State('strong')
let off = new State('off')

// 弱光
weak.handle(context)
console.log(context.getState())

// 强光
strong.handle(context)
console.log(context.getState())

// 关闭
off.handle(context)
console.log(context.getState())
复制代码
```

#### 场景

- 一个对象的行为取决于它的状态，并且它必须在运行时刻根据状态改变它的行为
- 一个操作中含有大量的分支语句，而且这些分支语句依赖于该对象的状态

#### 优点

- 定义了状态与行为之间的关系，封装在一个类里，更直观清晰，增改方便
- 状态与状态间，行为与行为间彼此独立互不干扰
- 用对象代替字符串来记录当前状态，使得状态的切换更加一目了然
- 

#### 缺点

- 会在系统中定义许多状态类
- 逻辑分散

------

### 迭代器模式

提供一种方法顺序一个聚合对象中各个元素，而又不暴露该对象的内部表示。

```
class Iterator {
    constructor(conatiner) {
        this.list = conatiner.list
        this.index = 0
    }
    next() {
        if (this.hasNext()) {
            return this.list[this.index++]
        }
        return null
    }
    hasNext() {
        if (this.index >= this.list.length) {
            return false
        }
        return true
    }
}

class Container {
    constructor(list) {
        this.list = list
    }
    getIterator() {
        return new Iterator(this)
    }
}

// 测试代码
let container = new Container([1, 2, 3, 4, 5])
let iterator = container.getIterator()
while(iterator.hasNext()) {
  console.log(iterator.next())
}
复制代码
```

#### 场景例子

- Array.prototype.forEach
- jQuery中的$.each()
- ES6 Iterator

#### 特点

- 访问一个聚合对象的内容而无需暴露它的内部表示。
- 为遍历不同的集合结构提供一个统一的接口，从而支持同样的算法在不同的集合结构上进行操作

#### 总结

对于集合内部结果常常变化各异，不想暴露其内部结构的话，但又想让客户代码透明的访问其中的元素，可以使用迭代器模式

------

### 桥接模式

桥接模式（Bridge）将抽象部分与它的实现部分分离，使它们都可以独立地变化。

```
class Color {
    constructor(name){
        this.name = name
    }
}
class Shape {
    constructor(name,color){
        this.name = name
        this.color = color 
    }
    draw(){
        console.log(`${this.color.name} ${this.name}`)
    }
}

//测试
let red = new Color('red')
let yellow = new Color('yellow')
let circle = new Shape('circle', red)
circle.draw()
let triangle = new Shape('triangle', yellow)
triangle.draw()

复制代码
```

#### 优点

- 有助于独立地管理各组成部分， 把抽象化与实现化解耦
- 提高可扩充性

#### 缺点

- 大量的类将导致开发成本的增加，同时在性能方面可能也会有所减少。

------

### 组合模式

- 将对象组合成树形结构，以表示“整体-部分”的层次结构。
- 通过对象的多态表现，使得用户对单个对象和组合对象的使用具有一致性。

```
class TrainOrder {
	create () {
		console.log('创建火车票订单')
	}
}
class HotelOrder {
	create () {
		console.log('创建酒店订单')
	}
}

class TotalOrder {
	constructor () {
		this.orderList = []
	}
	addOrder (order) {
		this.orderList.push(order)
		return this
	}
	create () {
		this.orderList.forEach(item => {
			item.create()
		})
		return this
	}
}
// 可以在购票网站买车票同时也订房间
let train = new TrainOrder()
let hotel = new HotelOrder()
let total = new TotalOrder()
total.addOrder(train).addOrder(hotel).create()
复制代码
```

#### 场景

- 表示对象-整体层次结构
- 希望用户忽略组合对象和单个对象的不同，用户将统一地使用组合结构中的所有对象（方法）

#### 缺点

如果通过组合模式创建了太多的对象，那么这些对象可能会让系统负担不起。

------

### 原型模式

原型模式（prototype）是指用原型实例指向创建对象的种类，并且通过拷贝这些原型创建新的对象。

```
class Person {
  constructor(name) {
    this.name = name
  }
  getName() {
    return this.name
  }
}
class Student extends Person {
  constructor(name) {
    super(name)
  }
  sayHello() {
    console.log(`Hello， My name is ${this.name}`)
  }
}

let student = new Student("xiaoming")
student.sayHello()
复制代码
```

原型模式，就是创建一个共享的原型，通过拷贝这个原型来创建新的类，用于创建重复的对象，带来性能上的提升。

------

### 策略模式

定义一系列的算法，把它们一个个封装起来，并且使它们可以互相替换

```
<html>
<head>
    <title>策略模式-校验表单</title>
    <meta content="text/html; charset=utf-8" http-equiv="Content-Type">
</head>
<body>
    <form id = "registerForm" method="post" action="http://xxxx.com/api/register">
        用户名：<input type="text" name="userName">
        密码：<input type="text" name="password">
        手机号码：<input type="text" name="phoneNumber">
        <button type="submit">提交</button>
    </form>
    <script type="text/javascript">
        // 策略对象
        const strategies = {
          isNoEmpty: function (value, errorMsg) {
            if (value === '') {
              return errorMsg;
            }
          },
          isNoSpace: function (value, errorMsg) {
            if (value.trim() === '') {
              return errorMsg;
            }
          },
          minLength: function (value, length, errorMsg) {
            if (value.trim().length < length) {
              return errorMsg;
            }
          },
          maxLength: function (value, length, errorMsg) {
            if (value.length > length) {
              return errorMsg;
            }
          },
          isMobile: function (value, errorMsg) {
            if (!/^(13[0-9]|14[5|7]|15[0|1|2|3|5|6|7|8|9]|17[7]|18[0|1|2|3|5|6|7|8|9])\d{8}$/.test(value)) {
              return errorMsg;
            }                
          }
        }
        
        // 验证类
        class Validator {
          constructor() {
            this.cache = []
          }
          add(dom, rules) {
            for(let i = 0, rule; rule = rules[i++];) {
              let strategyAry = rule.strategy.split(':')
              let errorMsg = rule.errorMsg
              this.cache.push(() => {
                let strategy = strategyAry.shift()
                strategyAry.unshift(dom.value)
                strategyAry.push(errorMsg)
                return strategies[strategy].apply(dom, strategyAry)
              })
            }
          }
          start() {
            for(let i = 0, validatorFunc; validatorFunc = this.cache[i++];) {
              let errorMsg = validatorFunc()
              if (errorMsg) {
                return errorMsg
              }
            }
          }
        }

        // 调用代码
        let registerForm = document.getElementById('registerForm')

        let validataFunc = function() {
          let validator = new Validator()
          validator.add(registerForm.userName, [{
            strategy: 'isNoEmpty',
            errorMsg: '用户名不可为空'
          }, {
            strategy: 'isNoSpace',
            errorMsg: '不允许以空白字符命名'
          }, {
            strategy: 'minLength:2',
            errorMsg: '用户名长度不能小于2位'
          }])
          validator.add(registerForm.password, [ {
            strategy: 'minLength:6',
            errorMsg: '密码长度不能小于6位'
          }])
          validator.add(registerForm.phoneNumber, [{
            strategy: 'isMobile',
            errorMsg: '请输入正确的手机号码格式'
          }])
          return validator.start()
        }

        registerForm.onsubmit = function() {
          let errorMsg = validataFunc()
          if (errorMsg) {
            alert(errorMsg)
            return false
          }
        }
    </script>
</body>
</html>
复制代码
```

#### 场景例子

- 如果在一个系统里面有许多类，它们之间的区别仅在于它们的'行为'，那么使用策略模式可以动态地让一个对象在许多行为中选择一种行为。
- 一个系统需要动态地在几种算法中选择一种。
- 表单验证

#### 优点

- 利用组合、委托、多态等技术和思想，可以有效的避免多重条件选择语句
- 提供了对开放-封闭原则的完美支持，将算法封装在独立的strategy中，使得它们易于切换，理解，易于扩展
- 利用组合和委托来让Context拥有执行算法的能力，这也是继承的一种更轻便的代替方案

#### 缺点

- 会在程序中增加许多策略类或者策略对象
- 要使用策略模式，必须了解所有的strategy，必须了解各个strategy之间的不同点，这样才能选择一个合适的strategy

------

### 享元模式

运用共享技术有效地支持大量细粒度对象的复用。系统只使用少量的对象，而这些对象都很相似，状态变化很小，可以实现对象的多次复用。由于享元模式要求能够共享的对象必须是细粒度对象，因此它又称为轻量级模式，它是一种对象结构型模式

```
let examCarNum = 0         // 驾考车总数
/* 驾考车对象 */
class ExamCar {
    constructor(carType) {
        examCarNum++
        this.carId = examCarNum
        this.carType = carType ? '手动档' : '自动档'
        this.usingState = false    // 是否正在使用
    }

    /* 在本车上考试 */
    examine(candidateId) {
        return new Promise((resolve => {
            this.usingState = true
            console.log(`考生- ${ candidateId } 开始在${ this.carType }驾考车- ${ this.carId } 上考试`)
            setTimeout(() => {
                this.usingState = false
                console.log(`%c考生- ${ candidateId } 在${ this.carType }驾考车- ${ this.carId } 上考试完毕`, 'color:#f40')
                resolve()                       // 0~2秒后考试完毕
            }, Math.random() * 2000)
        }))
    }
}

/* 手动档汽车对象池 */
ManualExamCarPool = {
    _pool: [],                  // 驾考车对象池
    _candidateQueue: [],        // 考生队列

    /* 注册考生 ID 列表 */
    registCandidates(candidateList) {
        candidateList.forEach(candidateId => this.registCandidate(candidateId))
    },

    /* 注册手动档考生 */
    registCandidate(candidateId) {
        const examCar = this.getManualExamCar()    // 找一个未被占用的手动档驾考车
        if (examCar) {
            examCar.examine(candidateId)           // 开始考试，考完了让队列中的下一个考生开始考试
              .then(() => {
                  const nextCandidateId = this._candidateQueue.length && this._candidateQueue.shift()
                  nextCandidateId && this.registCandidate(nextCandidateId)
              })
        } else this._candidateQueue.push(candidateId)
    },

    /* 注册手动档车 */
    initManualExamCar(manualExamCarNum) {
        for (let i = 1; i <= manualExamCarNum; i++) {
            this._pool.push(new ExamCar(true))
        }
    },

    /* 获取状态为未被占用的手动档车 */
    getManualExamCar() {
        return this._pool.find(car => !car.usingState)
    }
}

ManualExamCarPool.initManualExamCar(3)          // 一共有3个驾考车
ManualExamCarPool.registCandidates([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])  // 10个考生来考试
复制代码
```

#### 场景例子

- 文件上传需要创建多个文件实例的时候
- 如果一个应用程序使用了大量的对象，而这些大量的对象造成了很大的存储开销时就应该考虑使用享元模式

#### 优点

- 大大减少对象的创建，降低系统的内存，使效率提高。

#### 缺点

- 提高了系统的复杂度，需要分离出外部状态和内部状态，而且外部状态具有固有化的性质，

## 不应该随着内部状态的变化而变化，否则会造成系统的混乱

### 模板方法模式

模板方法模式由两部分结构组成，第一部分是抽象父类，第二部分是具体的实现子类。通常在抽象父类中封装了子类的算法框架，包括实现一些公共方法和封装子类中所有方法的执行顺序。子类通过继承这个抽象类，也继承了整个算法结构，并且可以选择重写父类的方法。

```
class Beverage {
    constructor({brewDrink, addCondiment}) {
        this.brewDrink = brewDrink
        this.addCondiment = addCondiment
    }
    /* 烧开水，共用方法 */
    boilWater() { console.log('水已经煮沸=== 共用') }
    /* 倒杯子里，共用方法 */
    pourCup() { console.log('倒进杯子里===共用') }
    /* 模板方法 */
    init() {
        this.boilWater()
        this.brewDrink()
        this.pourCup()
        this.addCondiment()
    }
}
/* 咖啡 */
const coffee = new Beverage({
     /* 冲泡咖啡，覆盖抽象方法 */
     brewDrink: function() { console.log('冲泡咖啡') },
     /* 加调味品，覆盖抽象方法 */
     addCondiment: function() { console.log('加点奶和糖') }
})
coffee.init() 
复制代码
```

#### 场景例子

- 一次性实现一个算法的不变的部分，并将可变的行为留给子类来实现
- 子类中公共的行为应被提取出来并集中到一个公共父类中的避免代码重复

#### 优点

- 提取了公共代码部分，易于维护

#### 缺点

- 增加了系统复杂度，主要是增加了的抽象类和类间联系

------

### 职责链模式

使多个对象都有机会处理请求，从而避免请求的发送者和接受者之间的耦合关系，将这些对象连成一条链，并沿着这条链传递该请求，直到有一个对象处理它为止

```
// 请假审批，需要组长审批、经理审批、总监审批
class Action {
    constructor(name) {
        this.name = name
        this.nextAction = null
    }
    setNextAction(action) {
        this.nextAction = action
    }
    handle() {
        console.log( `${this.name} 审批`)
        if (this.nextAction != null) {
            this.nextAction.handle()
        }
    }
}

let a1 = new Action("组长")
let a2 = new Action("经理")
let a3 = new Action("总监")
a1.setNextAction(a2)
a2.setNextAction(a3)
a1.handle()
复制代码
```

#### 场景例子

- JS 中的事件冒泡
- 作用域链
- 原型链

#### 优点

- 降低耦合度。它将请求的发送者和接收者解耦。
- 简化了对象。使得对象不需要知道链的结构
- 增强给对象指派职责的灵活性。通过改变链内的成员或者调动它们的次序，允许动态地新增或者删除责任
- 增加新的请求处理类很方便。

#### 缺点

- 不能保证某个请求一定会被链中的节点处理，这种情况可以在链尾增加一个保底的接受者节点来处理这种即将离开链尾的请求。
- 使程序中多了很多节点对象，可能再一次请求的过程中，大部分的节点并没有起到实质性的作用。他们的作用仅仅是让请求传递下去，从性能当面考虑，要避免过长的职责链到来的性能损耗。

------

### 命令模式

将一个请求封装成一个对象，从而让你使用不同的请求把客户端参数化，对请求排队或者记录请求日志，可以提供命令的撤销和恢复功能。

```
// 接收者类
class Receiver {
    execute() {
      console.log('接收者执行请求')
    }
  }
  
// 命令者
class Command {  
    constructor(receiver) {
        this.receiver = receiver
    }
    execute () {    
        console.log('命令');
        this.receiver.execute()
    }
}
// 触发者
class Invoker {   
    constructor(command) {
        this.command = command
    }
    invoke() {   
        console.log('开始')
        this.command.execute()
    }
}
  
// 仓库
const warehouse = new Receiver();   
// 订单    
const order = new Command(warehouse);  
// 客户
const client = new Invoker(order);      
client.invoke()
复制代码
```

#### 优点

- 对命令进行封装，使命令易于扩展和修改
- 命令发出者和接受者解耦，使发出者不需要知道命令的具体执行过程即可执行

#### 缺点

- 使用命令模式可能会导致某些系统有过多的具体命令类。

------

### 备忘录模式

在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态。这样以后就可将该对象恢复到保存的状态。

```
//备忘类
class Memento{
    constructor(content){
        this.content = content
    }
    getContent(){
        return this.content
    }
}
// 备忘列表
class CareTaker {
    constructor(){
        this.list = []
    }
    add(memento){
        this.list.push(memento)
    }
    get(index){
        return this.list[index]
    }
}
// 编辑器
class Editor {
    constructor(){
        this.content = null
    }
    setContent(content){
        this.content = content
    }
    getContent(){
     return this.content
    }
    saveContentToMemento(){
        return new Memento(this.content)
    }
    getContentFromMemento(memento){
        this.content = memento.getContent()
    }
}

//测试代码

let editor = new Editor()
let careTaker = new CareTaker()

editor.setContent('111')
editor.setContent('222')
careTaker.add(editor.saveContentToMemento())
editor.setContent('333')
careTaker.add(editor.saveContentToMemento())
editor.setContent('444')

console.log(editor.getContent()) //444
editor.getContentFromMemento(careTaker.get(1))
console.log(editor.getContent()) //333

editor.getContentFromMemento(careTaker.get(0))
console.log(editor.getContent()) //222
复制代码
```

#### 场景例子

- 分页控件
- 撤销组件

#### 优点

- 给用户提供了一种可以恢复状态的机制，可以使用户能够比较方便地回到某个历史的状态

#### 缺点

- 消耗资源。如果类的成员变量过多，势必会占用比较大的资源，而且每一次保存都会消耗一定的内存。

------

### 中介者模式

解除对象与对象之间的紧耦合关系。增加一个中介者对象后，所有的 相关对象都通过中介者对象来通信，而不是互相引用，所以当一个对象发生改变时，只需要通知 中介者对象即可。中介者使各对象之间耦合松散，而且可以独立地改变它们之间的交互。中介者 模式使网状的多对多关系变成了相对简单的一对多关系（类似于观察者模式，但是单向的，由中介者统一管理。）

```
class A {
    constructor() {
        this.number = 0
    }
    setNumber(num, m) {
        this.number = num
        if (m) {
            m.setB()
        }
    }
}
class B {
    constructor() {
        this.number = 0
    }
    setNumber(num, m) {
        this.number = num
        if (m) {
            m.setA()
        }
    }
}
class Mediator {
    constructor(a, b) {
        this.a = a
        this.b = b
    }
    setA() {
        let number = this.b.number
        this.a.setNumber(number * 10)
    }
    setB() {
        let number = this.a.number
        this.b.setNumber(number / 10)
    }
}

let a = new A()
let b = new B()
let m = new Mediator(a, b)
a.setNumber(10, m)
console.log(a.number, b.number)
b.setNumber(10, m)
console.log(a.number, b.number)
复制代码
```

#### 场景例子

- 系统中对象之间存在比较复杂的引用关系，导致它们之间的依赖关系结构混乱而且难以复用该对象
- 想通过一个中间类来封装多个类中的行为，而又不想生成太多的子类。

#### 优点

- 使各对象之间耦合松散，而且可以独立地改变它们之间的交互
- 中介者和对象一对多的关系取代了对象之间的网状多对多的关系
- 如果对象之间的复杂耦合度导致维护很困难，而且耦合度随项目变化增速很快，就需要中介者重构代码

#### 缺点

- 系统中会新增一个中介者对象，因 为对象之间交互的复杂性，转移成了中介者对象的复杂性，使得中介者对象经常是巨大的。中介 者对象自身往往就是一个难以维护的对象。

------

### 解释器模式

给定一个语言, 定义它的文法的一种表示，并定义一个解释器, 该解释器使用该表示来解释语言中的句子。

此例来自[心谭博客](https://link.juejin.cn?target=https%3A%2F%2Fxin-tan.com%2Fpassages%2F2019-01-25-interpreter-pattern%2F%23_3-%E5%A4%9A%E8%AF%AD%E8%A8%80%E5%AE%9E%E7%8E%B0)

```
class Context {
    constructor() {
      this._list = []; // 存放 终结符表达式
      this._sum = 0; // 存放 非终结符表达式(运算结果)
    }
  
    get sum() {
      return this._sum;
    }
    set sum(newValue) {
      this._sum = newValue;
    }
    add(expression) {
      this._list.push(expression);
    }
    get list() {
      return [...this._list];
    }
  }
  
  class PlusExpression {
    interpret(context) {
      if (!(context instanceof Context)) {
        throw new Error("TypeError");
      }
      context.sum = ++context.sum;
    }
  }
  class MinusExpression {
    interpret(context) {
      if (!(context instanceof Context)) {
        throw new Error("TypeError");
      }
      context.sum = --context.sum;
    }
  }
  
  /** 以下是测试代码 **/
  const context = new Context();
  
  // 依次添加: 加法 | 加法 | 减法 表达式
  context.add(new PlusExpression());
  context.add(new PlusExpression());
  context.add(new MinusExpression());
  
  // 依次执行: 加法 | 加法 | 减法 表达式
  context.list.forEach(expression => expression.interpret(context));
  console.log(context.sum);
复制代码
```

#### 优点

- 易于改变和扩展文法。
- 由于在解释器模式中使用类来表示语言的文法规则，因此可以通过继承等机制来改变或扩展文法

#### 缺点

- 执行效率较低，在解释器模式中使用了大量的循环和递归调用，因此在解释较为复杂的句子时其速度慢
- 对于复杂的文法比较难维护

------

### 访问者模式

表示一个作用于某对象结构中的各元素的操作。它使你可以在不改变各元素的类的前提下定义作用于这些元素的新操作。

```
// 访问者  
class Visitor {
    constructor() {}
    visitConcreteElement(ConcreteElement) {
        ConcreteElement.operation()
    }
}
// 元素类  
class ConcreteElement{
    constructor() {
    }
    operation() {
       console.log("ConcreteElement.operation invoked");  
    }
    accept(visitor) {
        visitor.visitConcreteElement(this)
    }
}
// client
let visitor = new Visitor()
let element = new ConcreteElement()
element.accept(visitor)
复制代码
```

#### 场景例子

- 对象结构中对象对应的类很少改变，但经常需要在此对象结构上定义新的操作
- 需要对一个对象结构中的对象进行很多不同的并且不相关的操作，而需要避免让这些操作"污染"这些对象的类，也不希望在增加新操作时修改这些类。

#### 优点

- 符合单一职责原则
- 优秀的扩展性
- 灵活性

#### 缺点

- 具体元素对访问者公布细节，违反了迪米特原则
- 违反了依赖倒置原则，依赖了具体类，没有依赖抽象。
- 具体元素变更比较困难


作者：九思
链接：https://juejin.cn/post/6844904032826294286
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。