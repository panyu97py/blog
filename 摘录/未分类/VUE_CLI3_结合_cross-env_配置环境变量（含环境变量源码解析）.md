# VUE CLI3 结合 cross-env 配置环境变量（含环境变量源码解析）

## 使用

package.json

```
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

```
VUE_APP_API_ENV=${API_ENV}
```

项目中取用

```
console.log(process.env.VUE_APP_API_ENV)
```

## 源码解析

vue cli3 中 [环境变量和模式](https://cli.vuejs.org/zh/guide/mode-and-env.html) 写到它可以通过获取`.env`、`.env.local`、`.env.[mode]`、`.env.[mode].local`文件中的内容定义环境变量。

比如：

```
VUE_APP_TITLE=TITLE
```

文档中写道：只有以 `VUE_APP_` 开头的变量会被 `webpack.DefinePlugin` 静态嵌入到客户端侧的包中。

那我们来看源码:

### [base.js](https://github.com/vuejs/vue-cli/blob/dev/packages/%40vue/cli-service/lib/config/base.js#L192)

> 引入方法`resolveClientEnv.js`,通过 [`webpack-chain`](https://github.com/neutrinojs/webpack-chain), 链式配置了`DefinePlugin`使变量静态嵌入到客户端侧的包中。

```
 const resolveClientEnv = require('../util/resolveClientEnv')
    webpackConfig
      .plugin('define')
        .use(require('webpack').DefinePlugin, [
          resolveClientEnv(options)
        ])
```

### [resolveClientEnv.js](https://github.com/vuejs/vue-cli/blob/dev/packages/%40vue/cli-service/lib/util/resolveClientEnv.js)

> 就是这个方法定义了环境变量必须为`VUE_APP_`开始的规则，那么问题来了`process.env`里的值是哪里来的，为什么我们定义在配置文件中的值会被写入到`process.env`里接着往下看。

```

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

```
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

```
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

```
VUE_APP_API_ENV=${API_ENV}
```

项目中取用

```
console.log(process.env.VUE_APP_API_ENV)
```

### 那么有人就会问了，这个在实际项目中的意义是什么？

假如公司有个项目共有三个环境：测试、预发、线上，而接口地址是需要通过环境变量判断的。并且我们希望在开发的过程中也可以调试线上或其他的服务接口且发布到线上的项目希望`mode`为`production`，本地调试的项目我们希望`mode`为`development`那我得新建多少配置文件？

%23%23%20%E4%BD%BF%E7%94%A8%0A%0Apackage.json%0A%60%60%60%0A%7B%0A%20%20%20%20...%0A%20%20%20%20%22scripts%22%3A%7B%0A%20%20%20%20%20%20%20%20%22serve%3Adev%22%3A%20%22cross-env%20API_ENV%3Ddev%20vue-cli-service%20serve%22%0A%20%20%20%20%20%20%20%20...%0A%20%20%20%20%7D%2C%0A%20%20%20%20%22dependencies%22%3A%20%7B%0A%20%20%20%20%20%20...%0A%20%20%20%20%20%20%22cross-env%22%3A%20%22%5E7.0.2%22%2C%0A%20%20%20%20%20%20...%0A%20%20%20%20%7D%2C%0A%20%20%20%20...%0A%7D%0A%60%60%60%0A.env%0A%60%60%60%0AVUE_APP_API_ENV%3D%24%7BAPI_ENV%7D%0A%60%60%60%0A%E9%A1%B9%E7%9B%AE%E4%B8%AD%E5%8F%96%E7%94%A8%0A%60%60%60%0Aconsole.log(process.env.VUE_APP_API_ENV)%0A%60%60%60%0A%0A%0A%23%23%20%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90%0A%0A%0Avue%20cli3%20%E4%B8%AD%20%5B%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E5%92%8C%E6%A8%A1%E5%BC%8F%5D(https%3A%2F%2Fcli.vuejs.org%2Fzh%2Fguide%2Fmode-and-env.html)%20%E5%86%99%E5%88%B0%E5%AE%83%E5%8F%AF%E4%BB%A5%E9%80%9A%E8%BF%87%E8%8E%B7%E5%8F%96%60.env%60%E3%80%81%60.env.local%60%E3%80%81%60.env.%5Bmode%5D%60%E3%80%81%60.env.%5Bmode%5D.local%60%E6%96%87%E4%BB%B6%E4%B8%AD%E7%9A%84%E5%86%85%E5%AE%B9%E5%AE%9A%E4%B9%89%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E3%80%82%0A%0A%0A%E6%AF%94%E5%A6%82%EF%BC%9A%0A%60%60%60%0AVUE_APP_TITLE%3DTITLE%0A%60%60%60%0A%E6%96%87%E6%A1%A3%E4%B8%AD%E5%86%99%E9%81%93%EF%BC%9A%E5%8F%AA%E6%9C%89%E4%BB%A5%20%60VUE_APP_%60%20%E5%BC%80%E5%A4%B4%E7%9A%84%E5%8F%98%E9%87%8F%E4%BC%9A%E8%A2%AB%20%60webpack.DefinePlugin%60%20%E9%9D%99%E6%80%81%E5%B5%8C%E5%85%A5%E5%88%B0%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BE%A7%E7%9A%84%E5%8C%85%E4%B8%AD%E3%80%82%0A%0A%E9%82%A3%E6%88%91%E4%BB%AC%E6%9D%A5%E7%9C%8B%E6%BA%90%E7%A0%81%3A%0A%0A%23%23%23%20%5Bbase.js%5D(https%3A%2F%2Fgithub.com%2Fvuejs%2Fvue-cli%2Fblob%2Fdev%2Fpackages%2F%2540vue%2Fcli-service%2Flib%2Fconfig%2Fbase.js%23L192)%0A%3E%E5%BC%95%E5%85%A5%E6%96%B9%E6%B3%95%60resolveClientEnv.js%60%2C%E9%80%9A%E8%BF%87%20%5B%60webpack-chain%60%5D(https%3A%2F%2Fgithub.com%2Fneutrinojs%2Fwebpack-chain)%2C%20%E9%93%BE%E5%BC%8F%E9%85%8D%E7%BD%AE%E4%BA%86%60DefinePlugin%60%E4%BD%BF%E5%8F%98%E9%87%8F%E9%9D%99%E6%80%81%E5%B5%8C%E5%85%A5%E5%88%B0%E5%AE%A2%E6%88%B7%E7%AB%AF%E4%BE%A7%E7%9A%84%E5%8C%85%E4%B8%AD%E3%80%82%0A%60%60%60%0A%20const%20resolveClientEnv%20%3D%20require('..%2Futil%2FresolveClientEnv')%0A%20%20%20%20webpackConfig%0A%20%20%20%20%20%20.plugin('define')%0A%20%20%20%20%20%20%20%20.use(require('webpack').DefinePlugin%2C%20%5B%0A%20%20%20%20%20%20%20%20%20%20resolveClientEnv(options)%0A%20%20%20%20%20%20%20%20%5D)%0A%60%60%60%0A%0A%0A%23%23%23%20%5BresolveClientEnv.js%5D(https%3A%2F%2Fgithub.com%2Fvuejs%2Fvue-cli%2Fblob%2Fdev%2Fpackages%2F%2540vue%2Fcli-service%2Flib%2Futil%2FresolveClientEnv.js)%0A%3E%20%E5%B0%B1%E6%98%AF%E8%BF%99%E4%B8%AA%E6%96%B9%E6%B3%95%E5%AE%9A%E4%B9%89%E4%BA%86%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E5%BF%85%E9%A1%BB%E4%B8%BA%60VUE_APP_%60%E5%BC%80%E5%A7%8B%E7%9A%84%E8%A7%84%E5%88%99%EF%BC%8C%E9%82%A3%E4%B9%88%E9%97%AE%E9%A2%98%E6%9D%A5%E4%BA%86%60process.env%60%E9%87%8C%E7%9A%84%E5%80%BC%E6%98%AF%E5%93%AA%E9%87%8C%E6%9D%A5%E7%9A%84%EF%BC%8C%E4%B8%BA%E4%BB%80%E4%B9%88%E6%88%91%E4%BB%AC%E5%AE%9A%E4%B9%89%E5%9C%A8%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E4%B8%AD%E7%9A%84%E5%80%BC%E4%BC%9A%E8%A2%AB%E5%86%99%E5%85%A5%E5%88%B0%60process.env%60%E9%87%8C%E6%8E%A5%E7%9D%80%E5%BE%80%E4%B8%8B%E7%9C%8B%E3%80%82%0A%60%60%60%0A%0A%2F%2F%20%E6%AD%A3%E5%88%99%E5%8C%B9%E9%85%8D%E4%BB%A5%20VUE_APP_%20%E5%BC%80%E5%A4%B4%E7%9A%84%20key%0Aconst%20prefixRE%20%3D%20%2F%5EVUE_APP_%2F%0A%0Amodule.exports%20%3D%20function%20resolveClientEnv%20(options%2C%20raw)%20%7B%0A%20%20const%20env%20%3D%20%7B%7D%0A%20%20%0A%20%20%2F%2F%20%E5%BE%AA%E7%8E%AF%20process.env%20%E7%9A%84%20key%0A%20%20Object.keys(process.env).forEach(key%20%3D%3E%20%7B%0A%20%20%0A%20%20%20%20%2F%2F%20%E5%8C%B9%E9%85%8Dkey%E7%AC%A6%E5%90%88%E6%AD%A3%E5%88%99%E6%88%96key%E7%AD%89%E4%BA%8ENODE_ENV%0A%20%20%20%20if%20(prefixRE.test(key)%20%7C%7C%20key%20%3D%3D%3D%20'NODE_ENV')%20%7B%0A%20%20%20%20%20%20env%5Bkey%5D%20%3D%20process.env%5Bkey%5D%0A%20%20%20%20%7D%0A%20%20%7D)%0A%20%20env.BASE_URL%20%3D%20options.publicPath%0A%0A%20%20if%20(raw)%20%7B%0A%20%20%20%20return%20env%0A%20%20%7D%0A%0A%20%20for%20(const%20key%20in%20env)%20%7B%0A%20%20%20%20env%5Bkey%5D%20%3D%20JSON.stringify(env%5Bkey%5D)%0A%20%20%7D%0A%20%20%0A%20%20%2F%2F%20%E8%BF%94%E5%9B%9E%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E9%85%8D%E7%BD%AE%0A%20%20return%20%7B%0A%20%20%20%20'process.env'%3A%20env%0A%20%20%7D%0A%7D%0A%60%60%60%0A%23%23%23%20%5BService.js%5D(https%3A%2F%2Fgithub.com%2Fvuejs%2Fvue-cli%2Fblob%2Fdev%2Fpackages%2F%2540vue%2Fcli-service%2Flib%2FService.js%23L89)%0A%3E%20%E4%BB%8E%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E4%B8%AD%E8%8E%B7%E5%8F%96%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E7%9A%84%E7%9B%B8%E5%85%B3%E4%BB%A3%E7%A0%81%E5%85%B6%E4%B8%AD%E5%85%B3%E9%94%AE%E7%9A%84%E6%98%AF%5B%60dotenv%60%5D(https%3A%2F%2Fgithub.com%2Fmotdotla%2Fdotenv%2Fblob%2Fmaster%2Flib%2Fmain.js)%E3%80%81%5B%60dotenv-expand%60%5D(https%3A%2F%2Fgithub.com%2Fmotdotla%2Fdotenv-expand%2Fblob%2Fmaster%2Flib%2Fmain.js)%20%0A%3E%0A%3E%60%20dotenv%60%3A%E5%B0%86%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E4%B8%AD%E7%9A%84%E5%8F%82%E6%95%B0%E8%AF%BB%E5%8F%96%E5%B9%B6%E5%86%99%E5%85%A5%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%0A%3E%0A%3E%60dotenv-expand%60%20%3A%20%E5%B0%86%60dotenv%60%E8%AF%BB%E5%8F%96%E7%9A%84%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E8%BF%9B%E8%A1%8C%E5%86%8D%E6%AC%A1%E5%A4%84%E7%90%86%E7%AD%9B%E9%80%89%E5%85%B6%E4%B8%AD%E4%BB%A5%60%24%7Bkey%7D%60%E5%AE%9A%E4%B9%89%E7%9A%84%E5%8F%98%E9%87%8F%EF%BC%8C%E5%B9%B6%E6%9F%A5%E8%AF%A2%60node%60%E7%9A%84%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E4%B8%AD%E6%98%AF%E5%90%A6%E5%8C%85%E5%90%AB%E5%AF%B9%E5%BA%94%E7%9A%84%60key%60%E5%81%87%E5%A6%82%E5%AD%98%E5%9C%A8%E5%B0%B1%E8%B5%8B%E5%80%BC%E3%80%82%0A%3E%0A%3E%E5%85%B7%E4%BD%93%E7%9A%84%E4%BB%A3%E7%A0%81%E6%88%91%E5%B0%B1%E4%B8%8D%E6%94%BE%E4%BA%86%E6%83%B3%E7%9C%8B%E5%8F%AF%E4%BB%A5%E7%82%B9%E4%B8%8A%E9%9D%A2%E7%9A%84%E9%93%BE%E6%8E%A5%E8%87%AA%E5%B7%B1%E7%9C%8B%E3%80%82%0A%60%60%60%0Aconst%20dotenv%20%3D%20require('dotenv')%0Aconst%20dotenvExpand%20%3D%20require('dotenv-expand')%0A...%0Amodule.exports%20%3D%20class%20Service%20%7B%0A%0A%20%20...%0A%0A%20%20%2F%2F%20%E5%8A%A0%E8%BD%BD%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%0A%20%20loadEnv%20(mode)%20%7B%0A%20%20%20%20const%20logger%20%3D%20debug('vue%3Aenv')%0A%20%20%20%20const%20basePath%20%3D%20path.resolve(this.context%2C%20%60.env%24%7Bmode%20%3F%20%60.%24%7Bmode%7D%60%20%3A%20%60%60%7D%60)%0A%20%20%20%20const%20localPath%20%3D%20%60%24%7BbasePath%7D.local%60%0A%0A%20%20%20%20const%20load%20%3D%20envPath%20%3D%3E%20%7B%0A%20%20%20%20%20%20try%20%7B%0A%20%20%20%20%20%20%20%20const%20env%20%3D%20dotenv.config(%7B%20path%3A%20envPath%2C%20debug%3A%20process.env.DEBUG%20%7D)%0A%20%20%20%20%20%20%20%20dotenvExpand(env)%0A%20%20%20%20%20%20%20%20logger(envPath%2C%20env)%0A%20%20%20%20%20%20%7D%20catch%20(err)%20%7B%0A%20%20%20%20%20%20%20%20%2F%2F%20only%20ignore%20error%20if%20file%20is%20not%20found%0A%20%20%20%20%20%20%20%20if%20(err.toString().indexOf('ENOENT')%20%3C%200)%20%7B%0A%20%20%20%20%20%20%20%20%20%20error(err)%0A%20%20%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%0A%20%20%20%20load(localPath)%0A%20%20%20%20load(basePath)%0A%0A%20%20%20%20%2F%2F%20by%20default%2C%20NODE_ENV%20and%20BABEL_ENV%20are%20set%20to%20%22development%22%20unless%20mode%0A%20%20%20%20%2F%2F%20is%20production%20or%20test.%20However%20the%20value%20in%20.env%20files%20will%20take%20higher%0A%20%20%20%20%2F%2F%20priority.%0A%20%20%20%20if%20(mode)%20%7B%0A%20%20%20%20%20%20%2F%2F%20always%20set%20NODE_ENV%20during%20tests%0A%20%20%20%20%20%20%2F%2F%20as%20that%20is%20necessary%20for%20tests%20to%20not%20be%20affected%20by%20each%20other%0A%20%20%20%20%20%20const%20shouldForceDefaultEnv%20%3D%20(%0A%20%20%20%20%20%20%20%20process.env.VUE_CLI_TEST%20%26%26%0A%20%20%20%20%20%20%20%20!process.env.VUE_CLI_TEST_TESTING_ENV%0A%20%20%20%20%20%20)%0A%20%20%20%20%20%20const%20defaultNodeEnv%20%3D%20(mode%20%3D%3D%3D%20'production'%20%7C%7C%20mode%20%3D%3D%3D%20'test')%0A%20%20%20%20%20%20%20%20%3F%20mode%0A%20%20%20%20%20%20%20%20%3A%20'development'%0A%20%20%20%20%20%20if%20(shouldForceDefaultEnv%20%7C%7C%20process.env.NODE_ENV%20%3D%3D%20null)%20%7B%0A%20%20%20%20%20%20%20%20process.env.NODE_ENV%20%3D%20defaultNodeEnv%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%20%20if%20(shouldForceDefaultEnv%20%7C%7C%20process.env.BABEL_ENV%20%3D%3D%20null)%20%7B%0A%20%20%20%20%20%20%20%20process.env.BABEL_ENV%20%3D%20defaultNodeEnv%0A%20%20%20%20%20%20%7D%0A%20%20%20%20%7D%0A%20%20%7D%0A%20%20%0A%20%20...%0A%20%0A%20%7D%0A%60%60%60%0A%0A%23%23%23%20%E7%BB%BC%E4%B8%8A%E6%89%80%E8%BF%B0%EF%BC%9A%E6%88%91%E4%BB%AC%E5%B0%B1%E5%8F%AF%E4%BB%A5%E8%BF%99%E6%A0%B7%E5%AE%9A%E4%B9%89%E6%88%91%E4%BB%AC%E7%9A%84%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%0A%0Apackage.json%0A%60%60%60%0A%7B%0A%20%20%20%20...%0A%20%20%20%20%22scripts%22%3A%7B%0A%20%20%20%20%20%20%20%20%22serve%3Adev%22%3A%20%22cross-env%20API_ENV%3Ddev%20vue-cli-service%20serve%22%0A%20%20%20%20%20%20%20%20...%0A%20%20%20%20%7D%2C%0A%20%20%20%20%22dependencies%22%3A%20%7B%0A%20%20%20%20%20%20...%0A%20%20%20%20%20%20%22cross-env%22%3A%20%22%5E7.0.2%22%2C%0A%20%20%20%20%20%20...%0A%20%20%20%20%7D%2C%0A%20%20%20%20...%0A%7D%0A%60%60%60%0A.env%0A%60%60%60%0AVUE_APP_API_ENV%3D%24%7BAPI_ENV%7D%0A%60%60%60%0A%E9%A1%B9%E7%9B%AE%E4%B8%AD%E5%8F%96%E7%94%A8%0A%60%60%60%0Aconsole.log(process.env.VUE_APP_API_ENV)%0A%60%60%60%0A%23%23%23%20%E9%82%A3%E4%B9%88%E6%9C%89%E4%BA%BA%E5%B0%B1%E4%BC%9A%E9%97%AE%E4%BA%86%EF%BC%8C%E8%BF%99%E4%B8%AA%E5%9C%A8%E5%AE%9E%E9%99%85%E9%A1%B9%E7%9B%AE%E4%B8%AD%E7%9A%84%E6%84%8F%E4%B9%89%E6%98%AF%E4%BB%80%E4%B9%88%EF%BC%9F%0A%0A%E5%81%87%E5%A6%82%E5%85%AC%E5%8F%B8%E6%9C%89%E4%B8%AA%E9%A1%B9%E7%9B%AE%E5%85%B1%E6%9C%89%E4%B8%89%E4%B8%AA%E7%8E%AF%E5%A2%83%EF%BC%9A%E6%B5%8B%E8%AF%95%E3%80%81%E9%A2%84%E5%8F%91%E3%80%81%E7%BA%BF%E4%B8%8A%EF%BC%8C%E8%80%8C%E6%8E%A5%E5%8F%A3%E5%9C%B0%E5%9D%80%E6%98%AF%E9%9C%80%E8%A6%81%E9%80%9A%E8%BF%87%E7%8E%AF%E5%A2%83%E5%8F%98%E9%87%8F%E5%88%A4%E6%96%AD%E7%9A%84%E3%80%82%E5%B9%B6%E4%B8%94%E6%88%91%E4%BB%AC%E5%B8%8C%E6%9C%9B%E5%9C%A8%E5%BC%80%E5%8F%91%E7%9A%84%E8%BF%87%E7%A8%8B%E4%B8%AD%E4%B9%9F%E5%8F%AF%E4%BB%A5%E8%B0%83%E8%AF%95%E7%BA%BF%E4%B8%8A%E6%88%96%E5%85%B6%E4%BB%96%E7%9A%84%E6%9C%8D%E5%8A%A1%E6%8E%A5%E5%8F%A3%E4%B8%94%E5%8F%91%E5%B8%83%E5%88%B0%E7%BA%BF%E4%B8%8A%E7%9A%84%E9%A1%B9%E7%9B%AE%E5%B8%8C%E6%9C%9B%60mode%60%E4%B8%BA%60production%60%EF%BC%8C%E6%9C%AC%E5%9C%B0%E8%B0%83%E8%AF%95%E7%9A%84%E9%A1%B9%E7%9B%AE%E6%88%91%E4%BB%AC%E5%B8%8C%E6%9C%9B%60mode%60%E4%B8%BA%60development%60%E9%82%A3%E6%88%91%E5%BE%97%E6%96%B0%E5%BB%BA%E5%A4%9A%E5%B0%91%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%EF%BC%9F%0A
