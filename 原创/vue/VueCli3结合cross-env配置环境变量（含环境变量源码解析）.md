# VUE CLI3 结合 cross-env 配置环境变量（含环境变量源码解析）



## 着急写项目的同行们，写在前面拿走不谢。

package.json

```javascript
{
    ...
    "scripts":{
        "serve:dev": "cross-env API_ENV=dev vue-cli-service serve"
        ...
    },
    "dependencies": {
      ...
      "cross-env": "^7.0.2",
      ...
    },
    ...
}
```

.env

```javascript
VUE_APP_API_ENV=${API_ENV}
```

项目中取用

```javascript
console.log(process.env.VUE_APP_API_ENV)
```

## 源码解析

vue cli3 中 [环境变量和模式](https://cli.vuejs.org/zh/guide/mode-and-env.html) 写到它可以通过获取`.env`、`.env.local`、`.env.[mode]`、`.env.[mode].local`文件中的内容定义环境变量。

比如：

```javascript
VUE_APP_TITLE=TITLE
```

文档中写道：只有以 `VUE_APP_` 开头的变量会被 `webpack.DefinePlugin` 静态嵌入到客户端侧的包中。

那我们来看源码:

### [base.js](https://github.com/vuejs/vue-cli/blob/dev/packages/%40vue/cli-service/lib/config/base.js#L192)

> 引入方法`resolveClientEnv.js`,通过 [`webpack-chain`](https://github.com/neutrinojs/webpack-chain), 链式配置了`DefinePlugin`使变量静态嵌入到客户端侧的包中。

```javascript
 const resolveClientEnv = require('../util/resolveClientEnv')
    webpackConfig
      .plugin('define')
        .use(require('webpack').DefinePlugin, [
          resolveClientEnv(options)
        ])
```

### [resolveClientEnv.js](https://github.com/vuejs/vue-cli/blob/dev/packages/%40vue/cli-service/lib/util/resolveClientEnv.js)

> 就是这个方法定义了环境变量必须为`VUE_APP_`开始的规则，那么问题来了`process.env`里的值是哪里来的，为什么我们定义在配置文件中的值会被写入到`process.env`里接着往下看。

```javascript
// 正则匹配以 VUE_APP_ 开头的 key
const prefixRE = /^VUE_APP_/

module.exports = function resolveClientEnv (options, raw) {
  const env = {}
  
  // 循环 process.env 的 key
  Object.keys(process.env).forEach(key => {
  
    // 匹配key符合正则或key等于NODE_ENV
    if (prefixRE.test(key) || key === 'NODE_ENV') {
      env[key] = process.env[key]
    }
  })
  env.BASE_URL = options.publicPath

  if (raw) {
    return env
  }

  for (const key in env) {
    env[key] = JSON.stringify(env[key])
  }
  
  // 返回环境变量配置
  return {
    'process.env': env
  }
}
```

### [Service.js](https://github.com/vuejs/vue-cli/blob/dev/packages/%40vue/cli-service/lib/Service.js#L89)

> 从配置文件中获取环境变量的相关代码其中关键的是[`dotenv`](https://github.com/motdotla/dotenv/blob/master/lib/main.js)、[`dotenv-expand`](https://github.com/motdotla/dotenv-expand/blob/master/lib/main.js)
>
> `dotenv`:将配置文件中的参数读取并写入环境变量
>
> `dotenv-expand` : 将`dotenv`读取的环境变量进行再次处理筛选其中以`${key}`定义的变量，并查询`node`的环境变量中是否包含对应的`key`假如存在就赋值。
>
> 具体的代码我就不放了想看可以点上面的链接自己看。

```javascript
const dotenv = require('dotenv')
const dotenvExpand = require('dotenv-expand')
...
module.exports = class Service {

  ...

  // 加载环境变量
  loadEnv (mode) {
    const logger = debug('vue:env')
    const basePath = path.resolve(this.context, `.env${mode ? `.${mode}` : ``}`)
    const localPath = `${basePath}.local`

    const load = envPath => {
      try {
        const env = dotenv.config({ path: envPath, debug: process.env.DEBUG })
        dotenvExpand(env)
        logger(envPath, env)
      } catch (err) {
        // only ignore error if file is not found
        if (err.toString().indexOf('ENOENT') < 0) {
          error(err)
        }
      }
    }

    load(localPath)
    load(basePath)

    // by default, NODE_ENV and BABEL_ENV are set to "development" unless mode
    // is production or test. However the value in .env files will take higher
    // priority.
    if (mode) {
      // always set NODE_ENV during tests
      // as that is necessary for tests to not be affected by each other
      const shouldForceDefaultEnv = (
        process.env.VUE_CLI_TEST &&
        !process.env.VUE_CLI_TEST_TESTING_ENV
      )
      const defaultNodeEnv = (mode === 'production' || mode === 'test')
        ? mode
        : 'development'
      if (shouldForceDefaultEnv || process.env.NODE_ENV == null) {
        process.env.NODE_ENV = defaultNodeEnv
      }
      if (shouldForceDefaultEnv || process.env.BABEL_ENV == null) {
        process.env.BABEL_ENV = defaultNodeEnv
      }
    }
  }
  
  ...
 
 }
```

### 综上所述：我们就可以这样定义我们的环境变量

package.json

```json
{
    ...
    "scripts":{
        "serve:dev": "cross-env API_ENV=dev vue-cli-service serve"
        ...
    },
    "dependencies": {
      ...
      "cross-env": "^7.0.2",
      ...
    },
    ...
}
```

.env

```javascript
VUE_APP_API_ENV=${API_ENV}
```

项目中取用

```javascript
console.log(process.env.VUE_APP_API_ENV)
```

### 那么有人就会问了，这个在实际项目中的意义是什么？

假如公司有个项目共有三个环境：测试、预发、线上，而接口地址是需要通过环境变量判断的。并且我们希望在开发的过程中也可以调试线上或其他的服务接口且发布到线上的项目希望`mode`为`production`，本地调试的项目我们希望`mode`为`development`那我得新建多少配置文件？