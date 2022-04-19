# 用rollup打包typescript编写的库

## [#](https://www.philxu.cn/gl-widget-tech/rollup.html#rollup-or-webpack)rollup or webpack

### [#](https://www.philxu.cn/gl-widget-tech/rollup.html#rollup)rollup

- rollup配置少，开箱即用，支持treeshaking，打包后比较干净
- 适合构建第三方库

### [#](https://www.philxu.cn/gl-widget-tech/rollup.html#webpack)webpack

- rollup能做的webpack也能做，但是添加额外的内容较多，配置相对繁琐
- 适合构建webapp

我要做的是webgl库，肯定选择rollup

## [#](https://www.philxu.cn/gl-widget-tech/rollup.html#所需插件)所需插件

```js
import typescript from 'rollup-plugin-typescript2'; // 处理typescript
import babel from 'rollup-plugin-babel'; // 处理es6
import resolve from '@rollup/plugin-node-resolve'; // 你的包用到第三方npm包
import commonjs from '@rollup/plugin-commonjs'; // 你的包用到的第三方只有commonjs形式的包
import builtins from 'rollup-plugin-node-builtins'; // 如果你的包或依赖用到了node环境的builtins fs等
import globals from 'rollup-plugin-node-globals'; // 如果你的包或依赖用到了globals变量
import { terser } from 'rollup-plugin-terser'; // 压缩，可以判断模式，开发模式不加入到plugins
```

## [#](https://www.philxu.cn/gl-widget-tech/rollup.html#发布配置)发布配置

```js
export default {
  input: 'src/index.ts', // 源文件入口
  output: [
    {
      file: 'dist/index.esm.js', // package.json 中 "module": "dist/index.esm.js"
      format: 'esm', // es module 形式的包， 用来import 导入， 可以tree shaking
      sourcemap: true
    }, {
      file: 'dist/index.cjs.js', // package.json 中 "main": "dist/index.cjs.js",
      format: 'cjs', // commonjs 形式的包， require 导入 
      sourcemap: true
    }, {
      file: 'dist/index.umd.js',
      name: 'GLWidget',
      format: 'umd', // umd 兼容形式的包， 可以直接应用于网页 script
      sourcemap: true
    }
  ],
  plugins: plugins
}
```

这样就可以同时发布3种格式的包供其他人选择使用

## [#](https://www.philxu.cn/gl-widget-tech/rollup.html#发布ts声明文件)发布ts声明文件

tsconfig.json

```json
{
  "compilerOptions": {
    "declaration": true // 生成*.d.ts
    ...
  }
  ...
}
```

如果rollup-plugin-typescript2没有额外配置的话，会在dist文件夹生成对应的声明文件，在package.json中指定types 字段，那其他人用typescript开发时就可以获取提示了

```json
{
  "types": "dist/index.d.ts"
  ...
}
```

## [#](https://www.philxu.cn/gl-widget-tech/rollup.html#自定义插件)自定义插件

在开发中我有个需求，想要模块化glsl，这样可以更好的组织shader代码，另一方面编写glsl文件可以获得编辑器的提示和高亮辅助

### [#](https://www.philxu.cn/gl-widget-tech/rollup.html#已有方案)已有方案

现在有比较知名的库glslify来做模块化，我为什么没用呢

- 因为我想做webgl库,把glslify整个打包进来没有必要，我的glsl应该在编译好后就不会有太大变更了

那怎么不用glslify的rollup插件在打包阶段解决呢

- glslify其实做的是模块化，我的需求更多的是类似include把代码插入的功能

那听起来和scss做的模块化类似，可以@import 变量进来，那我就仿照scss的方式写一个简单的rollup插件解决自己的需求

### [#](https://www.philxu.cn/gl-widget-tech/rollup.html#插件形式)插件形式

```js
function includeText(userOptions = {}) {
  return {
    name: 'include-text',
    async transform(source, id) { // hooks
      let transformedCode = xxx(souce) // 按你的方式改变code
      return { code: `export default ${JSON.stringify(transformedCode.toString())}`, map: { mappings: '' }};
    }
  }
module.exports = includeText;
```

#### [#](https://www.philxu.cn/gl-widget-tech/rollup.html#主要功能)主要功能

我主要用了transform这个hook，source是代码，id是这个代码对应的文件，我要做的就是

- 找到代码里的@import "**/*.glsl";
- 递归把代码替换到相应的位置
- 压缩glsl代码，去掉注释、空行、代码段的回车等
- 因为glsl代码不是js，为了后续正常处理，将代码转成string，export出去当做js的string变量来处理

#### [#](https://www.philxu.cn/gl-widget-tech/rollup.html#监听文件变化)监听文件变化

到这里基本功能就完成了，在使用中会发现，你修改js中import的glsl中代码，是可以触发自动编译打包的，但是glsl中import的glsl文件是无法触发的，那么就要用到addWatchFile这个api

```js
async transform(source, id) {
  this.addWatchFile(xxx)
}
```

递归的将所有找到的@import文件全部进行addWatchFile操作

## [#](https://www.philxu.cn/gl-widget-tech/rollup.html#总结)总结

以上就是在GLWidget项目中用到rollup相关内容，完成了用typescript编写，发布3种形式npm包的步骤，在过程中编写了处理glsl文件的插件。