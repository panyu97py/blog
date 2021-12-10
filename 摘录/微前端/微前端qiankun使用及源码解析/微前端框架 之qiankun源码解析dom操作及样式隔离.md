## 封装`dom`操作的原因

​		考虑一个场景，多例模式下由于沙盒的设置，微应用加载后执行微应用中的脚本所访问全局变量`window`是以`proxy`的方式进行。但是过生成`script`标签访问远程脚本时访问的就是真正的`window`。通过js生成`<style>`内容(无论内联还是远程)的时候样式文件的scoped并未绑定对应微应用，这会导致样式与脚本变量会影响到全局，

​		所以为了解决这一问题需要对远程加载的`script`、`link`、`style`标签做特殊处理，也就是对操作`dom`的方法`createElement`、`apend`、`insertBefore`进行封装。

## 源码解析

### 单例模式

单例模式中使用较为简单的`patchLooseSandbox`，其中与多例模式中最大的不同就是`patchHTMLDynamicAppendPrototypeFunctions`函数的入参。

```typescript
// src/sandbox/patchers/dynamicAppend/forLooseSandbox.ts

import { checkActivityFunctions } from 'single-spa';
export function patchLooseSandbox(
  appName: string,
  appWrapperGetter: () => HTMLElement | ShadowRoot,
  proxy: Window,
  mounting = true,
  scopedCSS = false,
  excludeAssetFilter?: CallableFunction,
): Freer {
	// .....  
  const unpatchDynamicAppendPrototypeFunctions = patchHTMLDynamicAppendPrototypeFunctions(
      // 判断是否为微服务调用，这里与多例模式不同，仅判断当前微应用是否运行
      () => checkActivityFunctions(window.location).some((name) => name === appName),
    	// 获取微服务容器配置，这里由于是单例所以直接返回当前微服务的配置
      () => ({
        appName,
        appWrapperGetter,
        proxy,
        strictGlobal: false,
        scopedCSS,
        dynamicStyleSheetElements,
        excludeAssetFilter,
      }),
    );
  	// .....  
}
```

### 多例模式

多例模式较为复杂劫持了`document.createElement`方法，用于收集各子应用的`script`、`link`、`style`标签及对应子应用的映射关系。

```typescript
// src/sandbox/patchers/dynamicAppend/forStrictSandbox.ts

declare global {
  interface Window {
    __proxyAttachContainerConfigMap__: WeakMap<WindowProxy, ContainerConfig>;
  }
}

// Get native global window with a sandbox disgusted way, thus we could share it between qiankun instances🤪
Object.defineProperty(nativeGlobal, '__proxyAttachContainerConfigMap__', { enumerable: false, writable: true });

// Share proxyAttachContainerConfigMap between multiple qiankun instance, thus they could access the same record
nativeGlobal.__proxyAttachContainerConfigMap__ =
  nativeGlobal.__proxyAttachContainerConfigMap__ || new WeakMap<WindowProxy, ContainerConfig>();
const proxyAttachContainerConfigMap = nativeGlobal.__proxyAttachContainerConfigMap__;

// script、link、style 标签与其对应子应用的映射表
const elementAttachContainerConfigMap = new WeakMap<HTMLElement, ContainerConfig>();

const docCreatePatchedMap = new WeakMap<typeof document.createElement, typeof document.createElement>();
                                        
function patchDocumentCreateElement() {
  const docCreateElementFnBeforeOverwrite = docCreatePatchedMap.get(document.createElement);

  // 判断 document.createElement 是否已经被劫持
  if (!docCreateElementFnBeforeOverwrite) {
    // 缓存原始 document.createElement 方法用于卸载时恢复
    const rawDocumentCreateElement = document.createElement;
    // 重写/劫持 document.createElement 方法
    Document.prototype.createElement = function createElement<K extends keyof HTMLElementTagNameMap>(
      this: Document,
      tagName: K,
      options?: ElementCreationOptions,
    ): HTMLElement {
      // 创建元素
      const element = rawDocumentCreateElement.call(this, tagName, options);
      // 判断标签类型是否为  script、link、style
      if (isHijackingTag(tagName)) {
        const { window: currentRunningSandboxProxy } = getCurrentRunningApp() || {};
        if (currentRunningSandboxProxy) {
          // 获取容器配置
          const proxyContainerConfig = proxyAttachContainerConfigMap.get(currentRunningSandboxProxy);
          if (proxyContainerConfig) {
            // 保存映射关系
            elementAttachContainerConfigMap.set(element, proxyContainerConfig);
          }
        }
      }

      return element;
    };

    // It means it have been overwritten while createElement is an own property of document
    if (document.hasOwnProperty('createElement')) {
      document.createElement = Document.prototype.createElement;
    }

    docCreatePatchedMap.set(Document.prototype.createElement, rawDocumentCreateElement);
  }

  // 恢复被重写的方法
  return function unpatch() {
    if (docCreateElementFnBeforeOverwrite) {
      Document.prototype.createElement = docCreateElementFnBeforeOverwrite;
      document.createElement = docCreateElementFnBeforeOverwrite;
    }
  };
}
```

基于各子应用的`script`、`link`、`style`标签及对应子应用的映射关系`elementAttachContainerConfigMap`，实现多例模式下`dom`操作的封装。

```typescript
export function patchStrictSandbox(
  appName: string,
  appWrapperGetter: () => HTMLElement | ShadowRoot,
  proxy: Window,
  mounting = true,
  scopedCSS = false,
  excludeAssetFilter?: CallableFunction,
): Freer {
  //...
    const unpatchDynamicAppendPrototypeFunctions = patchHTMLDynamicAppendPrototypeFunctions(
   	// 判断元素是否为微服务调用 
    (element) => elementAttachContainerConfigMap.has(element),
    // 获取元素对应的子服务配置
    (element) => elementAttachContainerConfigMap.get(element)!,
  );
  //...
}
```

### 重写`createElement`、`apend`、`insertBefore`

```typescript
// src/sandbox/patchers/dynamicAppend/common.ts

export function patchHTMLDynamicAppendPrototypeFunctions(
  isInvokedByMicroApp: (element: HTMLElement) => boolean,
  containerConfigGetter: (element: HTMLElement) => ContainerConfig,
) {
  // Just overwrite it while it have not been overwrite
  if (
    HTMLHeadElement.prototype.appendChild === rawHeadAppendChild &&
    HTMLBodyElement.prototype.appendChild === rawBodyAppendChild &&
    HTMLHeadElement.prototype.insertBefore === rawHeadInsertBefore
  ) {
    HTMLHeadElement.prototype.appendChild = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawHeadAppendChild,
      containerConfigGetter,
      isInvokedByMicroApp,
    }) as typeof rawHeadAppendChild;
    HTMLBodyElement.prototype.appendChild = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawBodyAppendChild,
      containerConfigGetter,
      isInvokedByMicroApp,
    }) as typeof rawBodyAppendChild;

    HTMLHeadElement.prototype.insertBefore = getOverwrittenAppendChildOrInsertBefore({
      rawDOMAppendOrInsertBefore: rawHeadInsertBefore as any,
      containerConfigGetter,
      isInvokedByMicroApp,
    }) as typeof rawHeadInsertBefore;
  }

  // Just overwrite it while it have not been overwrite
  if (
    HTMLHeadElement.prototype.removeChild === rawHeadRemoveChild &&
    HTMLBodyElement.prototype.removeChild === rawBodyRemoveChild
  ) {
    HTMLHeadElement.prototype.removeChild = getNewRemoveChild(
      rawHeadRemoveChild,
      (element) => containerConfigGetter(element).appWrapperGetter,
    );
    HTMLBodyElement.prototype.removeChild = getNewRemoveChild(
      rawBodyRemoveChild,
      (element) => containerConfigGetter(element).appWrapperGetter,
    );
  }

  return function unpatch() {
    HTMLHeadElement.prototype.appendChild = rawHeadAppendChild;
    HTMLHeadElement.prototype.removeChild = rawHeadRemoveChild;
    HTMLBodyElement.prototype.appendChild = rawBodyAppendChild;
    HTMLBodyElement.prototype.removeChild = rawBodyRemoveChild;

    HTMLHeadElement.prototype.insertBefore = rawHeadInsertBefore;
  };
}
```



### `script`、`link`、`style`标签的核心处理逻辑

`getOverwrittenAppendChildOrInsertBefore` 封装原生`apendChild`，`insertBefore`方法，在此过程中，
 若增加或插入的元素绑定了微应用，且绑定的应用是激活状态则

* `style`：则将其存在数组中，将元素插入到`mountDOM`中，`scrope`绑定`mountDOM`
*  `script`：则加载并执行脚本，将对应注释插入到`mountDOM`中



```tsx
// 获取重写的AppendChild， InsertBefore 方法，返回封装后的方法
// 用于挂载的时候
// 是shadow dom，且绑定的应用是激活状态，且为style，则将其存在数组中，将元素插入到mountDOM中，scoped绑定mountDOM
// 是shadow dom，且绑定的应用是激活状态，且为script，则加载并以proxy为上下文代替window执行脚本，将对应注释插入到mountDOM中
// 其他走正常流程插入
function getOverwrittenAppendChildOrInsertBefore(opts: {
    appName: string;
    proxy: WindowProxy;
    singular: boolean;
    dynamicStyleSheetElements: HTMLStyleElement[];
    appWrapperGetter: CallableFunction;
    rawDOMAppendOrInsertBefore: <T extends Node>(newChild: T, refChild?: Node | null) => T;
    scopedCSS: boolean;
    excludeAssetFilter?: CallableFunction;
  }) {
    return function appendChildOrInsertBefore(
      this: HTMLHeadElement | HTMLBodyElement,
      newChild: T,
      refChild?: Node | null,
    ) {
      let element = newChild as any;
      // 原生方法
      const { rawDOMAppendOrInsertBefore } = opts;

      // 这里传入的element可能是字符串？？正常dom对象是有tagName的
      if (element.tagName) {
        ..........

        // 如果element里面有微应用相关信息，说明当前dom操作是在微应用中，在之前createElement时缓存的信息
        const storedContainerInfo = element[attachElementContainerSymbol];

        // 如果要插入的元素是shadow dom，则将shadow dom里面携带的应用相关的配置信息覆盖掉上面参数传入的配置信息
        if (storedContainerInfo) {
          // 覆盖应用信息
        }
        // 检查element对应的应用是否是shadow dom且激活状态
        ..........
  
        switch (element.tagName) {
          // 如果是样式dom
          case LINK_TAG_NAME:
          case STYLE_TAG_NAME: {
            const stylesheetElement = newChild;

            // 如果不是shadow dom或者绑定的应用没激活，或者href被excludeAssetFilter排除在外
            // 则直接走正常插入方法
            if (!invokedByMicroApp || (excludeAssetFilter && href && excludeAssetFilter(href))) {
              return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
            }
  
            const mountDOM = appWrapperGetter();
  
  
            // 需要将新插入的style元素对应的css样式作用域绑定到mountDOM上
            if (scopedCSS) {
              css.process(mountDOM, stylesheetElement, appName);
            }
  
            // eslint-disable-next-line no-shadow
            // 将样式元素缓存起来，将来free，rebuild有用
            dynamicStyleSheetElements.push(stylesheetElement);

            // 将样式元素加入到mountDOM中
            const referenceNode = mountDOM.contains(refChild) ? refChild : null;
            return rawDOMAppendOrInsertBefore.call(mountDOM, stylesheetElement, referenceNode);
          }
  
          case SCRIPT_TAG_NAME: {
            const { src, text } = element;

            // some script like jsonp maybe not support cors which should't use execScripts
            // 如果不是shadow dom或者绑定的应用没激活，或者href被excludeAssetFilter排除在外
            // 则直接走正常插入方法
            if (!invokedByMicroApp || (excludeAssetFilter && src && excludeAssetFilter(src))) {
              return rawDOMAppendOrInsertBefore.call(this, element, refChild) as T;
            }
  
            const mountDOM = appWrapperGetter();
            const { fetch } = frameworkConfiguration;
            const referenceNode = mountDOM.contains(refChild) ? refChild : null;
            
            // 远程链接脚本执行，并触发onload事件，execScripts里面会将proxy作为执行环境上下文，import-html-entry里面
            if (src) {
              execScripts(null, [src], proxy, {
                fetch,
                strictGlobal: !singular,
                beforeExec: () => {
                  Object.defineProperty(document, 'currentScript', {
                    get(): any {
                      return element;
                    },
                    configurable: true,
                  });
                },
                // 手动触发onload事件
                success: () => {
                    element.onload(loadEvent) || element.dispatchEvent(loadEvent);
                },
                error: () => {
                    element.onerror(errorEvent) || element.dispatchEvent(errorEvent);
                },
              });
              // 将注释插入到mountDOM中
              const dynamicScriptCommentElement = document.createComment(`dynamic script ${src} replaced by qiankun`);
              return rawDOMAppendOrInsertBefore.call(mountDOM, dynamicScriptCommentElement, referenceNode);
            }
  
            // 内联脚本的执行
            execScripts(null, [`<script>${text}</script>`], proxy, {
              strictGlobal: !singular,
              success: element.onload,
              error: element.onerror,
            });
            // 将注释插入到mountDOM中
            const dynamicInlineScriptCommentElement = document.createComment('dynamic inline script replaced by qiankun');
            return rawDOMAppendOrInsertBefore.call(mountDOM, dynamicInlineScriptCommentElement, referenceNode);
          }
  
          default:
            break;
        }
      }
  
      // refChild为null，则为appendChild, 否则为insertBefore
      return rawDOMAppendOrInsertBefore.call(this, element, refChild);
    };
  }
```
