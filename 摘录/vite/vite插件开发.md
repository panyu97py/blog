## 前言

上一篇文章[Vite 开发实践 - 项目搭建](https://juejin.cn/post/7000182183344209934)中我们讲了 Vite 项目的环境搭建，主要补充了一下 vite 中脚手架搭建项目时的缺陷。而具体开发部分官方已经给了我们很好的体验和示例，一些比较特殊的用法比如**glob 目录扫描**、**静态资源导入**等推荐直接看官方文档就行了。本篇文章就主要讲一下如何在开发过程中编写相关插件来提高开发体验。

## Vite 中的插件机制

先附一段官方的话：

> Vite 插件扩展了设计出色的 Rollup 接口，带有一些 Vite 独有的配置项。因此，你只需要编写一个 Vite 插件，就可以同时为开发环境和生产环境工作。

我们可以看出 vite 的插件是基于`rollup`扩展的，通过插件机制我们可以对应用进行一系列的拓展，下面就简单介绍一下 vite 中的插件机制，

### 插件配置

vite 中使用插件很简单，只需要在`vite.config.ts`中的`plugins`选项中进行声明就行了。

```ts
import vitePlugin from 'vite-plugin-feature'
import rollupPlugin from 'rollup-plugin-feature'

export default defineConfig({
  plugins: [vitePlugin(), rollupPlugin()]
})
复制代码
```

每个插件的函数都需要返回一个带`name`字段的`Plugin`对象，该对象上可以包含有一些 vite 提供的插件钩子，vite 在对应执行时机会对其进行调用。

并且用于插件的假值会被 vite 自动忽略，所以如果是可选插件的话也不需要像`webpack`的配置一样需要先过滤可用的插件集合了。

### 插件钩子

上面我们提了下 vite 的插件钩子的相关概念，现在具体说下 vite 中到底有哪些钩子。

#### 兼容 rollup 的钩子

> vite 并没有兼容全部的 rollup 钩子，只有以下钩子有被调用，可以认为 vite 的开发服务器只调用了 `rollup.rollup()` 而没有调用 `bundle.generate()`。

- 以下钩子在服务器启动时被调用：

  - [`options`](https://link.juejin.cn?target=https%3A%2F%2Frollupjs.org%2Fguide%2Fen%2F%23options)：替换或操作传递给 rollup 的选项对象，当返回 null 不会替换任何内容。并且这是唯一一个无法访问大多数插件上下文实用程序功能的钩子，因为它在 rollup 完全配置之前运行。

    **注：** 如果我们仅仅想要访问传递给 rollup 的选项对象，推荐使用下面的`buildStart`。

    **类型定义：**

    ```ts
      interface PluginHooks {
          options: (
                      this: MinimalPluginContext,
                      options: InputOptions
              ) => Promise<InputOptions | null | undefined> | InputOptions | null | undefined;
      }
    复制代码
    ```

  - [`buildStart`](https://link.juejin.cn?target=https%3A%2F%2Frollupjs.org%2Fguide%2Fen%2F%23buildstart)：在每个 rollup 构建时调用。 当需要访问传递给 rollup 的选项时，这是推荐使用的钩子，因为它考虑了所有选项钩子的转换，并且还包含未设置选项的正确默认值。

    **类型定义：**

    ```ts
      interface PluginHooks {
          buildStart: (this: PluginContext, options: NormalizedInputOptions) => Promise<void> | void;
      }
    复制代码
    ```

- 以下钩子会在每个传入模块请求时被调用：

  - [`resolveId`](https://link.juejin.cn?target=https%3A%2F%2Frollupjs.org%2Fguide%2Fen%2F%23resolveid)：可用于定义自定义的 id 路径解析器，比如可以拿到`import foo from './foo.js'`中的`./foo.js`，一般用来定位第三方依赖。

    **类型定义：**

    ```ts
    interface Plugin {
        // 扩展了 rollup 的 resolveId，添加了 ssr 的 flag
        resolveId?(this: PluginContext, source: string, importer: string | undefined, options:{
      custom?: CustomPluginOptions;
      }, ssr?: boolean): Promise<ResolveIdResult> | ResolveIdResult;
    }
    复制代码
    ```

  - [`load`](https://link.juejin.cn?target=https%3A%2F%2Frollupjs.org%2Fguide%2Fen%2F%23load)：可用于定义自定义的模块解析 loader，比如可以直接返回一个已经转换好的 ast。

    **类型定义：**

    ```ts
    interface Plugin {
       // 扩展了 rollup 的 resolveId，添加了 ssr 的 flag
       load?(this: PluginContext, id: string, ssr?: boolean): Promise<LoadResult> | LoadResult;
    }
    复制代码
    ```

  - [`transform`](https://link.juejin.cn?target=https%3A%2F%2Frollupjs.org%2Fguide%2Fen%2F%23transform)，也可以返回 ast，但在这个时候已经拿到了具体路径文件的 code，所以一般用于转换已经加载后的模块，比如下面要讲的 **markdown 解析**就是基于该 hook。

    **类型定义：**

    ```ts
    interface Plugin {
       // 扩展了 rollup 的 resolveId，添加了 ssr 的 flag
       transform?(this: TransformPluginContext, code: string, id: string, ssr?: boolean): Promise<TransformResult_2> | TransformResult_2;
    }
    复制代码
    ```

- 以下钩子在服务器关闭时被调用：

  - [`buildEnd`](https://link.juejin.cn?target=https%3A%2F%2Frollupjs.org%2Fguide%2Fen%2F%23buildend)：构建完成时调用，可以拿到构建失败的错误信息。

    **类型定义：**

    ```ts
    interface PluginHooks {
       buildEnd: (this: PluginContext, err?: Error) => Promise<void> | void;
    }
    复制代码
    ```

  - [`closeBundle`](https://link.juejin.cn?target=https%3A%2F%2Frollupjs.org%2Fguide%2Fen%2F%23closebundle)：可在这个时期清理任何可能正在运行的外部服务。

    **类型定义：**

    ```ts
    interface PluginHooks {
    closeBundle: (this: PluginContext) => Promise<void> | void;
    }
    复制代码
    ```

#### Vite 独有钩子

> vite 的独有钩子会被 Rollup 忽略

- [`config`](https://link.juejin.cn?target=https%3A%2F%2Fcn.vitejs.dev%2Fguide%2Fapi-plugin.html%23config): 在解析 vite 配置前调用，可以再这里扩展或直接修改用户原始定义。

  **类型定义：**

  ```ts
  interface Plugin {
     config?: (config: UserConfig, env: ConfigEnv) => UserConfig | null | void | Promise<UserConfig | null | void>;
  }
  复制代码
  ```

- [`configResolved`](https://link.juejin.cn?target=https%3A%2F%2Fcn.vitejs.dev%2Fguide%2Fapi-plugin.html%23configresolved)：在解析 vite 配置后调用。使用这个钩子读取和存储最终解析的配置。

  **类型定义：**

  ```ts
  interface Plugin {
     configResolved?: (config: ResolvedConfig) => void | Promise<void>;
  }
  复制代码
  ```

- [`configureServer`](https://link.juejin.cn?target=https%3A%2F%2Fcn.vitejs.dev%2Fguide%2Fapi-plugin.html%23configureserver)：用于配置开发服务器的钩子，可在该 hook 访问开发服务器实例，可以用来添加自定义中间件拦截请求，比如下面要讲的 **mock 插件**就是基于此 hook 扩展。

  **类型定义：**

  ```ts
  export declare type ServerHook = (server: ViteDevServer) => (() => void) | void | Promise<(() => void) | void>;
  interface Plugin {
     configureServer?: ServerHook;
  }
  复制代码
  ```

- [`transformIndexHtml`](https://link.juejin.cn?target=https%3A%2F%2Fcn.vitejs.dev%2Fguide%2Fapi-plugin.html%23transformindexhtml)：转换 `index.html` 的专用钩子。 **类型定义：**

  ```ts
  export declare type IndexHtmlTransformHook = (html: string, ctx: IndexHtmlTransformContext) => IndexHtmlTransformResult | void | Promise<IndexHtmlTransformResult | void>;
  interface Plugin {
     transformIndexHtml?: IndexHtmlTransform;
  }
  复制代码
  ```

- [`handleHotUpdate`](https://link.juejin.cn?target=https%3A%2F%2Fcn.vitejs.dev%2Fguide%2Fapi-plugin.html%23handlehotupdate)：可用于自定义`hmr`更新处理。

  ```ts
  interface Plugin {
     handleHotUpdate?(ctx: HmrContext): Array<ModuleNode> | void | Promise<Array<ModuleNode> | void>;
  }
  复制代码
  ```

#### 钩子的执行顺序

插件的钩子执行基本也是同`rollup`一致的，只是在中间会穿插着 vite 独有的插件钩子。

**大致顺序如下：**

1. `config`
2. `configResolved`
3. `options`
4. `configureServer`：注意这是在初始化时，所以可以事先存储`configResolved`这类钩子的配置然后在这使用。
5. `buildStart`
6. `transformIndexHtml`
7. `load`
8. `resolveId`
9. `transform`
10. `buildEnd`：打包时才会触发。
11. `closeBundle`：打包时才会触发。

`handleHotUpdate`是在每次`hmr`触发时的钩子，并不太依靠内部顺序。

### 插件执行顺序

一个 Vite 插件可以额外指定一个 `enforce` 属性（类似于 webpack 加载器）来调整它的应用顺序。`enforce` 的值可以是`pre` 或 `post`。解析后的插件将按照以下顺序排列：

- Alias
- 带有 `enforce: 'pre'` 的用户插件
- Vite 核心插件
- 没有 enforce 值的用户插件
- Vite 构建用的插件
- 带有 `enforce: 'post'` 的用户插件
- Vite 后置构建插件（最小化，manifest，报告）

## 插件编写实践

前面已经说了 vite 中的插件机制和规范，现在就来动手编写两个插件吧。这边以**本地 mock 插件**和 **markdown 转换插件**为例子。

### 做一个本地 Mock 服务器插件

> 其实这个例子和 vite 之前的主要联系就只有配置开发服务器部分，但是本身也可以作为 mock 数据相关插件开发的一种思路，所以会尽量展示完整代码。

前端在本地 mock 数据可以帮助快速我们进行前端接口测试，并且等到后端真实接口开发完毕后可以无缝接入，这里主要分为三个方面来介绍如何编写一个本地 mock 插件`my-vite-plugin-mock`。

- vite 中的服务中间件
- mock 文件的编写规则
- 监听与加载本地 mock 文件

具体使用方法如下：

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import path from 'path'
import reactRefresh from '@vitejs/plugin-react-refresh'
import viteMockPlugin from 'my-vite-plugin-mock'

function resolve(relativePath: string) {
  return path.resolve(__dirname, relativePath)
}

export default defineConfig({
  plugins: [
    reactRefresh(),
    viteMockPlugin({
      dir: [resolve('./mock')] // 自动解析 mock 文件夹下的文件
    })
  ],
})
复制代码
```

然后在 mock 文件夹中创建符合规则的 mock 文件：

```js
// ./mock/hello.js
// 如果是 js 文件，因为 mock 服务器运行在 node 端，需要使用 commonjs
const userModule = {
  'get /hello': async () => {
    // 可以直接返回值给前端
    return {
      data: 'hello world'
    }
  }
}

module.exports = {
  default: userModule
}
复制代码
// ./mock/user.ts
// 同时支持 ts 文件，可以使用 ts 定义给与提示
import { Routes } from 'my-vite-plugin-mock'
function wait(time: number) {
  return new Promise((resolve) => {
    setTimeout(resolve, time)
  })
}

// 路由前缀，该文件中所有的路由都会加上这个前缀
export const prefix = '/user'

const userModule: Routes = {
  'get /getUserInfo': async ({ query }, req, res) => {
    // 第一个参数是已经被解析的上下文参数，有 query,params,body 等解析项，当然，为了灵活性，也提供给用户原生的 req 与 res
    await wait(2000)
    if (query.username) {
      return {
        data: {
          username: query.username
        }
      }
    }
    return {
      data: {
        username: 'xxx',
        email: 'xxx@xxx.com'
      }
    }
  }
}

export default userModule
复制代码
```

配置完毕后，插件会自动解析配置项并生成对应的路由，拦截对应路径的请求。

#### Vite 中的服务中间件

vite 的开发服务器使用 Node 的`http`模块搭建，并且引入了[`connect`](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fsenchalabs%2Fconnect)库作为中间件模块的拓展能力，同时 vite 还为我们提供的`configureServer`钩子来暴露`server`实例。

下面是具体的使用例子：

```ts
import { Plugin } from 'vite'
fcuntion myPlugin(): Plugin {
  return {
      name: 'configure-server',
      // 服务器实例
      configureServer(server) {
        // 添加中间件
        server.middlewares.use((req, res) => {
          // 自定义请求处理...
        })
        // 匹配前缀
        server.middlewares.use('/foo'，(req, res, next) => {
          // 自定义请求处理...
        })
     }
   }
}
复制代码
```

在加了自定义的中间件后，就可以拦截开发模式的请求并做进一步处理。

**注意：** 如果传入了第三个参数`next`，必须要手动调用，否则不会进入下一个中间件。

#### mock 文件的编写规则

为了能够便于 mock 数据与前端的通信，我们需要手动指定相关规则来让用户遵守。我这里**使用约定式的 mock 路径配置**和 **Typescript 的类型定义**进行限制。

- 约定式配置：我们需要手动指定一个 mock 目录，我们的插件会自动遍历整个目录并解析目录下的所有`js|ts`文件，每个文件都需要有一个默认导出的路由配置对象，并且还可以有一个可选的导出项`prefix`用于统一的路径前缀.

  就像下面这样：

  ```ts
  export const prefix = '/user'
  const userModule = {
  'get /getUserInfo': async () => {
      // coding
    }
  }
  export userModule
  复制代码
  ```

- 路由的类型定义：就像上面的`userModule`一样，我们需要创建一个限制该语法的`Routes`类型出来：

  ```ts
  import { Routes } from 'my-vite-plugin-mock'
  export const prefix = '/user'
  const userModule: Routes = {
  // 这里会限制键和值的书写
  'get /getUserInfo': async () => {
      // coding
    }
  }
  export userModule
  复制代码
  ```

  这里就直接给出类型定义了：

  ```ts
  // ./type.ts
  import http from 'http'
  import { Connect } from 'vite'
  // 解析数组
  type Item<T> = T extends Array<infer U> ? U : never
  // 支持的所有请求
  export type Methods = [
    'all',
    'get',
    'post',
    'put',
    'delete',
    'patch',
    'options',
    'head'
  ]
  export type MethodProps = Item<Methods>
  // 大小写兼容
  export type MethodsType = MethodProps | `${Uppercase<MethodProps>}`
  
  // 返回给前端 json 格式
  export interface HandlerResult {
    status?: number
    message?: string
    data?: any
  }
  
  // 因为原生 http 模块没有帮我们解析上下文，所以我们插件内部做一层解析，方便获取参数
  export interface HandlerContext {
    body: Record<string, any>
    query: Record<string, string | string[] | undefined>
    params: Record<string, string>
  }
  
  // 路由的 handle 函数
  export type RouteHandle = (
    ctx: HandlerContext,
    req: Connect.IncomingMessage,
    res: http.ServerResponse
  ) => Promise<HandlerResult | void> | HandlerResult | void
  
  // 路由 Map，键需要符合如 'get /xxx'这样的格式
  export type Routes = Record<`${MethodsType} ${string}`, RouteHandle>
  // 注意上面的类型要生效 typescript 的版本必须在 4.4 以上，如果因为项目原因不能升级到 4.4，请修改为下面这样：
  // export type Routes = Record<string, RouteHandle>
  复制代码
  ```

  > 因为在 [typescript 4.4 的更新](https://link.juejin.cn?target=https%3A%2F%2Fwww.typescriptlang.org%2Fdocs%2Fhandbook%2Frelease-notes%2Ftypescript-4-4.html%23symbol-and-template-string-pattern-index-signatures)中才支持了模板字符串和联合类型的索引签名，在 4.4 以下的版本如果这样写会有类型错误。

#### 加载 mock 文件

> 下面两步算是附加项，一般情况下我们其实也可以选择手动导入和合并配置项，但是这边为了以后拓展时 mock 文件时写法更轻松，这里还是加入了这个功能。

这里加载 mock 文件要考虑两个方面，**一方面是在服务启动的加载所有 mock 文件，还有一方面是在 mock 目录有文件更新时只需要加载更新的文件**。

所以我这边会写两个函数，**分别用于加载文件与目录（本质上都会调用加载文件的函数）**，加载文件时可以**使用`require`动态访问（但要注意我们还可以直接引用`ts`文件，所以还要加一层解析操作），在 mock 文件改变时重新执行`require`（注意这里要删除`require`的缓存）**。

不过首先，我们需要在前面的类型定义文件中添加两个类型：

```ts
// ./type.ts
// ...

// 内部的处理函数，不暴露给用户
export type MockRoutes = Record<
  `${MethodsType} ${string}`,
  {
    handler: RouteHandle
    method: MethodsType
  }
>
// 参考自 https://github.com/anncwb/vite-plugin-mock
// node 的模块处理能力，node 本身为了避免我们使用没有定义出来，但是这里我们明确要用到
export interface NodeModuleWithCompile extends NodeModule {
  _compile(code: string, filename: string): any
}
复制代码
```

下面是对于 mock 文件的全部解析函数：

```ts
// ./utils.ts
import * as fs from 'fs'
import { build } from 'esbuild'
import * as path from 'path'
import { MethodsType, MockRoutes, NodeModuleWithCompile, Routes } from './type'

export interface loadMockFilesOptions {
  // 监听的目录
  dir: string | string[]
  // 需包含或排除的文件
  include?: RegExp | ((filename: string) => boolean)
  exclude?: RegExp | ((filename: string) => boolean)
}

// 工具函数，用来匹配 include 和 exclude
export function matchFiles({
  file,
  include,
  exclude
}: {
  file: string
  include?: RegExp | ((filename: string) => boolean)
  exclude?: RegExp | ((filename: string) => boolean)
}): boolean {
  if (
    (exclude instanceof RegExp && exclude.test(file)) ||
    (typeof exclude === 'function' && exclude(file))
  ) {
    return false
  }
  if (
    include &&
    !(
      (include instanceof RegExp && include.test(file)) ||
      (typeof include === 'function' && include(file))
    )
  ) {
    return false
  }
  return true
}

// 加载所有文件，会在首次插件加载的时候运行
export async function loadMockFiles({
  dir,
  exclude,
  include
}: loadMockFilesOptions): Promise<MockRoutes | null> {
  let mockRoutes: MockRoutes | null = null
  // 判断数组和字符串分别加载目录
  if (Array.isArray(dir)) {
    mockRoutes = (
      await Promise.all(dir.map((d) => loadDir({ dir: d, exclude, include })))
    ).reduce((prev, next) => {
      return { ...prev, ...next }
    }, {} as MockRoutes)
  } else {
    mockRoutes = await loadDir({ dir, exclude, include })
  }
  return mockRoutes
}

// 加载目录
async function loadDir({
  dir,
  include,
  exclude
}: Omit<loadMockFilesOptions, 'dir'> & { dir: string }) {
  const mockRoutes: MockRoutes = {}
  if (fs.existsSync(dir)) {
    const files = fs.readdirSync(dir)
    const childMockRoutesArr = await Promise.all(
      files
        .map((file) => path.resolve(dir, file))
        .filter((file) => matchFiles({ include, exclude, file }))
        .map((file) => {
          const currentPath = path.resolve(dir, file)
          const stat = fs.statSync(currentPath)
          if (stat.isDirectory()) {
            // 递归加载
            return loadDir({
              include,
              exclude,
              dir: currentPath
            })
          } else {
            return loadFile(currentPath)
          }
        })
    )
    // 合并所有的 routes
    childMockRoutesArr.forEach((childMockRoutes) => {
      Object.keys(childMockRoutes).forEach((key) => {
        mockRoutes[key as keyof MockRoutes] =
          childMockRoutes[key as keyof MockRoutes]
      })
    })
  }
  return mockRoutes
}

// 加载文件，在监听文件改变的时候也会调用
export async function loadFile(filename: string) {
  const mockRoutes: MockRoutes = {}
  // 如果是目录直接返回空 routes
  if (fs.statSync(filename).isDirectory()) {
    return mockRoutes
  }
  // resolveModule 负责解析模块
  const { prefix: routePrefix, default: routes } = (await resolveModule(
    filename
  )) as {
    prefix?: string
    default: Routes
  }
  typeof routes === 'object' &&
    routes !== null &&
    Object.keys(routes).forEach((routeKey) => {
      const [method, routePath] = routeKey.split(' ')
      mockRoutes[path.join(routePrefix || '', routePath) as keyof MockRoutes] =
        {
          method: method as MethodsType,
          handler: routes[routeKey as keyof Routes]
        }
    })
  return mockRoutes
}

// 解析文件
async function resolveModule(filename: string): Promise<any> {
  // 如果是 ts 文件，用 esbuild 快速打包得到 js
  if (filename.endsWith('.ts')) {
    const res = await build({
      entryPoints: [filename],
      write: false,
      platform: 'node',
      bundle: true,
      format: 'cjs',
      target: 'es2015'
    })
    const { text } = res.outputFiles[0]
    // 改变 node 的解析规则
    return loadConfigFromBundledFile(filename, text)
  }
  // 一定要删除缓存，让 node 重新加载
  delete require.cache[filename]
  return require(filename)
}

async function loadConfigFromBundledFile(
  filename: string,
  bundle: string
): Promise<any> {
  const extension = path.extname(filename)
  const defaultLoader = require.extensions[extension]
  require.extensions[extension] = (module: NodeModule, fName: string) => {
    if (filename === fName) {
      ;(module as NodeModuleWithCompile)._compile(bundle, filename)
    } else {
      defaultLoader?.(module, fName)
    }
  }

  // 删除缓存
  delete require.cache[filename]
  const moduleValue = require(filename)
  // 改回原来的解析规则
  if (defaultLoader) {
    require.extensions[extension] = defaultLoader
  }
  return moduleValue
}
复制代码
```

**注：** 你可能注意到了加载`ts`文件的时候我们使用了`esbuild`来预处理文件，然后通过改变`require。extensions`的`lodaer`来解决 ts 的引入问题。

但是实际上`require.extensions`从`10.6`开始就被弃用了，因为注册时会导致整个程序的解析变慢，但是在这里由于我们只是开发模式使用，并且`require.extensions`虽然被弃用但是官方说法是可能永远也不会被移除，所以目前来说还是能作为解析方式使用的。

#### 监听 mock 文件改变

这里用`chokidar`来监听指定文件夹的变化，每次有文件变化重新加载对应的 mock 文件。

```ts
// ./index.ts
import * as chokidar from 'chokidar'
import type { WatchOptions } from 'chokidar'
import { Plugin } from 'vite'
export * from './type'
import { loadMockFiles, loadFile, matchFiles } from './utils'

export interface viteMockPluginOptions extends WatchOptions {
  dir: string[] | string
  /**
   * @description: 路径前缀
   * @default: /mock
   */
  mockPrefix?: string
  include?: RegExp | ((filename: string) => boolean)
  exclude?: RegExp | ((filename: string) => boolean)
}

function viteMockPlugin(options: viteMockPluginOptions): Plugin {
  const { dir } = options
  return {
    name: 'my-vite-plugin-mock',
    enforce: 'pre',
    apply: 'serve',
    async configureServer() {
      // 初始化加载所有 mock 文件
      let mockFiles = await loadMockFiles(options)
      // 监听目录
      chokidar.watch(dir, { ignoreInitial: true, ...options }).on('all',(action, file) => { // 监听所有行为，添加文件、修改文件等
         // file 为改变的的文件名
         if (
          // 这里也要判断是否符合匹配条件
          matchFiles({
            include: options.include,
            exclude: options.exclude,
            file
          })
        ) {
          // 缓存加载
          mockFiles = { ...mockFiles, ...(await loadFile(file)) }
        }
      })
     // http handler
    }
  }
}

export { viteMockPlugin }
export default viteMockPlugin
复制代码
```

#### 解析前端请求

在前面，我们学习了如何加载 mock 文件，并且将所有路由用对象的方式存储了起来，现在只需要解析前端的请求，将匹配的请求和路由对应起来就能完成我们的本地 mock 功能了。

```ts
// ./index.ts
import * as chokidar from 'chokidar'
import type { WatchOptions } from 'chokidar'
import { parse } from 'querystring'
import { pathToRegexp, match } from 'path-to-regexp'
import { Plugin } from 'vite'
export * from './type'
import { loadFile, loadMockFiles, matchFiles } from './utils'

export interface viteMockPluginOptions extends WatchOptions {
  dir: string[] | string
  /**
   * @description: 路径前缀
   * @default: /mock
   */
  mockPrefix?: string
  include?: RegExp | ((filename: string) => boolean)
  exclude?: RegExp | ((filename: string) => boolean)
}


function safeJsonParse<T extends Record<string | number | symbol, any>>(
  jsonStr: string,
  defaultValue: T
) {
  try {
    return JSON.parse(jsonStr)
  } catch (err) {
    return defaultValue
  }
}

// 解析 body 数据
function parseBody(
  req: Connect.IncomingMessage
): Promise<Record<string, any>> {
  return new Promise((resolve) => {
    let body = ''
    req.on('data', function (chunk) {
      body += chunk
    })
    req.on('end', function () {
      resolve(safeJsonParse(body, {}))
    })
  })
}


function viteMockPlugin(options: viteMockPluginOptions): Plugin {
  const { dir } = options
  return {
    name: 'my-vite-plugin-mock',
    enforce: 'pre',
    apply: 'serve',
    async configureServer({ middlewares }) {
      let mockFiles = await loadMockFiles(options)
      chokidar
        .watch(dir, { ignoreInitial: true, ...options })
        .on('all', async (_, file) => {
          if (
            matchFiles({
              include: options.include,
              exclude: options.exclude,
              file
            })
          ) {
            // 缓存加载
            mockFiles = { ...mockFiles, ...(await loadFile(file)) }
          }
        })
      middlewares.use(options.mockPrefix || '/mock', async (req, res) => {
        if (mockFiles) {
          const [url, search] = req.url!.split('?')
          // 遍历所有路由接口
          for (const [pathname, { handler, method }] of Object.entries(
            mockFiles
          )) {
           // 匹配路由
            if (
              pathToRegexp(pathname).test(url) &&
              req.method?.toLowerCase() === method.toLowerCase()
            ) {
              // 解析 query、params、body，方便操作
              // eslint-disable-next-line no-await-in-loop
              const body = await parseBody(req)
              const query = parse(search)
              const matched = match(pathname)(url)
              // eslint-disable-next-line no-await-in-loop
              const result = await handler(
                {
                  body,
                  query,
                  params:
                    (matched && (matched.params as Record<string, string>)) ||
                    {}
                },
                req,
                res
              )
              // 如果用户没有在 mock 函数中使用 res.end()，也就是直接 return 的数据
              if (!res.headersSent && result) {
                res.setHeader('Content-Type', 'application/json')
                res.statusMessage = result.message || 'ok'
                res.statusCode = result.status || 200
                res.end(
                  JSON.stringify({
                    message: 'ok',
                    status: 200,
                    ...result
                  })
                )
              }
              return
            }
          }
          // 如果都不匹配返回 404
          res.setHeader('Content-Type', 'application/json')
          res.statusMessage = '404 Not Found'
          res.statusCode = 404
          res.end(
            JSON.stringify({
              message: '404 Not Found',
              status: 404
            })
          )
        }
      })
    }
  }
}

export { viteMockPlugin }
export default viteMockPlugin
复制代码
```

至此，我们的 mock 插件终于完成了。代码有点多，在这就不全部放出来了，所有代码已经放在 [github](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FCol0ring%2Fvite-plugin-mock) 上，有需要的同学可以自取。

### 做一个 Markdown 转换插件

这个插件主要实现下面这些功能：

- 服务端（这里的服务端是开发模式下的`vite`服务器和打包时的`rollup`解析器）解析`md`文件，拦截生成可展示的`html`字符串与目录。
- 兼容 React 组件导入与懒加载（Vue 组件其实也能实现，不过由于笔者这边多是 React 技术栈，而且本文更多只是当做教程使用，所以 Vue 组件的解析各位可自行完善）。
- 能够自定义样式，可以使用 UI 组件统一样式。

具体使用方法很简单：

```ts
import { defineConfig } from 'vite'
import reactRefresh from '@vitejs/plugin-react-refresh'
import viteMdPlugin from '@col0ring/vite-plugin-md'

export default defineConfig({
  plugins: [
    reactRefresh(),
    viteMdPlugin()
  ]
})
复制代码
```

传入插件之后，会将`md`文件转换为`React Component`，我们就可以直接在页面中使用了。当然，该插件也会有一些可传入的参数的，我们在下面会接着说明。

#### 将 markdown 转换为 React 组件

要将`markdown`转换为 React 组件，首先我们需要将它转换为`html`字符串，我这里使用了比较成熟的第三方库 [`marked`](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fmarkedjs%2Fmarked)。

拿到`html`字符串后，如何将它渲染为 React 组件呢？思路很简单，提供 React 渲染的组件模板，然后将其渲染到模板中就行了，这里使用`art-template`来做模板渲染：

下面是一个`.art`模板文件：

```art
import React, { useEffect } from 'react'
// 自定义导入
{{if imports}}
  {{imports}}
{{/if}}

// 默认展示 jsx，支持服务端传入组件
const content = <>{{ content }}</>
// 原始的 html
const nativeContent = {{ nativeContent }}
// 是否用原始的 html
const native = {{ isNative }}
// 目录
const toc = {{ toc }}

const MarkdownComponent = ({ onLoad, className }) => {
  useEffect(() => {
    // 给外部提供一个获取内容的函数
    onLoad?.({
      toc,
      html: nativeContent
    })
  }, [])
  return (
    <div
      className={`${className ? className + ' ' : ''}vite-markdown`}
      dangerouslySetInnerHTML={native ? { __html: nativeContent } : undefined}
    >
      {native ? null : content}
    </div>
  )
}

// 暴露目录和原始 html 内容
export { toc, nativeContent as html }
export default MarkdownComponent
复制代码
```

渲染模板：

```ts
// ./utils.ts
import * as marked from 'marked'
import * as fs from 'fs'
import * as path from 'path'
import { render } from 'art-template'

// 目录
export interface TocProps {
  level: number
  text: string
  slug: string
}

// 渲染的 props
export interface RenderProps {
  content: string
  isNative: boolean
  nativeContent: string
  toc: string
  imports?: string
}


export interface MarkedRenderOptions {
  // 模板顶部的 imports
  imports?: string
  // marked 的配置项
  markedOptions?: marked.MarkedOptions
  // 是否是原生 html
  native?: boolean
}

const MarkdownComponent = fs
  .readFileSync(path.resolve(__dirname, '../templates/markdown-component.art'))
  .toString()

// 写一个转换函数
export function markdown2jsx(markdown: string, options: MarkedRenderOptions) {
  let renderer = options.markedOptions?.renderer
  if (!renderer) {
    renderer = new marked.Renderer()
  }
  // 因为要依靠 header 渲染的时候生成目录，所以这里要获取原来的 header
  const headingRender = renderer.heading
  // 目录
  const toc: TocProps[] = []
  renderer.heading = function (text, level, raw, slugger) {
    return headingRender.call(this, text, level, raw, {
      ...slugger,
      slug(...args) {
        // 唯一标识，在得到目录锚点的时候使用
        const res = slugger.slug(...args)
        toc.push({
          level,
          text,
          slug: res
        })
        return res
      }
    })
  }
  // 转换后的 html
  const content = marked(markdown, {
    xhtml: true,
    ...options.markedOptions,
    renderer
  })

  const renderOptions: RenderProps = {
    isNative: options.native || false,
    nativeContent: JSON.stringify(content) || '',
    content: options.native ? '' : content,
    imports: options.imports,
    toc: JSON.stringify(toc)
  }

  return {
    content: render(MarkdownComponent, renderOptions, {
      // 不要编码，原生输出
      escape: false
    })
  }
}
复制代码
```

经过这么一顿操作后，已经在服务端拿到了`React`组件了，但是浏览器是没办法解析`jsx`文件的，下面又该如何做呢？答案是，**我们再将转换后的`React`组件再手动打包成`js`字符串就行了，也就是需要使用运行时打包**，这里为了提高开发时的打包速度，使用了 vite 自带的`esbuild`来打包（这里不用太担心生产环境下的兼容性，因为在生产环境下还是会经过一层`rollup`打包的）。

```ts
import { transform } from 'esbuild'
import { markdown2jsx } from './utils'
// md => tsx
const { content } = markdown2jsx(code, options)
// tsx => js
transform(content, {
    loader: 'tsx',
    target: 'esnext',
    treeShaking: true
}).then({ code } => console.log(code))
复制代码
```

#### 拦截 md 文件

在之前我们说了，想要解析已经加载后的文件，一般使用`tranform`钩子，该钩子提供了源文件的内容与文件名，我们可以轻松地基于这两者做定制化解析：

```ts
// ./index.ts
import { Plugin } from 'vite'
import { SourceDescription } from 'rollup'
import { transform } from 'esbuild'
import { markdown2jsx, MarkedRenderOptions } from './utils'
// 匹配 markdown
const mdRegex = /\.md$/

export type mdPluginOptions = MarkedRenderOptions

function mdPlugin(options: mdPluginOptions = {}): Plugin {
  return {
    name: 'vite-jsx-md-plugin',
    enforce: 'pre',
    // 这里是额外的优化，我们可以官方提供的 react hmr 插件再对结果做一层 transform，提供开发模式的 hmr 功能
    configResolved({ plugins }) {
      const reactRefresh = plugins.find(
        (plugin) => plugin.name === 'react-refresh'
      )
      this.transform = async function (code, id, ssr) {
        if (mdRegex.test(id)) {
          // md => tsx
          const { content } = markdown2jsx(code, options)
          // tsx => js
          const { code: transformedCode } = await transform(content, {
            loader: 'tsx',
            target: 'esnext',
            treeShaking: true
          })

          // 再添加一层 hmr 代码
          // 仅用于开发模式，生产模式该值为空
          const sourceDescription = (await reactRefresh?.transform?.call(
            this,
            transformedCode,
            `${id}.tsx`,
            ssr
          )) as SourceDescription
          return (
            sourceDescription || {
              code: transformedCode,
              map: { mappings: '' }
            }
          )
        }
      } as Plugin['transform']
    }
  }
}

export default mdPlugin
复制代码
```

经过简单的处理，我们完成了一个具有`loader`功能的插件，该插件可以帮助我们将应用不能识别的`md`文件解析为`React`组件供我们在应用中使用。并且，由于我们是通过模板的方式生成的`React`组件，我们可以轻松地使用对组件的样式等进行调整：

```ts
import { defineConfig } from 'vite'
import reactRefresh from '@vitejs/plugin-react-refresh'
import viteMdPlugin from '@col0ring/vite-plugin-md'
import marked from 'marked'

const renderer = new marked.Renderer()

// 像下面这样，我们可以直接在这里使用 react-router 的 Link 组件替换 a 标签
renderer.link = (href, title, text) => {
  return `<Link to="${href}" title="${title}">${text}</Link>`
}

export default defineConfig({
  plugins: [
    reactRefresh(),
    viteMdPlugin({
      markedOptions: {
        renderer
      },
      // 在上面使用了 a 标签，在这里引入
      imports: `
      import { Link } from 'react-router-dom'
      `
    })
  ]
})
复制代码
```

该插件代码也放在 [github](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FCol0ring%2Fvite-plugin-md) 上了，各位有需要可以自行查看。

## 总结

本文从 vite 的插件机制开始说起，介绍了 vite 插件的相关配置与执行钩子。并在后续花了大量篇幅进行了插件实践，从零到一开发了两个不同用途的插件（一个主要用于开发模式提效，一个提供了`loader`解析能力）。当然，因为文中实际上是插件本身的逻辑代码偏多，而关于 vite 钩子的用法只占了一小部分，所以也可以看做是对这两个特殊插件的编写教程（逃）。

至于 vite 插件 API 更深入的用法，可以参考其他优秀的插件仓库学习使用，有兴趣的读者可以去官方帮我们整理的社区插件集 [awesome-vite](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fvitejs%2Fawesome-vite) 中寻找。

这里再多提一句，由于 vite 目前的生态还并不是和很完善，原生 vite 在解决一些定制化需求时和`webpack`还是有着一定地差距的，在很多时候**针对单个项目的专属插件开发甚至也要被列入项目开发的一部分**。这也是为什么我会将插件开发也列为 vite 开发实践的原因。

## 参考

- [rollup 插件文档](https://link.juejin.cn?target=https%3A%2F%2Frollupjs.org%2Fguide%2Fzh%2F%23plugin-development)
- [vite 官方文档](https://link.juejin.cn?target=https%3A%2F%2Fcn.vitejs.dev%2Fguide%2Fapi-plugin.html)
- [vite-plugin-mock](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fanncwb%2Fvite-plugin-mock)
- [vite-plugin-mdx](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fbrillout%2Fvite-plugin-mdx)


作者：Col0ring
链接：https://juejin.cn/post/7011305334690021412
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。