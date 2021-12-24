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

import * as webpack from 'webpack'
import { META_TYPE } from '@tarojs/helper'

import { IBuildConfig, Func } from './utils/types'
import { printBuildError, bindProdLogger, bindDevLogger } from './utils/logHelper'
import buildConf from './webpack/build.conf'
import { Prerender } from './prerender/prerender'
import { isEmpty } from 'lodash'
import { makeConfig } from './webpack/chain'

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
## `@tarojs/mini-runner/src/webpack/build.conf.ts`

`@tarojs/mini-runner/src/index.ts` 中`export default`的`build`方法中使用`buildConf`方法初始化了`webpackConfig`及相关的配置。于是我们就顺着逻辑看看`buildConf`方法到底做了什么。

1. 使用`copy-webpack-plugin`实现了编译过程中文件的拷贝即 [`copy`](https://docs.taro.zone/docs/config-detail#copy) 配置。
2. 基于`framework`配置修改`react-dom`指向。

   在 `react` 体系中，`react` 库实现了 `ReactComponent` 和 `ReactElement` 的核心部分，而 `react-dom` 库负责通过操作 `DOM` 来实现 `react` 在浏览器中的渲染更新操作。在小程序中，并不能直接操作 `DOM` 树或者说没有传统的 `DOM` 树，此时直接使用 `react-dom` 则会导致报错。所以，`taro` 实现了一套在小程序上的 仿 `react-dom` 运行时，以保证 `React` 可以正常在小程序端渲染、更新节点。我们也可以这么理解，`react-dom` 是浏览器端的 `render`，`react-native` 是原生 APP 的 `render`，而 `@tarojs/react` 是小程序上的 `render`。`nervjs`同理。
3. 使用`webpack.DefinePlugin`实现了[`defineConstants`](https://docs.taro.zone/docs/config-detail#defineconstants) 配置。
4. `@tarojs/mini-runner/src/plugins/MiniSplitChunksPlugin.ts` 压缩主包大小。
5. 使用`@tarojs/mini-runner/src/plugins/BuildNativePlugin.ts`或`@tarojs/mini-runner/src/plugins/MiniPlugin.ts`将 `framework` 源文件转换为 `platform` 平台代码。
6. 使用 `mini-css-extract-plugin` 将所有的 `css` 文件提取到一个文件中
7. 使用`webpack.ProvidePlugin` 将运行时环境从浏览器环境切换到 `taro` 的运行时环境，比如将 `window` 替换成 [`@tarojs/runtime`](taro-runtime源码解析.md) 中导出的 `window`
8. 使用 `terser-webpack-plugin` 开启 `js` 代码压缩
9. 使用 `csso-webpack-plugin` 压缩 `css` 代码

```typescript

// ......

import getBaseConf from './base.conf'

// ......

export default (appPath: string, mode, config: Partial<IBuildConfig>): any => {
    const chain = getBaseConf(appPath)
    const {
        // ....
    } = config
    config.modifyComponentConfig?.(componentConfig, config)

    
    // taro 中 copy 配置的实现  https://docs.taro.zone/docs/config-detail#copy
    let { copy } = config
    
    // ..... 省略一些 copy-webpack-plugin 配置初始化的代码
    
    if (copy) {
        // 这边使用 copy-webpack-plugin 实现文件的拷贝
        plugin.copyWebpackPlugin = getCopyWebpackPlugin({ copy, appPath })
    }
    alias[taroJsComponents + '$'] = taroComponentsPath || `${taroJsComponents}/mini`
    if (framework === 'react') {
        // 使用 @tarojs/react 替代 react-dom
        alias['react-dom$'] = '@tarojs/react'
        
        // ...
        
    }
    if (framework === 'nerv') {
        // 使用 nervjs 替代 react-dom
        alias['react-dom'] = 'nervjs'
        alias.react = 'nervjs'
    }
    
    env.FRAMEWORK = JSON.stringify(framework)
    env.TARO_ENV = JSON.stringify(buildAdapter)
    
    // 从配置中读取运行时常量
    const runtimeConstants = getRuntimeConstants(runtime)
    
    // 合并运行时常量与 defineConstants 中配置的常量。
    const constantsReplaceList = mergeOption([processEnvOption(env), defineConstants, runtimeConstants])
    
    const entryRes = getEntry({
        sourceDir,
        entry,
        isBuildPlugin
    })
    const defaultCommonChunks = isBuildPlugin
        ? ['plugin/runtime', 'plugin/vendors', 'plugin/taro', 'plugin/common']
        : ['runtime', 'vendors', 'taro', 'common']
    let customCommonChunks = defaultCommonChunks
    if (typeof commonChunks === 'function') {
        customCommonChunks = commonChunks(defaultCommonChunks.concat()) || defaultCommonChunks
    } else if (Array.isArray(commonChunks) && commonChunks.length) {
        customCommonChunks = commonChunks
    }
    
    // 使用 webpack.DefinePlugin 实现全局变量
    plugin.definePlugin = getDefinePlugin([constantsReplaceList])

   // 主包大小优化
   if (optimizeMainPackage.enable) {
      plugin.miniSplitChunksPlugin = getMiniSplitChunksPlugin({
         exclude: optimizeMainPackage.exclude,
         fileType
      })
   }
    
   // 初始化 miniPlugin 插件配置
   const miniPluginOptions = {
      // ......
   }

   // 将 framework 源文件转换为 platform 平台代码
   plugin.miniPlugin = !isBuildNativeComp ? getMiniPlugin(miniPluginOptions) : getBuildNativePlugin(miniPluginOptions)

   // 使用 mini-css-extract-plugin 将所有的 css 文件提取到一个文件中
   plugin.miniCssExtractPlugin = getMiniCssExtractPlugin([{
      filename: `[name]${fileType.style}`,
      chunkFilename: `[name]${fileType.style}`
   }, miniCssExtractPluginOption])

   // 使用 webpack.ProvidePlugin 将运行时环境从浏览器环境切换到 taro 的运行时环境，比如将 window 替换成 @tarojs/runtime 中导出的 window
   plugin.providerPlugin = getProviderPlugin({
      window: ['@tarojs/runtime', 'window'],
      document: ['@tarojs/runtime', 'document'],
      navigator: ['@tarojs/runtime', 'navigator'],
      requestAnimationFrame: ['@tarojs/runtime', 'requestAnimationFrame'],
      cancelAnimationFrame: ['@tarojs/runtime', 'cancelAnimationFrame'],
      Element: ['@tarojs/runtime', 'TaroElement'],
      SVGElement: ['@tarojs/runtime', 'SVGElement']
   })

   const isCssoEnabled = !((csso && csso.enable === false))

   const isTerserEnabled = !((terser && terser.enable === false))

   if (mode === 'production') {
       
       // 使用 terser-webpack-plugin 开启代码压缩
      if (isTerserEnabled) {
         minimizer.push(getTerserPlugin([
            enableSourceMap,
            terser ? terser.config : {}
         ]))
      }

      // 使用 csso-webpack-plugin 压缩 js 代码
      if (isCssoEnabled) {
         const cssoConfig: any = csso ? csso.config : {}
         plugin.cssoWebpackPlugin = getCssoWebpackPlugin([cssoConfig])
      }
   }
    
   // 修改通用 webapck 配置
   chain.merge({
      mode,
      devtool: getDevtool(enableSourceMap, sourceMapType),
      entry: entryRes!.entry,
      output: getOutput(appPath, [{
         outputRoot,
         publicPath: '/',
         globalObject
      }, output]),
      target: createTarget({
         framework
      }),
      // rule 及对应 loader 配置，其中包括了相对比较重要的 miniTemplateLoader
      module: getModule(appPath,{
          // ......
      }),
      // 路径别名配置
      resolve: { alias },
      // webpack 插件配置
      plugin,
      optimization:{
          // ....
      }
   })

   switch (framework) {
      // 适配 vue 2.x 修改 webpack 配置
      case FRAMEWORK_MAP.VUE:
         customVueChain(chain)
         break
      // 适配 vue 3.x 修改 webpack 配置
      case FRAMEWORK_MAP.VUE3:
         customVue3Chain(chain)
         break
      default:
   }

   return chain
}
```

## `@tarojs/mini-runner/src/plugins/MiniPlugin.ts`

## `@tarojs/mini-runner/src/plugins/BuildNativePlugin.ts`

## `@tarojs/mini-runner/src/plugins/miniTemplateLoader.ts`

## 参考
[Taro 源码解读 - miniRunner 篇](https://github.com/a1029563229/blogs/blob/master/Source-Code/taro/4.md)
