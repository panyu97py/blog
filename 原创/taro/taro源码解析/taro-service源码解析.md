## `package.json`

```json
{
  "name": "@tarojs/service",
  "version": "3.3.17",
  "description": "Taro Service",
  "main": "index.js",
  "types": "types/index.d.ts",
  "scripts": {
    "build": "run-s clean prod",
    "dev": "tsc -w",
    "prod": "tsc",
    // ......
  }
  // ......
}
```

由`main`配置可知代码的入口文件为`index.js`,并声明了类型文件`types/index.d.ts`。

### `@tarojs/service/index.js`

```javascript
module.exports = require('./dist/index.js').default
module.exports.default = module.exports
```

### `@tarojs/service/src/index.ts`

```typescript
import Kernel from './Kernel'
import { TaroPlatformBase } from './platform-plugin-base'

export { Kernel }
export { TaroPlatformBase }
export default { Kernel, TaroPlatformBase }
```

经由入口文件引入及导出可以看出`@tarojs/service`主要实现的是`Kernel`，`TaroPlatformBase`



## `@tarojs/service/src/Kernel.ts`



### `Kernel.run`

> 从[taro-cli源码解析](taro-cli源码解析.md)我们可知，不论执行什么`taro`命令，最后都将调用`Kernel.run`

```typescript
export default class Kernel extends EventEmitter {
  // .....
 async run (args: string | { name: string, opts?: any }) {
   
   // 此处做兼容，既支持 name 无参调用，也支持对象带参调用
   let name
   let opts
   if (typeof args === 'string') {
     name = args
   } else {
     name = args.name
     opts = args.opts
   }
   
   // ...
   
   // 保存运行参数
   this.setRunOpts(opts)
   
   // 初始化相关配置（这是关键点）
   await this.init()
   
   // 执行钩子函数 onStart
   await this.applyPlugins('onStart')
   
   // 判断命令是否存在
   if (!this.commands.has(name)) {
     throw new Error(`${name} 命令不存在`)
   }
   
   // 处理 --help 的日志输出 例如：taro build --help
   if (opts?.isHelp) {
     
     // .......
     
     // 省略了输出帮助日志的逻辑
     
     // .......
     
     return
   }
   
   // 获取平台配置
   if (opts?.options?.platform) {
     opts.config = this.runWithPlatform(opts.options.platform)
   }
   
   // 执行钩子函数 modifyRunnerOpts
   await this.applyPlugins({
     name: 'modifyRunnerOpts',
     opts: {
       opts: opts?.config
     }
   })
   
   // 执行传入的命令
   await this.applyPlugins({
     name,
     opts
   })
 }
}
```



### `Kernel.init`

> 由`Kernel.run`调用，初始化项目配置、项目路径、项目预设命令及插件。

```typescript
export default class Kernel extends EventEmitter {
  
  //......
  
  async init () {
    this.debugger('init')
    
    // 初始化项目配置
    this.initConfig()
    
    // 初始化基础的项目路径
    this.initPaths()
    
    // 初始化项目预设命令及插件
    this.initPresetsAndPlugins()
    
    // 触发钩子
    await this.applyPlugins('onReady')
  }
  
  //......
}
```

### `kernel.initPresetsAndPlugins`

> 初始化项目预设命令及插件

```typescript
export default class Kernel extends EventEmitter {
  
  //......
  
  async initPresetsAndPlugins () {
    const initialConfig = this.initialConfig
    const allConfigPresets = mergePlugins(this.optsPresets || [], initialConfig.presets || [])()
    const allConfigPlugins = mergePlugins(this.optsPlugins || [], initialConfig.plugins || [])()
    this.debugger('initPresetsAndPlugins', allConfigPresets, allConfigPlugins)
    
    // 为 require 方法注册了 babel，为它加上一个钩子。此后，每当使用 require 加载.js、.jsx、.es 和.es6 后缀名的文件，就会先用 Babel 进行转码。
    process.env.NODE_ENV !== 'test' &&
    createBabelRegister({
      only: [...Object.keys(allConfigPresets), ...Object.keys(allConfigPlugins)]
    })
    this.plugins = new Map()
    this.extraPlugins = []
    
    // 加载所有的 presets 和 plugin，最后都以 plugin 的形式注册到 this.plugins Map 集合中
    this.resolvePresets(allConfigPresets)
    this.resolvePlugins(allConfigPlugins)
  }
  
  //......
}
```





### `Kernel.applyPlugins`

> 运行项目插件`fn`方法，这里使用`AsyncSeriesWaterfallHook`流水线执行`hook`我们考虑一个场景[`ctx.modifyWebpackChain`](https://docs.taro.zone/docs/plugin#%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B%E6%89%A9%E5%B1%95)是一个公共的钩子，用于编译中修改 `webpack` 配置。那么下一次修改的入参应当是上一次修改的结果。而`AsyncSeriesWaterfallHook`类似于 reduce，如果前一个 Hook 函数的结果 result !== undefined，则 result 会作为后一个 Hook 函数的第一个参数。

```typescript
export default class Kernel extends EventEmitter {
  // ......
  
 async applyPlugins (args: string | { name: string, initialVal?: any, opts?: any }) {
    let name
    let initialVal
    let opts
    if (typeof args === 'string') {
      name = args
    } else {
      name = args.name
      initialVal = args.initialVal
      opts = args.opts
    }
   
   	//......
   
    if (typeof name !== 'string') {
      throw new Error('调用失败，未传入正确的名称！')
    }
   
   	// 获取 plugin 注册的内容，即 registerCommand、registerPlatform、registerMethod 中配置的 name 获取注册的内容
   	// 在执行 registerCommand、registerPlatform、registerMethod 方法时会执行 this.ctx.hooks.set(hook.name, hooks.concat(hook))
    const hooks = this.hooks.get(name) || []
    
    // 创建流水线 - AsyncSeriesWaterfallHook 工程
    // 类似于 reduce，如果前一个 Hook 函数的结果 result !== undefined，则 result 会作为后一个 Hook 函数的第一个参数
    const waterfall = new AsyncSeriesWaterfallHook(['arg'])
    
    if (hooks.length) {
      const resArr: any[] = []
      
      // 循环 hooks 数组，将所有的钩子注册到 waterfall 中
      for (const hook of hooks) {
        waterfall.tapPromise({
          name: hook.plugin!,// 后置感叹号为 typescript 非空断言
          stage: hook.stage || 0,
          // @ts-ignore
          before: hook.before
        }, async arg => {
          const res = await hook.fn(opts, arg)
          // 如果是修改或事件 hook 则返回当前 hook 执行的返回值为后一个 hook 函数的第一个参数
          if (IS_MODIFY_HOOK.test(name) && IS_EVENT_HOOK.test(name)) {
            return res
          }
          // 如果是添加事件则 push 至返回值数组并返回该数组为为后一个 hook 函数的第一个参数
          if (IS_ADD_HOOK.test(name)) {
            resArr.push(res)
            return resArr
          }
          return null
        })
      }
    }
   
   	// 执行流水线任务。
    return await waterfall.promise(initialVal)
 }
  
  // .......
}
```



### `Kernel.initPluginCtx`

> 初始化插件上下文，暴露 `Kernel`类中的方法、插件注册的方法`this.methods`、`onReady`、` onStart`供插件调用。

```typescript
export default class Kernel extends EventEmitter {
  
  //......
  
  initPlugin (plugin: IPlugin) {
    const { id, path, opts, apply } = plugin
    
    // 初始化插件上下文
    const pluginCtx = this.initPluginCtx({ id, path, ctx: this })
    
    this.debugger('initPlugin', plugin)
    
    // 注册插件
    this.registerPlugin(plugin)
    
    // 执行插件函数
    apply()(pluginCtx, opts)
    
    // 校验插件参数
    this.checkPluginOpts(pluginCtx, opts)
  }
 
  initPluginCtx ({ id, path, ctx }: { id: string, path: string, ctx: Kernel }) {
    // 新建一个插件实例
    const pluginCtx = new Plugin({ id, path, ctx })
   
    const internalMethods = ['onReady', 'onStart']
    const kernelApis = [
      'appPath',
      'plugins',
      'platforms',
      'paths',
      'helper',
      'runOpts',
      'initialConfig',
      'applyPlugins'
    ]
    
    // 注册钩子
    internalMethods.forEach(name => {
      if (!this.methods.has(name)) {
        pluginCtx.registerMethod(name)
      }
    })
    
    // 使用 prox 包装插件上下文
    return new Proxy(pluginCtx, {
      
      // 拦截 插件上下文 get 方法
      get: (target, name: string) => {
        if (this.methods.has(name)) {
          const method = this.methods.get(name)
          
          // 如果是数组则返回遍历数组中函数并执行的方法（这是 modifyWebpackChain 方法注册的关键）
          // 这里如果注册的是一个钩子那么，这里就是回调函数向 ctx.hooks 注册可执行方法的地方。
          if (Array.isArray(method)) {
            return (...arg) => {
              method.forEach(item => {
                item.apply(this, arg)
              })
            }
          }
          // 这里存在疑问，因为从逻辑上看 method 一定是一个数据。也有可能是我哪里没看明白
          return method
        }
        
        // 这边判断 kernelApis 返回 Kernel 类中的方法供插件调用
        if (kernelApis.includes(name)) {
          return typeof this[name] === 'function' ? this[name].bind(this) : this[name]
        }
        return target[name]
      }
    })
  }
  
  //......
}
```

## `@tarojs/service/src/Plugin.ts`

> 插件类主要实现了：命令、适配平台、插件、插件方法的注册。这边实现比较特殊的就是插件方法的注册

```typescript
export default class Plugin {
  
  // .......
  
  register (hook: IHook) {
    if (typeof hook.name !== 'string') {
      throw new Error(`插件 ${this.id} 中注册 hook 失败， hook.name 必须是 string 类型`)
    }
    if (typeof hook.fn !== 'function') {
      throw new Error(`插件 ${this.id} 中注册 hook 失败， hook.fn 必须是 function 类型`)
    }
    const hooks = this.ctx.hooks.get(hook.name) || []
    hook.plugin = this.id
    this.ctx.hooks.set(hook.name, hooks.concat(hook))
  }
  
  // ......
  
  registerMethod (...args) {
    const { name, fn } = processArgs(args)
    const methods = this.ctx.methods.get(name) || []
    // fn 函数为 underfind 时则注册的是钩子，反之则是插件公共方法
    methods.push(fn || function (fn: (...args: any[]) => void) {
      this.register({
        name,
        fn
      })
    }.bind(this))
    this.ctx.methods.set(name, methods)
  }
  
  // ..........
  
}
```



## `@tarojs/service/src/platform-plugin-base.ts`

