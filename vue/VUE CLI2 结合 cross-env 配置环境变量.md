## VUE CLI2 结合 cross-env 配置环境变量

### 安装

```bash
yarn add cross-env
npm install cross-env --save
```

### 相关配置

#### config/dev.env.js

```js
'use strict'
const merge = require('webpack-merge')
const prodEnv = require('./prod.env')

module.exports = merge(prodEnv, {
  NODE_ENV: '"development"',
  API_ENV: `"${process.env.API_ENV}"`
})
```

#### config/prod.env.js

```js
'use strict'
module.exports = {
  NODE_ENV: '"production"',
  API_ENV: `"${process.env.API_ENV}"`
}
```

#### package.json

```json
{
    //……
    "scripts":{
        "dev": "cross-env API_ENV=dev webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
        "prod": "cross-env API_ENV=prod webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
        "alpha": "cross-env API_ENV=alpha webpack-dev-server --inline --progress --config build/webpack.dev.conf.js",
        "start": "npm run dev",
        "lint": "eslint --fix --ext .js,.vue src",
        "build:dev": "cross-env API_ENV=dev node build/build.js",
        "build:prod": "cross-env API_ENV=prod node build/build.js",
        "build:alpha": "cross-env API_ENV=alpha node build/build.js",
        "build:alpha_analyz": "npm_config_report=true npm run build:alpha"
    },
     "dependencies": {
         //……
        "cross-env": "^5.2.0",
         //……
     }
     //……
}
```

### 项目中使用

```js
process.env.API_ENV
```

