## 介绍

根据 `@tarojs/mini-runner` 代码库中`README.md`文件中的描述。`@tarojs/mini-runner`是暴露给 `@tarojs/cli` 的小程序 Webpack 启动器。

`@tarojs/mini-runner` 从 `@tarojs/cli` 接受 [Taro 编译配置](https://taro-docs.jd.com/taro/docs/config.html)，把编译配置分解成 Webpack 配置，并刚启动 Webpack 把项目源码编译为适配小程序目录结构的代码。

## `package.json`

```json
{
  "name": "@tarojs/mini-runner",
  "version": "3.3.17",
  "description": "Mini app runner for taro",
  "main": "index.js",
  "scripts": {
    "build": "npm run clean && npm run prod",
    "dev": "npm run mv:comp && tsc -w",
    "prod": "tsc && npm run mv:comp"
    // ....
  }
}
// ...
```
由`main`配置可知代码的入口文件为`index.js`。

## `@tarojs/mini-runner/src/index.ts`

基于[`@tarojs/service/src/platform-plugin-base.ts`](taro-service源码解析.md)中`getRunner`方法的调用可知，当执行`taro build ...`时运行的时指向的是`@tarojs/mini-runner`

而我们基于`package.json`入口文件的配置可知，最后执行的是`@tarojs/mini-runner/src/index.ts`文件中的`build`方法

`build`方法先是获取各种配置，最后通过 [`webpack`](https://webpack.docschina.org/api/node/) 进行编译。

```typescript

// ...

const customizeChain = async (chain, modifyWebpackChainFunc: Func, customizeFunc?: Func) => {
    if (modifyWebpackChainFunc instanceof Function) {
        await modifyWebpackChainFunc(chain, webpack)
    }
    if (customizeFunc instanceof Function) {
        customizeFunc(chain, webpack, META_TYPE)
    }
}

export default async function build (appPath: string, config: IBuildConfig): Promise<webpack.Stats> {
  const mode = config.mode

  // 获取 sass 配置
  const newConfig = await makeConfig(config)

  // 初始化 webpackChain
  const webpackChain = buildConf(appPath, mode, newConfig)

  // 通过 webpackChain 执行 modifyWebpackChain 自定义修改 webpack 配置 
  await customizeChain(webpackChain, newConfig.modifyWebpackChain, newConfig.webpackChain)

    
  // 注册 webpack 生命周期回调 onWebpackChainReady
  if (typeof newConfig.onWebpackChainReady === 'function') {
    newConfig.onWebpackChainReady(webpackChain)
  }

  // 将 webpackChain 转换为 webpackConfig
  const webpackConfig: webpack.Configuration = webpackChain.toConfig()

  return new Promise<webpack.Stats>((resolve, reject) => {
      
    // 通过 Webpack 提供的 Node.js API 获取 Compiler 实例。https://webpack.docschina.org/api/node/
    const compiler = webpack(webpackConfig)
    
    // .......

    const callback = async (err: Error, stats: webpack.Stats) => {
        // .....
    }

    // 通过 webpack 编译
    if (newConfig.isWatch) {
      bindDevLogger(compiler)
      compiler.watch({
        aggregateTimeout: 300,
        poll: undefined
      }, callback)
    } else {
      bindProdLogger(compiler)
      compiler.run(callback)
    }
  })
}

```
