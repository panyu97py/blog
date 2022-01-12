# 介绍

taro 源码解析这一块的博客文章与其说是技术分享博客，其实称之为我自己阅读源码的方式或笔记会比较合适。这些文章会比较枯燥无聊甚至阅读时还可能会越来越困。
这里建议与源码一起阅读。让我的笔记作为你思考的辅助。如果觉得不错就 [给个 star 吧](https://github.com/panyu97py/blog) 。



# 我的源码阅读顺序

> 由 `taro build xxx` 为切入点

![taro流程图.png](assets/taro流程图.png)





# Taro 仓库概览

## [仓库组成](https://taro-docs.jd.com/taro/docs/codebase-overview#仓库组成)

> 以下列表摘自`taro`官方文档介绍了 Taro 由哪些 NPM 包所组成，以及每个包的功能。
>
> 
>
> 并作为此次源码解析的目录

### [基础](https://taro-docs.jd.com/taro/docs/codebase-overview#基础)

| 路径                                         | 描述                                           |
| -------------------------------------------- | ---------------------------------------------- |
| [`@tarojs/cli`](taro-cli源码解析.md)         | CLI 工具                                       |
| [`@tarojs/service`](taro-service源码解析.md) | 插件化内核                                     |
| `@tarojs/taro-loader`                        | Webpack loaders                                |
| `@tarojs/helper`                             | 工具库，主要供 CLI、编译时使用                 |
| `@tarojs/runner-utils`                       | 工具库，主要供小程序、H5 的编译工具使用        |
| `@tarojs/shared`                             | 工具库，主要供运行时使用                       |
| `@tarojs/taro`                               | 暴露各端所需要的 Taro 对象                     |
| `@tarojs/api`                                | 和各端无关的 Taro API                          |
| `babel-preset-taro`                          | Babel preset                                   |
| `eslint-config-taro`                         | ESLint 规则                                    |
| `postcss-pxtransform`                        | PostCSS 插件，转换 `px` 为各端的自适应尺寸单位 |

### [小程序](https://taro-docs.jd.com/taro/docs/codebase-overview#小程序)

| 路径                                                 | 描述                                              |
| ---------------------------------------------------- | ------------------------------------------------- |
| [`@tarojs/mini-runner`](taro-mini-runner源码解析.md) | 小程序编译工具，主要用于设置、调用 Webpack        |
| `@tarojs/react`                                      | 基于 `react-reconciler` 的小程序专用 React 渲染器 |
| `@tarojs/runtime`                                    | 小程序运行时适配器核心                            |
| `@tarojs/plugin-platform-weapp`                      | 微信小程序插件                                    |
| `@tarojs/plugin-platform-alipay`                     | 支付宝小程序插件                                  |
| `@tarojs/plugin-platform-swan`                       | 百度小程序插件                                    |
| `@tarojs/plugin-platform-tt`                         | 字节跳动小程序插件                                |
| `@tarojs/plugin-platform-qq`                         | qq 小程序插件                                     |
| `@tarojs/plugin-platform-jd`                         | 京东小程序插件                                    |
| `@tarojs/plugin-html`                                | 支持使用 HTML 标签的插件                          |
| `postcss-html-transform`                             | PostCSS 插件，用于处理 HTML 标签的类名            |
| `@tarojs/plugin-react-devtools`                      | 支持使用 React DevTools 的插件                    |
| `@tarojs/extend`                                     | 类似 jQuery 的库                                  |

### [H5](https://taro-docs.jd.com/taro/docs/codebase-overview#h5)

| 路径                             | 描述                                     |
| -------------------------------- | ---------------------------------------- |
| `@tarojs/webpack-runner`         | H5 编译工具，主要用于设置、调用 Webpack  |
| `@tarojs/router`                 | H5 路由                                  |
| `@tarojs/taro-h5`                | H5 端根据微信小程序规范实现的 API        |
| `@tarojs/components`             | H5 组件库（Web Components 版本）         |
| `@tarojs/components-react`       | H5 组件库（React 版本）                  |
| `babel-plugin-transform-taroapi` | Babel 插件，让 API 可以被 `tree-shaking` |
| `postcss-plugin-constparse`      | PostCSS 插件，用于处理 `tabbar` 的高度   |

### [RN](https://taro-docs.jd.com/taro/docs/codebase-overview#rn)

| 路径                                                | 描述                                                         |
| --------------------------------------------------- | ------------------------------------------------------------ |
| `@tarojs/components-rn`                             | RN 组件库                                                    |
| `@tarojs/rn-runner`                                 | RN 编译工具，主要用于设置、调用 metro                        |
| `@tarojs/rn-style-transformer`                      | RN 样式转换工具，让 RN 支持sass、less、stylus、postcss       |
| `@tarojs/rn-supporter`                              | RN 基础 metro 配置                                           |
| `@tarojs/rn-transformer`                            | RN 应用入口及页面转换工具，让 RN 支持 Taro 定义的 app 及 page config |
| `@tarojs/router-rn`                                 | RN 路由                                                      |
| `@tarojs/runtime-rn`                                | RN 运行时封装                                                |
| `@tarojs/taro-rn`                                   | RN 端根据微信小程序规范实现的 API                            |
| `babel-plugin-transform-react-jsx-to-rn-stylesheet` | Babel 插件，让 jsx 支持 className 属性                       |
| `taro-css-to-react-native`                          | 将 css 转为 RN 的 stylesheet                                 |

### [其它](https://taro-docs.jd.com/taro/docs/codebase-overview#其它)

| 路径                 | 描述                         |
| -------------------- | ---------------------------- |
| `@tarojs/taroize`    | 小程序转 Taro 的编译器       |
| `@tarojs/with-weapp` | 小程序转 Taro 的运行时适配器 |
