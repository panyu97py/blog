## `package.json`

```json
{
  "name": "@tarojs/cli",
  "version": "3.3.17",
  "description": "cli tool for taro",
  "main": "index.js",
  "scripts": {
    // ...
    "build": "npm run clean && npm run prod",
    "dev": "tsc -w",
    "prod": "tsc",
    // ...
  },
  //...
  "bin": {
    "taro": "bin/taro"
  },
  // ...
}
```

由`main`配置可知代码的入口文件为`index.js`,`bin`配置可知`@tarojs/cli`模块注册了名为`taro`的可执行文件，具体代码在路径`bin/taro`。



## `@tarojs/cli/bin/taro.js`

```javascript
#! /usr/bin/env node

require('../dist/util').printPkgVersion() // 每次运行 taro 命令输出版本信心 ----> 例如：👽 Taro v3.3.16

// 运行 ../dist/cli 暴露的 CLI 类中的 run 方法，这里的 ../dist/cli 指的是 @tarojs/cli/src/cli.ts 通过 tsc 命令编译后的文件
const CLI = require('../dist/cli').default
new CLI().run()
```

`#!`在Linux或者Unix中叫做`shebang`,声明`@tarojs/cli/bin/taro`是一个可执行文件。`/usr/bin/env node`则表示用`/usr/bin/env`目录下的`node`运行这个可执行文件。



## `@tarojs/cli/src/cli.ts`

```typescript
import * as path from 'path'

import * as minimist from 'minimist' // 一个命令参数解析库 https://github.com/substack/minimist
import { Kernel } from '@tarojs/service'

import init from './commands/init'
import customCommand from './commands/customCommand'
import { getPkgVersion } from './util'

export default class CLI {
  appPath: string
  constructor (appPath) {
    this.appPath = appPath || process.cwd() // process.cwd() 指的是命令运行的当前路径
  }

  // 由 @tarojs/cli/bin/taro.js 执行
  run () {
    this.parseArgs()
  }
  
  parseArgs () {
    // 解析命令参数 ----> 例如：taro --version 获取的就是 --version 的值
    const args = minimist(process.argv.slice(2), {
      
      // 为命令声明别名，便于后续逻辑使用
      alias: {
        version: ['v'],
        help: ['h'],
        port: ['p'],
        resetCache: ['reset-cache'], // specially for rn, Removes cached files.
        publicPath: ['public-path'], // specially for rn, assets public path.
        bundleOutput: ['bundle-output'], // specially for rn, File name where to store the resulting bundle.
        sourcemapOutput: ['sourcemap-output'], // specially for rn, File name where to store the sourcemap file for resulting bundle.
        sourceMapUrl: ['sourcemap-use-absolute-path'], // specially for rn, Report SourceMapURL using its full path.
        sourcemapSourcesRoot: ['sourcemap-sources-root'], // specially for rn, Path to make sourcemaps sources entries relative to.
        assetsDest: ['assets-dest'] // specially for rn, Directory name where to store assets referenced in the bundle.
      },
      
      // 声明 version、help 为布尔值
      boolean: ['version', 'help']
    })
    const _ = args._
    const command = _[0] // 具体执行的命令 ----> 例如：taro run ... 这里的 command 的值就是 run
    if (command) {
      const kernel = new Kernel({
        appPath: this.appPath,
        presets: [
          path.resolve(__dirname, '.', 'presets', 'index.js')
        ]
      })
      switch (command) {
        
        // 执行 taro build ... 命令后所执行的逻辑
        case 'build': {
          let plugin
          let platform = args.type // 若运行 taro run --type weapp ，platform 的值就是 weapp
          const { publicPath, bundleOutput, sourcemapOutput, sourceMapUrl, sourcemapSourcesRoot, assetsDest } = args
          
          // 若执行 taro build --plugin [typeName] ... ，则表示当前编译的内容为插件修改 platform 的值为 plugin ， 并保存 plugin 的编译平台
          if (typeof args.plugin === 'string') {
            plugin = args.plugin
            platform = 'plugin'
          }
          kernel.optsPlugins = [
            '@tarojs/plugin-platform-weapp',
            '@tarojs/plugin-platform-alipay',
            '@tarojs/plugin-platform-swan',
            '@tarojs/plugin-platform-tt',
            '@tarojs/plugin-platform-qq',
            '@tarojs/plugin-platform-jd'
          ]
          customCommand('build', kernel, {
            // 一系列参数，此处非重点故省略
          })
          break
        }
          
        // 执行 taro init ... 命令后所执行的逻辑
        case 'init': {
          const projectName = _[1] || args.name
          init(kernel, {
            // 一系列参数，此处非重点故省略
          })
          break
        }
        default:
          customCommand(command, kernel, args)
          break
      }
    } else {
      if (args.h) {
        // 执行 taro --help 时，输出帮助信息
      } else if (args.v) {
        // 执行 taro --version 时，输出版本信息
      }
  }
}
```

由代码可见`@tarojs/cli`中涉及到`customCommand`、`Kernel`、`init`，但我们还不知到他们实现了什么功能。而`Kernels`是在另一个模块[`@tarojs/service`](taro-service源码解析.md)中实现的。



### `@tarojs/cli/src/commands/customCommand.ts`

```typescript
import { Kernel } from '@tarojs/service'
export default function init (kernel: Kernel,{
  // 一系列参数，此处非重点故省略
}:{
  // 一系列参数类型定义，此处非重点故省略
}){
  kernel.run({
    name: 'init',
    opts: {
      // 省略的一系列参数
    }
  })
}
```

### `@tarojs/cli/src/commands/init.ts`

```typescript
import { Kernel } from '@tarojs/service'

export default function customCommand (
  command: string,
  kernel: Kernel,
  args: { _: string[], [key: string]: any }
) {
  
  //...
  
  // 省略一系列参数处理逻辑
  
  // ...
  
  kernel.run({
    name: command,
    opts: {
     // 一系列参数，此处非重点故省略
    }
  })
}
```



由代码可见`customCommand`与`init`方法都执行了`kernel.run`。此时我们能发现`Kernels`在`@tarojs/cli`模块中是关键逻辑。那么此时我们应该去[`@tarojs/service`](taro-service源码解析.md)模块看看`Kernels`到底实现了什么。