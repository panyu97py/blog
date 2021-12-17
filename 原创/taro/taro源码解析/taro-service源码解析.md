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



### `Kernel.init`

```typescript
export default class Kernel extends EventEmitter {
  async init () {
    this.debugger('init')
    this.initConfig()
    this.initPaths()
    this.initPresetsAndPlugins()
    await this.applyPlugins('onReady')
  }
}
```



### `Kernel.run`

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





## `@tarojs/service/src/platform-plugin-base.ts`

