# 从0实现一个single-spa的前端微服务（下）

## 前言

上一篇文章：[从0实现一个single-spa的前端微服务（中）](./从0实现一个single-spa的前端微服务（中）.md)中我们已经实现了`single-spa` + `systemJS`的前端微服务以及完善的开发和打包配置，今天主要讲一下这个方案存在的细节问题，以及`qiankun`框架的一些研究对比。

## single-spa + systemJs 方案存在的问题及解决办法

`single-spa`的三个生命周期函数`bootstrap`、 `mount`、 `unmount`分别表示初始化、加载时、卸载时。

1. 子系统导出`bootstrap`、`mount`和`unmount`函数是必需的，但是`unload`是可选的。
2. 每个生命周期函数必须返回`Promise`。
3. 如果导出一个函数数组（而不只是一个函数），这些函数将一个接一个地调用，等待一个函数的`promise`解析后再调用下一个。

### css污染问题的解决

我们知道，子系统卸载之后，其引入的`css`并不会被删掉，所以在子系统卸载时删掉这些`css`，是一种解决`css`污染的办法，但是不太好记录子系统引入了哪些`css`。

我们可以借助换肤的思路来解决`css`污染，首先`css-scoped`解决95%的样式污染，然后就是全局样式可能会造成污染，我们只需要将全局样式用一个`id/class`包裹着就可以了，这样这些全局样式仅在这个`id/class`范围内生效。

具体做法就是：在子系统加载时(`mount`)给`<body>`加一个特殊的`id/class`，然后在子系统卸载时(`unmount`)删掉这个`id/class`。而子系统的全局样式都仅在这个`id/class`范围内生效，如果子系统独立运行，只需要在子系统的入口文件`index.html`里面给`<body>`手动加上这个`id/class`即可。

代码如下：

```javascript
async function mount(props){
  //给body加class,以解决全局样式污染
  document.body.classList.add('app-vue-history')
}
async function unmount(props){
  //去掉body的class
  document.body.classList.remove('app-vue-history')
}

```

当然了，你写的全局样式也在这个`class`下面：

```less
.app-vue-history{
    h1{
        color: red
    }
}

```

### js污染问题的解决

暂时没有很好的办法解决，但是可以靠编码规范来约束：页面销毁之前清除自己页面上的定时器/全局事件，必要的时候，全局变量也应该销毁。

### 如何实现切换系统更换favicon.ico图标

这是一个比较常见的需求，类似还有某个系统需要插入一段特殊的`js/css`，而其他系统不需要，解决办法任然是在子系统加载时(`mount`)插入需要的`js/css`，在子系统卸载时(`unmount`)删掉。

```javascript
const headEle = document.querySelector('head');
let linkEle = null ;
// 因为新插入的icon会覆盖旧的，所以旧的不用删除，如果需要删除，可以在unmount时再插入进来
async function mount(props){
  linkEle = document.createElement("link");
  linkEle.setAttribute('rel','icon');
  linkEle.setAttribute('href','https://gold-cdn.xitu.io/favicons/favicon.ico');
  headEle.appendChild(linkEle);
}
async function unmount(props){
  headEle.removeChild(linkEle);
  linkEle = null;
}

```

> 注意：上面例子中是修改icon标签，不影响页面的加载。如果某个子系统需要在页面加载之前加载某个js(例如配置文件)，需要将加载 js 的函数写成 promise，并且将这个周期函数放到 single-spa-vue 返回的周期前面。

### 系统之间如何通信

系统之间通信一般有两种方式：自定义事件和本地存储。如果是两个系统相互跳转，可以用`URL`传数据。

一般来说，不会同时存在A、B两个子系统，常见的数据共享就是登陆信息，登陆信息一般使用本地存储记录。另外一个常见的场景就是子系统修改了用户信息，主系统需要重新请求用户信息，这个时候一般用自定义事件通信，自定义事件具体如何操作，可以看上一篇文章的例子。

另外，`single-spa`的注册函数`registerApplication`，第四个参数可以传递数据给子系统，但传递的数据必须是一个**对象**。

注册子系统的时候：

```javascript
singleSpa.registerApplication(
    'appVueHistory',
    () => System.import('appVueHistory'),
    location => location.pathname.startsWith('/app-vue-history/'),
    { authToken: "d83jD63UdZ6RS6f70D0" }
)

```

子系统（`appVueHistory`）接收数据：

```javascript
export function mount(props) {
  //官方文档写的是props.customProps.authToken，实际上发现是props.authToken
  console.log(props.authToken); 
  return vueLifecycles.mount(props);
}

```

关于子系统的生命周期函数：

1. 生命周期函数`bootstrap`,`mount`，`unmount`均包含参数`props`
2. 参数`props`是一个对象，包含`name`，`singleSpa`，`mountParcel`，`customProps` 。不同的版本可能略有差异
3. 参数对象中 `customProps` 就是注册的时候传递过来的参数

### 子系统如何实现keep-alive

查看`single-spa-vue`源码可以发现，在`unmount`生命周期，它将`vue`实例`destroy`（销毁了）并且清空了`DOM`。所以实现`keep-alive`的关键在于子系统的`unmount`周期中不销毁`vue`实例并且不清空`DOM`，采用`display:none`来隐藏子系统。而在`mount`周期，先判断子系统是否存在，如果存在，则去掉其`display:none`即可。

我们需要修改`single-spa-vue`的部分源代码：

```javascript
function mount(opts, mountedInstances, props) {
  let instance = mountedInstances[props.name];
  return Promise.resolve().then(() => {
    //先判断是否已加载，如果是，则直接将其显示出来
    if(!instance){
      //这里面都是其源码，生成DOM并实例化vue的部分
      instance = {};
      const appOptions = { ...opts.appOptions };
      if (props.domElement && !appOptions.el) {
        appOptions.el = props.domElement;
      }
      let domEl;
      if (appOptions.el) {
        if (typeof appOptions.el === "string") {
          domEl = document.querySelector(appOptions.el);
          if (!domEl) {
            throw Error(
              `If appOptions.el is provided to single-spa-vue, the dom element must 
                  exist in the dom. Was provided as ${appOptions.el}`
            );
          }
        } else {
          domEl = appOptions.el;
        }
      } else {
        const htmlId = `single-spa-application:${props.name}`;
        // CSS.escape 的文档（需考虑兼容性）
        // https://developer.mozilla.org/zh-CN/docs/Web/API/CSS/escape
        appOptions.el = `#${CSS.escape(htmlId)}`;
        domEl = document.getElementById(htmlId);
        if (!domEl) {
          domEl = document.createElement("div");
          domEl.id = htmlId;
          document.body.appendChild(domEl);
        }
      }
      appOptions.el = appOptions.el + " .single-spa-container";
      // single-spa-vue@>=2 always REPLACES the `el` instead of appending to it.
      // We want domEl to stick around and not be replaced. So we tell Vue to mount
      // into a container div inside of the main domEl
      if (!domEl.querySelector(".single-spa-container")) {
        const singleSpaContainer = document.createElement("div");
        singleSpaContainer.className = "single-spa-container";
        domEl.appendChild(singleSpaContainer);
      }
      instance.domEl = domEl;
      if (!appOptions.render && !appOptions.template && opts.rootComponent) {
        appOptions.render = h => h(opts.rootComponent);
      }
      if (!appOptions.data) {
        appOptions.data = {};
      }
      appOptions.data = { ...appOptions.data, ...props };
      instance.vueInstance = new opts.Vue(appOptions);
      if (instance.vueInstance.bind) {
        instance.vueInstance = instance.vueInstance.bind(instance.vueInstance);
      }
      mountedInstances[props.name] = instance;
    }else{
      instance.vueInstance.$el.style.display = "block";
    }
    return instance.vueInstance;
  });
}
function unmount(opts, mountedInstances, props) {
  return Promise.resolve().then(() => {
    const instance = mountedInstances[props.name];
    instance.vueInstance.$el.style.display = "none";
  });
}

```

而子系统内部页面则和正常`vue`系统一样使用`<keep-alive>`标签来实现缓存。

### 如何实现子系统的预请求(预加载)

`vue-router`路由配置的时候可以使用按需加载（代码如下），按需加载之后路由文件就会单独打包成一个`js`和`css`。

```javascript
path: "/about",
name: "about",
component: () => import( "../views/About.vue")

```

而 `vue-cli3`生成的模板打包后的`index.html`中是有使用`prefetch`和`preload`来实现路由文件的预请求的：

```html
<link href=/js/about.js rel=prefetch>
<link href=/js/app.js rel=preload as=script>

```

> `prefetch`预请求就是：浏览器网络空闲的时候请求并缓存文件

`systemJs`只能拿到入口文件，其他的路由文件是按需加载的，无法实现预请求。但是如果你没有使用路由的按需加载，则所有路由文件都打包到一个文件(`app.js`)，则可以实现预请求。

上述完整`demo`文件地址：[github.com/gongshun/si…](https://github.com/gongshun/single-spa-vue-demo)

## qiankun框架

`qiankun`是蚂蚁金服开源的基于`single-spa`的一个前端微服务框架。

### js沙箱（sandbox）是如何实现的

我们知道所有的全局的方法（`alert`，`setTimeout`，`isNaN`等）、全局的变/常量（`NaN`，`Infinity`，`var`声明的全局变量等）和全局对象（`Array`，`String`，`Date`等）都属于`window`对象，而能导致`js`污染的也就是这些全局的方法和对象。

所以`qiankun`解决`js`污染的办法是：在子系统加载之前对`window`对象做一个快照（拷贝），然后在子系统卸载的时候恢复这个快照，即可以保证每次子系统运行的时候都是一个全新的`window`对象环境。

那么如何监测`window`对象的变化呢，直接将`window`对象进行一下深拷贝，然后深度对比各个属性显然可行性不高，`qiankun`框架采用的是`ES6`新特性，`proxy`代理方法。

具体代码如下(源代码是`ts`版的，我简化修改了一些)：

```javascript
// 沙箱期间新增的全局变量
const addedPropsMapInSandbox = new Map();
// 沙箱期间更新的全局变量
const modifiedPropsOriginalValueMapInSandbox = new Map();
// 持续记录更新的(新增和修改的)全局变量的 map，用于在任意时刻做 snapshot
const currentUpdatedPropsValueMap = new Map();
const boundValueSymbol = Symbol('bound value');
const rawWindow = window;
const fakeWindow = Object.create(null);
const sandbox = new Proxy(fakeWindow, {
    set(target, propKey, value) {
      if (!rawWindow.hasOwnProperty(propKey)) {
        addedPropsMapInSandbox.set(propKey, value);
      } else if (!modifiedPropsOriginalValueMapInSandbox.has(propKey)) {
        // 如果当前 window 对象存在该属性，且 record map 中未记录过，则记录该属性初始值
        const originalValue = rawWindow[propKey];
        modifiedPropsOriginalValueMapInSandbox.set(propKey, originalValue);
      }
      currentUpdatedPropsValueMap.set(propKey, value);
      // 必须重新设置 window 对象保证下次 get 时能拿到已更新的数据
      rawWindow[propKey] = value;
      // 在 strict-mode 下，Proxy 的 handler.set 返回 false 会抛出 TypeError，
      // 在沙箱卸载的情况下应该忽略错误
      return true;
    },
    get(target, propKey) {
      if (propKey === 'top' || propKey === 'window' || propKey === 'self') {
        return sandbox;
      }
      const value = rawWindow[propKey];
      // isConstructablev :监测函数是否是构造函数
      if (typeof value === 'function' && !isConstructable(value)) {
        if (value[boundValueSymbol]) {
          return value[boundValueSymbol];
        }
        const boundValue = value.bind(rawWindow);
        Object.keys(value).forEach(key => (boundValue[key] = value[key]));
        Object.defineProperty(value, boundValueSymbol, 
            { enumerable: false, value: boundValue }
        )
        return boundValue;
      }
      return value;
    },
    has(target, propKey) {
      return propKey in rawWindow;
    },
});

```

大致原理就是记录`window`对象在子系统运行期间新增、修改和删除的属性和方法，然后会在子系统卸载的时候复原这些操作。

这样处理之后，全局变量可以直接复原，但是事件监听和定时器需要特殊处理：用`addEventListener`添加的事件，需要用`removeEventListener`方法来移除，定时器也需要特殊函数才能清除。所以它重写了事件绑定/解绑和定时器相关函数。

重写定时器(`setInterval`)部分代码如下：

```javascript
const rawWindowInterval = window.setInterval;
const hijack = function () {
  const timerIds = [];
  window.setInterval = (...args) => {
    const intervalId = rawWindowInterval(...args);
    intervalIds.push(intervalId);
    return intervalId;
  };
  return function free() {
    window.setInterval = rawWindowInterval;
    intervalIds.forEach(id => {
      window.clearInterval(id);
    });
  };
}


```

小细节：切换子系统不能立马清除子系统的延时定时器，比如说子系统有一个`message`提示，3秒钟后自动关闭，如果你立马清除掉了，就会一直存在了。那么延迟多久再清除子系统的定时器合适呢？5s？7s？10s？似乎都不太理想，作者最终决定不清除`setTimeout`，毕竟使用了一次之后就没用了，影响不大。

> 由于qiankun在js沙箱功能中使用了proxy新特性，所以它的兼容性和vue3一样，不支持IE11及以下版本的IE。不过作者说可以尝试禁用沙箱功能来提高兼容性，但是不保证都能运行。去掉了js沙箱功能，就变得索然无味了。

补充：qiankun 2.x 更新，对于不支持 proxy 的浏览器，支持 diff 方法来实现沙箱，就是子项目加载前浅拷贝一下 window，子项目卸载后 for 循环之前浅拷贝的 window，恢复之前的状态，但是多个子项目同时运行，他们共用一个沙箱，这等于没有沙箱。

#### 补充： 全局函数的影响如何消除

`function`关键字直接声明一个全局函数，这个函数属于`window`对象，但是无法被`delete`:

```javascript
function a(){}
Object.getOwnPropertyDescriptor(window, "a")
//控制台打印如下信息
/*{
    value: ƒ a(),
    writable: true,
    enumerable: true,
    configurable: false
}*/
delete window.a // 返回false，表示删除失败

```

> configurable：当且仅当指定对象的属性描述可以被改变或者属性可被删除时，为true

既然无法被`delete`，那么`qiankun`的`js`沙箱是如何做的呢，它是怎样消除子系统的全局函数的影响的呢?

声明全局函数有两种办法，一种是`function`关键字在全局环境下声明，另一种是以变量的形式添加：`window.a = () => {}`。我们知道`function`声明的全局函数是无法删除的，而变量的形式是可以删除的，`qiankun`直接避免了`function`关键字声明的全局函数。

首先，我们编写在`.vue`文件或者`main.js`文件中`function`声明的函数都不是全局函数，它只属于当前模块的。只有`index.html`中直接写的全局函数，或者不被打包文件里面的函数是全局的。

在`index.html`中编写的全局函数，会被处理成局部函数。 源代码：

```html
<script>
    function b(){}
    //测试全局变量污染
    console.log('window.b',window.b)
</script>

```

qiankun处理后：

```javascript
(function(window){;
    function b(){}
    //测试全局变量污染
    console.log('window.b',window.b)
}).bind(window.proxy)(window.proxy);

```

那他是如何实现的呢？首先用正则匹配到`index.html`里面的外链`js`和内联`js`，然后外链`js`请求到内容字符串后存储到一个对象中，内联`js`直接用正则匹配到内容也记录到这个对象中：

```javascript
const fetchScript = scriptUrl => scriptCache[scriptUrl] ||
		(scriptCache[scriptUrl] = fetch(scriptUrl).then(response => response.text()));

```

然后运行的时候，采用`eval`函数：

```javascript
//内联js
eval(`;(function(window){;${inlineScript}\n}).bind(window.proxy)(window.proxy);`)
//外链js
eval(`;(function(window){;${downloadedScriptText}\n}).bind(window.proxy)(window.proxy);`))

```

同时，他还会考虑到外链`js`的`async`属性，即考虑到`js`文件的先后执行顺序，不得不说，这个作者真的是细节满满。

### css污染他是如何解决的

它解决`css`污染的办法是：在子系统卸载的时候，将子系统引入`css`使用的`<link>`、`<style>`标签移除掉。移除的办法是重写`<head>`标签的`appendChild`方法，办法类似定时器的重写。

子系统加载时，会将所需要的`js/css`文件插入到`<head>`标签，而重写的`appendChild`方法会记录所插入的标签，然后子系统卸载的时候，会移除这些标签。

### 预请求是如何实现的

解决子系统预请求的的根本在于，我们需要知道子系统有哪些`js/css`需要加载，而借助`systemJs`加载子系统，只知道子系统的入口文件(`app.js`)。`qiankun`不仅支持`app.js`作为入口文件，还支持`index.html`作为入口文件，它会用正则匹配出`index.html`里面的`js/css`标签，然后实现预请求。

网络不好和移动端访问的时候，`qiankun`不会进行预请求，移动端大多是使用数据流量，预请求则会浪费用户流量，判断代码如下：

```javascript
const isMobile = 
   /Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent);
const isSlowNetwork = navigator.connection
  ? navigator.connection.saveData || /(2|3)g/.test(navigator.connection.effectiveType)
  : false;

```

请求`js/css`文件它采用的是`fetch`请求，如果浏览器不支持，还需要`polyfill`。

以下代码就是它请求`js`并进行缓存：

```javascript
const defaultFetch = window.fetch.bind(window);
//scripts是用正则匹配到的script标签
function getExternalScripts(scripts, fetch = defaultFetch) {
    return Promise.all(scripts.map(script => {
	if (script.startsWith('<')) {
	    // 内联js代码块
	    return getInlineCode(script);
	} else {
	    // 外链js
	    return scriptCache[script] ||
	           (scriptCache[script] = fetch(script).then(response => response.text()));
	}
    }));
}

```

## 用qiankun框架实现微前端

`qiankun`的[源码](https://github.com/umijs/qiankun)中已经给出了使用示例，使用起来也非常简单好用。接下来我演示下如何从0开始用`qianklun`框架实现微前端，内容改编自官方使用示例。PS：以下内容基于`qiankun1`版本，2.x版本请看：[改造已有的项目为 qiankun 子项目](https://juejin.im/post/6844904185910018062#heading-10) 和 [github 的 demo](https://github.com/gongshun/qiankun-vue-demo)

### 主项目main

1. `vue-cli3`生成一个全新的`vue`项目，注意路由使用`history`模式。
2. 安装`qiankun`框架：`npm i qiankun -S`
3. 修改`app.vue`,使其成为菜单和子项目的容器。其中两个数据，`loading`就是加载的状态，而`content`则是子系统生成的`HTML`片段（子系统独立运行时，这个`HTML`片段会被插入到`#app`里面的）

```html
<template>
  <div id="app">
    <header>
      <router-link to="/app-vue-hash/">app-vue-hash</router-link>
      <router-link to="/app-vue-history/">app-vue-history</router-link>
    </header>
    <div v-if="loading" class="loading">loading</div>
    <div class="appContainer" v-html="content">content</div>
  </div>
</template>

<script>
export default {
  props: {
    loading: {
      type: Boolean,
      default: false
    },
    content: {
      type: String,
      default: ''
    },
  },
}
</script>

```

1. 修改`main.js`，注册子项目，子项目入口文件采用`index.html`

```javascript
import Vue from 'vue'
import App from './App.vue'
import router from './router'
import { registerMicroApps, start } from 'qiankun';
Vue.config.productionTip = false
let app = null;
function render({ appContent, loading }) {
  if (!app) {
    app = new Vue({
      el: '#container',
      router,
      data() {
        return {
          content: appContent,
          loading,
        };
      },
      render(h){
        return h(App, {
          props: {
            content: this.content,
            loading: this.loading,
          },
        })
      } 
    });
  } else {
    app.content = appContent;
    app.loading = loading;
  }
}
function initApp() {
  render({ appContent: '', loading: false });
}
initApp();
function genActiveRule(routerPrefix) {
  return location => location.pathname.startsWith(routerPrefix);
}
registerMicroApps([
  { 
    name: 'app-vue-hash',
    entry: 'http://localhost:80', 
    render, 
    activeRule: genActiveRule('/app-vue-hash')
  },
  { 
    name: 'app-vue-history', 
    entry: 'http://localhost:1314',
    render, 
    activeRule: genActiveRule('/app-vue-history') 
  },
]);
start();

```

注意：主项目中的`index.html`模板里面的`<div id="app"></div>`需要改为`<div id="container"></div>`

### 子项目app-vue-hash

1. `vue-cli3`生成一个全新的`vue`项目，注意路由使用`hash`模式。
2. 在`src`目录新增文件`public-path.js`，主要用于修改子项目的`publicPath`。

```javascript
if (window.__POWERED_BY_QIANKUN__) {
  __webpack_public_path__ = window.__INJECTED_PUBLIC_PATH_BY_QIANKUN__;
}

```

1. 修改`main.js`，配合主项目导出`single-spa`需要的三个生命周期。注意：路由实例化需要在main.js里面完成，以便于路由的销毁，所以路由文件只需要导出路由配置即可（原模板导出的是路由实例）

```javascript
import './public-path';
import Vue from 'vue';
import VueRouter from 'vue-router';
import App from './App.vue';
import routes from './router';
import store from './store';

Vue.config.productionTip = false;
let router = null;
let instance = null;
function render() {
  router = new VueRouter({
    routes,
  });
  instance = new Vue({
    router,
    store,
    render: h => h(App),
  }).$mount('#appVueHash');// index.html 里面的 id 需要改成 appVueHash，否则子项目无法独立运行
}
if (!window.__POWERED_BY_QIANKUN__) {//全局变量来判断环境
  render();
}
export async function bootstrap() {
  console.log('vue app bootstraped');
}
export async function mount(props) {
  console.log('props from main framework', props);
  render();
}
export async function unmount() {
  instance.$destroy();
  instance = null;
  router = null;
}

```

1. 修改打包配置文件`vue.config.js` ，主要是允许跨域、以及打包成`umd`格式

```javascript
const { name } = require('./package');

const port = 7101; // dev port
module.exports = {
  devServer: {
    disableHostCheck: true,
    port,
    overlay: {
      warnings: false,
      errors: true,
    },
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  },
  // 自定义webpack配置
  configureWebpack: {
    output: {
      // 把子应用打包成 umd 库格式
      library: `${name}-[name]`,
      libraryTarget: 'umd',
      jsonpFunction: `webpackJsonp_${name}`,
    },
  },
};

```

### 子项目app-vue-history

`history`模式的`vue`项目与`hash`模式只有一个地方不同，其他的一模一样。

即`main.js`里面路由实例化的时候需要加入条件判断，注入路由前缀

```javascript
function render() {
  router = new VueRouter({
    base: window.__POWERED_BY_QIANKUN__ ? '/app-vue-history' : '/',
    mode: 'history',
    routes,
  });

  instance = new Vue({
    router,
    store,
    render: h => h(App),
  }).$mount('#appVueHistory');
}

```

### 项目之间的通信

自定义事件可以传递数据，但是似乎不太完美，数据不具备“双向传递性”。如果想在父子项目都能修改这个数据，并且都能响应，我们需要实现一个顶级`vuex`。

具体思路：

1. 在主项目实例化一个`Vuex`，然后在子项目注册时候传递给子项目
2. 子项目在`mounted`生命周期拿到主项目的`Vuex`，然后注册到全局去：`new Vue`的时候，在`data`中声明，这样子项目的任何一个组件都可以通过`this.$root`访问到这个`Vuex`。

大致代码如下，

主项目`main.js`:

```javascript
import store from './store';

registerMicroApps([
  { 
    name: 'app-vue-hash', 
    entry: 'http://localhost:7101', 
    render, 
    activeRule: genActiveRule('/app-vue-hash'), 
    props: { data : store } 
  },
  { 
    name: 'app-vue-history', 
    entry: 'http://localhost:1314', 
    render, 
    activeRule: genActiveRule('/app-vue-history'), 
    props: { data : store } 
  },
]);

```

子项目的`main.js`:

```javascript
function render(parentStore) {
  router = new VueRouter({
    routes,
  });
  instance = new Vue({
    router,
    store,
    data(){
      return {
        store: parentStore,
      }
    },
    render: h => h(App),
  }).$mount('#appVueHash');
}
export async function mount(props) {
  render(props.data);
}

```

子项目的`Home.vue`中使用：

```html
<template>
  <div class="home">
    <span @click="changeParentState">主项目的数据：{{ commonData.parent }},点击变为2</span>
  </div>
</template>
<script>
export default {
  computed: {
    commonData(){
      return this.$root.store.state.commonData;
    }
  },
  methods: {
    changeParentState(){
      this.$root.store.commit('setCommonData', { parent: 2 });
    }
  },
}
</script>

```

### 其他

1. 如果想关闭`js`沙箱和预请求，在`start`函数中配置即可

```javascript
start({
    prefetch: false, //默认是true，可选'all'
    jsSandbox: false, //默认是true
})

```

1. 子项目注册函数`registerMicroApps`也可以传递数据给子项目，并且可以设置全局的生命周期函数

```javascript
// 其中app对象的props属性就是传递给子项目的数据，默认是空对象
registerMicroApps(
  [
    { 
        name: 'app-vue-hash', 
        entry: 'http://localhost:80', 
        render, activeRule: 
        genActiveRule('/app-vue-hash') , 
        props: { data : 'message' } 
    },
    { 
        name: 'app-vue-history', 
        entry: 'http://localhost:1314', 
        render, 
        activeRule: genActiveRule('/app-vue-history') 
    },
  ],
  {
    beforeLoad: [
      app => { console.log('before load', app); },
    ],
    beforeMount: [
      app => { console.log('before mount', app); },
    ],
    afterUnmount: [
      app => { console.log('after unload', app); },
    ],
  },
);

```

1. `qiankun`的官方文档:[qiankun.umijs.org/zh/api/#reg…](https://qiankun.umijs.org/zh/api/#registermicroapps-apps-lifecycles-opts)
2. 上述`demo`的完整代码[github.com/gongshun/qi…](https://github.com/gongshun/qiankun-vue-demo)

## 总结

1. `js`沙箱并不能解决所有的`js`污染，例如我给`<body>`添加了一个点击事件，`js`沙箱并不能消除它的影响，所以说，还得靠代码规范和自己自觉。
2. 抛开兼容性，我觉得`qiankun`真的太好用了，无需对子项目做过多的修改，开箱即用。也不需要对子项目的开发部署做任何额外的操作。
3. `qiankun`框架使用`index.html`作为子项目的入口，会将里面的`style/link/script`标签以及注释代码解析并插入，但是他没有考虑`meta`和`title`标签，如果切换系统，其中`meta`标签有变化，则不会解析并插入，当然了，`meta`标签不影响页面展示，这样的场景并不多。而切换系统，修改页面的`title`，则需要通过全局钩子函数来实现。
4. `qiankun`框架不好实现`keep-alive`需求，因为解决`css/js`污染的办法就是删除子系统插入的标签和劫持`window`对象，卸载时还原成子系统加载前的样子，这与`keep-alive`相悖：`keep-alive`要求保留这些，仅仅是样式上的隐藏。
5. 微前端中子项目的入口文件常见的有两种方式：`JS entry` 和 `HTML entry`

纯`single-spa`采用的是`JS entry`，而`qiankun`既支持`JS entry`，又支持`HTML entry`。

`JS entry`的要求比较苛刻：

（1）将`css`打包到`js `里面

（2）去掉`chunk-vendors.js`，

（3）去掉文件名的`hash`值

（4）将入口文件(`app.js`)放置到`index.html`目录，其他文件不变，原因是要截取`app.js`的路径作为`publicPath`

| APP entry  | 优点                                    | 缺点                                                         |
| ---------- | --------------------------------------- | ------------------------------------------------------------ |
| JS entry   | 可以复用公共依赖(vue,vuex,vue-router等) | 需要各种打包配置配合，无法实现预加载                         |
| HTML entry | 简单方便，可以预加载                    | 多一层请求，需要先请求到HTML文件，再用正则匹配到其中的js和css，无法复用公共依赖(vue,vuex,vue-router等) |

我觉得可以将入口文件改为两者配合，使用一个对象来配置：

```json
{
  publicPath: 'http://www.baidu.com',
  entry: [
    "app.3249afbe.js"
    "chunk-vendors.75fba470.js",
  ],
  preload: [
    "about.3149afve.js",
    "test.71fba472.js",
  ]
}

```

这样既可以实现预加载，又可以复用公共依赖，并且不用修改太多的打包配置。难点在于如何将子系统需要的`js`文件写到配置文件里面去，有两个思路：方法1：写一个`node`服务，定期（或者子系统有更新时）去请求子系统的`index.html`文件，然后正则匹配到里面的`js`。方法2：子系统打包时，`webpack`会将生成的`js/css`文件的请求插入到`index.html`中（`HtmlWebpackPlugin`）,那么是否也可以将这些`js`文件的名称发送到服务器记录，但是有些静态`js`文件不是打包生成的就需要手动配置。

最后，有什么问题或者错误欢迎指出，互相成长，感谢！
