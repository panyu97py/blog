# 从0实现一个前端微服务（上）

## 前端微服务的几种实现方式

什么是前端微服务，网上大把的介绍，我就不啰嗦了，简单来说，就是把各个子项目整合到一起。

[《前端架构：从入门到微前端》](https://github.com/phodal/microfrontends#实施微前端的六种方式)这本书中介绍，微前端架构一般可以由以下几种方式进行：

1. 使用 HTTP 服务器的路由来重定向多个应用（也就是链接跳转）
2. 在不同的框架之上设计通讯、加载机制，诸如 Mooa 和 Single-SPA
3. 通过组合多个独立应用、组件来构建一个单体应用
4. iFrame。使用 iFrame 及自定义消息传递机制
5. 使用纯 Web Components 构建应用
6. 结合 Web Components 构建

其中比较常见的就是`iframe`和`single-spa`，这两者各有千秋。

## iframe和single-spa的优缺点

### iframe的优缺点

#### 缺点

1. 页面加载问题: 影响主页面加载，阻塞`onload`事件，本身加载也很慢，页面缓存过多会导致电脑卡顿。**(无法解决)**
2. 布局问题：`iframe`必须给一个指定的高度，否则会塌陷。解决办法：子系统实时计算高度并通过`postMessage`发送给主页面，主页面动态设置高度，修改子系统或者代理插入脚本。有些情况会出现多个滚动条，用户体验不佳。
3. 弹窗及遮罩层问题：只能在`iframe`范围内垂直水平居中，没法在整个页面垂直水平居中。
   - 解决办法1：通过与框架页面消息同步解决，将弹窗消息发送给主页面，主页面来弹窗，对原项目改动大且影响原项目的使用。
   - 解决办法2：修改弹窗的样式：隐藏遮罩层，修改弹窗的位置。修改的办法就是通过代理服务器插入css样式。
   - 补充：`iframe`里面的内容无法实现占满屏幕的弹窗（非全屏），他只能在iframe范围内全屏，无法跳出iframe的限制在主页面全屏，不过这种情况也很少。
4. 浏览器前进/后退问题：`iframe`和主页面共用一个浏览历史，`iframe`会影响页面的前进后退，大部分时候正常，`iframe`多次重定向则会导致浏览器的前进后退功能无法正常使用，不是全部页面都会出现，基本可以忽略。但是`iframe`页面刷新会重置（比如说从列表页跳转到详情页，然后刷新，会返回到列表页），因为浏览器的地址栏没有变化。
5. `iframe`的页面跳转到其他页面出问题，比如两个`iframe`之间相互跳转，直接跳转会只在`iframe`范围内跳转，所以必须通过主页面来进行跳转。不过`iframe`跳转的情况很少
6. 不同源的系统之间的通讯需要通过`postMessage`，存在一定的安全性

#### 优点

1. 完全隔离了`css`和`js`，避免了各个系统之间的样式和`js`污染
2. 可以在子系统完全不修改的情况下嵌入进来

### single-spa的优缺点

#### 缺点

1. `css`和`js`需要制定规范，进行隔离。否则容易造成全局污染，尤其是`vue`的全局组件，全局钩子。
2. **需要子系统配合修改**。但是不影响子系统独立开发部署，路由部分对子系统有一些改动，但是不影响功能。

#### 优点

1. 加载快，可以将所有系统共用的模块提取出来，实现按需加载，一次加载，其他的复用。
2. 修改子系统的样式，不需要代理服务器，直接修改，因为同属于一个`document`。
3. 用户体验好、快，内容的改变不需要重新加载整个页面，避免了不必要的跳转和重复渲染
4. `http`请求少，服务器压力小。

### single-spa和iframe对比

| 对比项   | single-spa                                                   | iframe                                                       | 补充                                                         |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 加载速度 | single-spa可以将所有系统共用的vue/vuex/vue-router等文件提取出来，只加载一次，各系统复用，加载速度很快，但是必须保证文件版本统一 | iframe会占用主系统的http通道，影响主系统的加载，加载速度很慢 | 两者都可以通过http缓存提高一定的加载速度，但是对于vue这些通用文件没法做cdn，因为内部系统很可能无法访问外网 |
| 兼容性   | single-spa只适用于vue、react、angular编写的系统，对一些jq写的老系统无能为力 | iframe则可以嵌入任何页面                                     |                                                              |
| 技术难度 | single-spa需要一定的技术储备，有一些学习成本                 | iframe门槛则很低，无需额外学习                               |                                                              |
| 局限性   | single-spa可以嵌入任何部件                                   | iframe只能嵌入页面，当然了也可以把一个部件单独写成一个页面   |                                                              |
| 改造成本 | single-spa一定要对子系统进行改造，但是改造的内容并不多很多，半小时即可完成 | iframe可以不对原系统进行改造，但是必须借助代理服务器进行插入脚本和css，增加了代理服务器也增加了系统的不稳定性（两台服务器中的任何一台挂掉都会导致系统不可用），服务器也需要成本。如对原系统进行改造，则工作量和single-spa相当 | 项目的源文件丢失或者其他一些无法改动源文件的情况，只能使用iframe |

补充：

1. 对于`SEO`，`iframe`无法解决，但是`single-spa`有办法解决（谷歌能支持单页应用的`SEO`，百度则需要`SSR`），但是内部系统，SEO的需求比较少。
2. `iframe`存在安全隐患，两个`iframe`页面互相引用则会导致无限嵌套`bug`，会导致页面卡死，目前只能通过代理服务器检查`iframe`页面内容来处理
3. 页面可以通过一些代码，不允许自己被`iframe`嵌入。这种情况下就只能选择其他的方案，淘宝京东腾讯等都曾设置过，代码如下：

```javascript
if(window.top !== window.self){ window.top.location = window.location;}

```

## iframe和single-spa的实现及简单原理

iframe很简单，一个标签就实现了。single-spa比较陌生，我会详细介绍。

### single-spa初探

以`vue`为例，`vue-cli4`生成的项目打包生成的`index.html`文件内容如下（精简了一些无关的内容）：

```html
<!DOCTYPE html>
<html lang=en>
<head>
  <meta charset=utf-8>
  <title>my-app</title>
  <link href=/js/about.6b1cbb89.js rel=prefetch>
  <link href=/css/app.c8c4d97c.css rel=preload as=style>
  <link href=/js/app.6a6f1dda.js rel=preload as=script>
  <link href=/js/chunk-vendors.164d8230.js rel=preload as=script>
  <link href=/css/app.c8c4d97c.css rel=stylesheet>
</head>
<body>
  <noscript>
    <strong>We're sorry but my-app doesn't work properly without JavaScript enabled. Please enable it to
      continue.</strong>
  </noscript>
  <div id=app></div>
  <script src=/js/chunk-vendors.164d8230.js> </script>
  <script src=/js/app.6a6f1dda.js> </script>
</body>
</html>

```

其中最核心的部分是：

```html
<link href=/css/app.c8c4d97c.css rel=stylesheet>
<div id=app></div>
<script src=/js/chunk-vendors.164d8230.js> </script>
<script src=/js/app.6a6f1dda.js> </script> 

```

猜想：能否借助`node`服务器，将子系统的`index.html`获取到，然后读取`HTML`，获取到这几个标签，返回给主系统，主系统直接插入到`body`中，能否呈现出子系统？

`node`代理代码实现，操作`DOM`使用的是`cheerio`插件:

```javascript
const http = require('http')
// 引入cheerio模块
const cheerio = require('cheerio')
const axios = require('axios')
const server = http.createServer(function (request, response) {
  //请求子系统服务器，获取到index.html文件
  axios.get('http://localhost/').then(res => {
    response.writeHead(200, { 
      'Content-Type': 'application/xml' ,
      'Access-Control-Allow-Origin': '*'
    })
    // 加载HTML字符串
    const $ = cheerio.load(res.data)
    $('link').each(function () {
      $(this).attr('href', 'http://localhost' + $(this).attr('href'))
    })
    $('script').each(function () {
      $(this).attr('src', 'http://localhost' + $(this).attr('src'))
    })
    const resp = $('body').prepend($('link[rel=stylesheet]')).html();
    response.end(resp)
  }).catch(e => {
    console.log(e)
  })
})
server.listen(8080)

```

需要注意的是：

1. 这里面的引入`js/css`文件路径都是相对路径，需要拼上子项目的前缀。
2. `v-html`插入的`DOM`片段，外链`script`不会生效，需要手动插入

结果是主系统中`#app`里面能渲染出子系统，但是`#app`里面动态生成的`HTML`，`img/video/audio`等文件的路径是相对的，所以会请求到主系统上，但是这些文件并不在主系统，所以会404，同样，按需加载的路由页面对应的`js/css`文件也是相对路径，会请求出错。**如果路由没按需加载，则不存在这个问题**

结论：可以实现微服务效果，但是需要解决文件相对路径的问题，`index.html`里面的`link/script`还可以手动加上，但是动态生成的`html`里面的`img/video/audio`等，以及按需加载路由页面对应的`js/css`无法通过代理服务器解决。

解决思路：

1. 这里面的`js/css/img/video`等都是相对路径，能否通过`webpack`打包，将这些路径全部打包成绝对路径？这样就可以解决文件请求失败的问题。
2. 能否像CDN一样，一个服务器挂了，会去其他服务器上请求对应文件。或者说服务器之间的文件共享，主系统上的文件请求失败会自动去子服务器上找到并返回。
3. 能否手动（或借助`node`）将子系统的文件全部拷贝到主项目服务器上，`node`监听子系统文件有更新，就自动拷贝过来，并且按`js/css/img`文件夹合并

查阅`webpack`和`vue-cli3`官网后发现：

默认情况下，`Vue CLI` 会假设你的应用是被部署在一个域名的根路径上，例如 `https://www.my-app.com/`。如果应用被部署在一个子路径上 `https://www.my-app.com/my-app/`，而你使用的是history模式的路由，对于url：`https://www.my-app.com/my-app/page1`，vue无法区分`my-app`是真实路径，而`page1`是路由参数，这个时候需要设置 `publicPath` 为 `/my-app/`，vue才能正确的请求文件资源和匹配路由。

这里可以将`vue-cli3`的 `publicPath`设置为`https://www.my-app.com/my-app/`，然后代码里面的`js/css/img/video`路径都会变成绝对路径，前缀是`https://www.my-app.com/my-app/`，这样就解决了`url`路径的问题。

这样就可以实现一个简单的`single-spa`应用，但是加载好的`Vue`子系统不会在切换到下一个系统的时候卸载掉，子系统过多则会导致卡顿，并且`css/js`污染的可能性增加，实用性不大。

## 最后

文章有什么疑问or错误，欢迎评论。下一篇文章：[从0实现一个single-spa的前端微服务（中）](../从0实现一个single-spa的前端微服务（中）/从0实现一个single-spa的前端微服务（中）.md)，包含完整的打包/开发调试流程，老项目如何改造等。
