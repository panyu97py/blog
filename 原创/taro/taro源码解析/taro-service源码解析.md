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



## `@tarojs/service/src/platform-plugin-base.ts`