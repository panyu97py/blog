# 前言

最近开始研究`Antd Form`(`V4`)，在研究源码后再次进行手写复盘，让我学到了很多设计逻辑以及代码技巧。因此，我也写了这篇文章来总结我的手写过程。在下面的过程中，我会以**增量开发**的模式去逐渐实现`Antd Form V4.17.2`的核心功能。我把开发分成三个阶段如下所示，每个阶段都实现部分重要特性：

1. 实现数据管理(把控件隐式处理成受控组件)
2. 实现用户交互功能(`onSubmit`、`onReset`、`onChange`)
3. 实现获取表单实例(`Form.useForm`)

通过增量开发，到最后实现一个简单可用的`mini Antd Form`，带大家由浅入深地去研究透`Antd Form`的内部运行逻辑以及为啥要这么设计。

**题外话：什么是增量开发**

增量开发是基于**增量模型**去构造产品的一种开发模式，其基于已知产品所有需要实现的特性的前提下，把特性进行拆分成几个阶段的目标，在每个阶段都完成部分特性，直至所有阶段完成后把产品进行交付。

> **增量模型 (Incremental Model)** 是您在部分中构建整个解决方案的地方，但是在每个阶段或部分结束时您没有, 任何可以审查或反馈的东西。您需要等到增量过程的最后阶段才能交付最终产品。

![image.png](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/3ef1530a6f544abe8454aba7e49ecc59~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

图片和引用来源：[增量和迭代开发有什么区别？](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Fchktsang%2Farticle%2Fdetails%2F87010449)

# 开始手写

## 第一阶段：实现数据管理

什么是**数据状态管理**，这里的**数据状态**指的是表单中变量的集合，例如下面的`form`表单：

```html
<form>
  <input type="text" name="name" />
  <input type="email" name="email" />
</form>
```

那这个`form`表单的数据就是用于记录`name`和`email`对应的值的变量，其通常被设计成一个对象如下所示：

```js
{
    name: undefined,
    email: undefined
}
```

在实现**数据状态管理**时，我们先依次思考两个问题：

1. **表单字段组件**里的**控件**是设计成**受控组件**还是**非受控组件**？

   先针对问题里的某些术语解释一下，**表单字段组件**指的是`Form.Item`，而里面的**控件**指的是`Form.Item`里的子组件，例如下面的`Input`组件实例就是**控件**：

   ```tsx
   <Form.Item
     label="Username"
     name="username"
     rules={[{ required: true, message: 'Please input your username!' }]}
   >
     <Input />
   </Form.Item>
   ```
   
   首先`React`官方是推荐**控件**设计成**受控组件**。`React`官方推荐一篇用于探讨在`form`中到底用**受控组件**还是**非受控组件**的文章: [Controlled and uncontrolled form inputs in React don't have to be complicated](https://link.juejin.cn?target=https%3A%2F%2Fgoshacmd.com%2Fcontrolled-vs-uncontrolled-inputs-react%2F)，其中的结论是，其实如果是简单的交互场景下，两者都可以。但对于一些复杂的交互场景，使用**受控组件**可以更简洁地实现其逻辑。该文章还列出一些常见的交互场景如下所示：
   
   | 特性                           | 非受控组件 | 受控组件 |
   | ------------------------------ | ---------- | -------- |
   | 在提交的时候取值               | ✅          | ✅        |
   | 在提交的时候校验值             | ✅          | ✅        |
   | 在值变化时进行即时验证         | ❌          | ✅        |
   | 根据值动态禁止提交按钮         | ❌          | ✅        |
   | 约束输入格式，不能输入非法字符 | ❌          | ✅        |
   | 一个值受多个表单元素控制       | ❌          | ✅        |
   | 根据值动态控制表单元素的显示   | ❌          | ✅        |
   
   由于`Antd Form`是公共组件，其设计需要考虑到各种场景，因此我们把**控件**设计成**受控组件**。
   
2. **数据状态**要放在哪里？

   在`Antd Form V3`中，**数据状态**放在`Form`组件上，使用`setState`对其进行更新。但这有一个缺陷：每当**数据状态**中的一个值发生变化时，由于`setState`的调用，会导致整个`Form`组件及其子组件都会进行更新，这对于一些庞大的表单而言会产生性能上的负担。

   由此，`Antd Form V4`对该缺陷进行了改善。把**数据状态**放到一个`formStore`中（管理公共状态的对象，类似于`Redux`中的`store`）。而该`formStore`都通过`React Context`注入到`Form`的子组件上。每当需要更新**数据状态**时，则调用`formStore`中指定的方法（此称`updateValue`）进行更新。

   阅读到此你会有新的疑问：调用`updateValue`对**数据状态**进行更新后，要如何让视图中对应的**控件**也进行更新？毕竟**控件**是**受控组件**。受控组件的注入值如`value`不更新，显示的值就不会有变化。

   对此，`formStore`除了会存储**数据状态**外，还会用一个变量（此称`fieldEntities`）存储每一个表单字段组件即`Form.Item`的实例。每当`updateValue`被调用后，就会遍历`fieldEntities`且执行其更新的方法（此称`onStateChange`），`onStateChange`会根据**数据状态**前后值以及`props`上的一些值（例如`name`、`shouldUpdate`、`dependencies`）去判断是否需要调用`forceUpdate`来更新组件。为什么是要遍历`fieldEntities`以执行所有实例的`onStateChange`方法？因为有时候**数据状态**中某个值的变化会影响好几个**表单字段组件**的变化，因为有些实例是通过`shouldUpdate`或`dependencies`这些`Form.Item`上的属性来对该值进行依赖。

   对于上面说的**数据状态**的触发更新流程，我们可以画出下面的流程图：

   ![image.png](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/376f4950eaae46b8a359b35d249d2185~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

------

okay!需要思考的问题已经解决，现在就直接上手代码吧。基于增量开发的模式下，在第一阶段，我们先实现可以完成**数据状态管理**的表单。

要写的文件主要有三个:

- **Form.tsx**: `Form`表单组件
- **FormStore.ts**: `FormStore`管理公共状态的对象
- **Field.tsx**: `Form.Item`表单字段组件

三者的层级关系如下图所示：

![image.png](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/8390be5ced4045a29cccc6b65973c947~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

首先先开始写`Form`组件

```tsx
// Form.tsx
import { useMemo, useRef } from 'react';
import FieldContext from './FieldContext';
import { FormStore } from './FormStore';
import type { Store } from './typings';

type BaseFormProps = Omit<React.FormHTMLAttributes<HTMLFormElement>, 'onSubmit'>;

// 第一阶段的props需要实现的参数只有initialValues、children
export interface FormProps<Values = any> extends BaseFormProps {
  initialValues?: Store;
  children?: React.ReactNode;
}

const Form: React.FC<FormProps> = ({ initialValues, children }) => {
  // FormStore的实例就是上面说到的管理“数据状态”和“fieldEntities”的对象，
  // 我们用useRef使其在组件的整个生命周期内持续存在。
  const formStore = useRef<FormStore>(new FormStore());

  // 通过mountRef让下面的逻辑仅在组件首次加载时执行
  const mountRef = useRef(false);
  if (initialValues) {
    // setInitialValues用于存放initialValues，存放initialValues的目的在于reset时候把initialValues赋值给store，
    // 第二个参数为true时，调用setInitialValues更新formStore内部的initialValues同时也会更新store，
    // store就是上面所说的存放“数据状态”的对象变量
    formStore.current.setInitialValues(initialValues, !mountRef.current);
  }
  if (!mountRef.current) {
    mountRef.current = true;
  }

  // 创建fieldContextValue用于注入到下面的FieldContext，
  // 使得Form中的子组件都能访问formStore
  const fieldContextValue = useMemo(
    () => ({
      formStore: formStore.current,
    }),
    [],
  );

  const wrapperNode = (
    <FieldContext.Provider value={fieldContextValue}>{children}</FieldContext.Provider>
  );

  return <form>{wrapperNode}</form>;
};

export default Form;
```

接下来就写`FormStore`

```ts
// FormStore.ts
import type { FieldEntity, NotifyInfo, Store, ValuedNotifyInfo } from './typings';

export class FormStore {
  // 保存数据状态的变量
  private store: Store = {};
  // 保存Form表单中的Form.Item实例
  private fieldEntities: FieldEntity[] = [];
  // 保存初始值，该初始值会受Form.props.initalValues和Form.Item.props.initalValues影响
  private initialValues: Store = {};

  // 设置initialValues，如果init为true，则顺带更新store
  public setInitialValues = (initialValues: Store | undefined, init: boolean) => {
    this.initialValues = initialValues || {};
    if (init) {
      this.store = { ...this.store, ...initialValues };
    }
  };

  // 根据name获取store中的值
  public getFieldValue = (name: string) => {
    return this.store[name];
  };

  // 获取整个store
  public getFieldsValue = () => {
    return { ...this.store };
  };

  // 内部更新store的函数
  public updateValue = (name: string | undefined, value: any) => {
    if (name === undefined) return;
    const prevStore = this.store;
    this.store = { ...this.store, [name]: value };
    this.notifyObservers(prevStore, [name], {
      type: 'valueUpdate',
      source: 'internal',
    });
  };

  // 获取那些带name的Form.Item实例
  private getFieldEntities = () => {
    return this.fieldEntities.filter((field) => field.props.name);
  };

  // 往fieldEntities注册Form.Item实例，每次Form.Item实例在componentDidMount时，都会调用该函数把自身注册到fieldEntities上
  // 最后返回一个解除注册的函数
  public registerField = (entity: FieldEntity) => {
    this.fieldEntities.push(entity);

    return () => {
      this.fieldEntities = this.fieldEntities.filter((item) => item !== entity);
    };
  };

  // Form.Item实例化时，在执行constructor期间会调用该函数以更新initialValue
  public initEntityValue = (entity: FieldEntity) => {
    const { initialValue, name } = entity.props;
    if (name !== undefined) {
      const prevValue = this.store[name];

      if (prevValue === undefined) {
        this.store = { ...this.store, [name]: initialValue };
      }
    }
  };

  // 生成更新信息mergedInfo且遍历所有的Form.Item实例调用其onStoreChange方法去判断是否需要更新执行
  private notifyObservers = (
    prevStore: Store,
    namePathList: string[] | undefined,
    info: NotifyInfo,
  ) => {
    const mergedInfo: ValuedNotifyInfo = {
      ...info,
      store: this.getFieldsValue(),
    };
    this.getFieldEntities().forEach(({ onStoreChange }) => {
      onStoreChange(prevStore, namePathList, mergedInfo);
    });
  };
}
```

对于上面`notifyObservers`方法中的第三个形参的类型`NotifyInfo`。我们可以看他的`typescript interface`定义：

```ts
interface ValueUpdateInfo {
  type: 'valueUpdate';
  source: 'internal' | 'external';
}
interface ResetInfo {
  type: 'reset';
}

// 这里我们先定义了两个Info类型，ValueUpdateInfo是内部和外部值更新时的信息类型，ResetInfo是重置时的信息类型
// onStateChange执行期间会根据Info的type做相应的处理，其处理过程就类似于Redux Reducer
export type NotifyInfo = ValueUpdateInfo | ResetInfo;
```

最后我们来写`Form.Item`的内容，**Field.tsx**文件中需要实现两个类: `WrapperField`和`Field`，我们依次来写这两个类：

`WrapperField`:

```tsx
// 除了fieldContext，其他都是在此阶段中，Field.props需要实现的变量
export interface FieldProps {
  name?: string;
  label?: string;
  initialValue?: any;
  children?: React.ReactElement;
  valuePropName?: string;
  trigger?: string;
  getValueFromEvent?: (...args: any[]) => any;
  fieldContext: FieldContextValues;
}

// 这里WrapperField作用主要在于把FieldContext的值提取出来注入到Field
// 为什么不直接在Field中获取fieldContext呢？原因如下：
// Field由于要把自身实例注册到formStore.fieldEntities里，因此自身设计成类组件而非函数组件，
// 而在类函数中获取FieldContext中的fieldContext有两种方法：Context.Provider 和 contextType
//    1. Context.Provider获取的fieldContext只能在jsx中使用，很不便
//    2. contextType只能针对只有一个Context的情况。在真实源码中有多个Context，如针对FormProvider的FormContext和size的SizeContext
// 因此，这里提前把fieldContext提取出来注入到Field上
function WrapperField(props: Omit<FieldProps, 'fieldContext'>) {
  const fieldContext = React.useContext(FieldContext);
  return (
    // 这里简单写一下label的布局，因为不是我们这篇文章探讨的重点，真实源码上使用antd的Row和Col做布局的。
    <div style={{ display: 'flex', marginBottom: 12 }}>
      <div style={{ width: 100 }}>{props.label}</div>
      <Field {...props} fieldContext={fieldContext} />
    </div>
  );
}
```

`Field`，在手写`Field`类之前需要明确其要如何处理**控件**：我们通过`React.cloneElement`向**控件**的`props`隐式混入`value`和`onChange`，从而使**控件**变成**受控组件**，当然这两个值可以分别通过`Form.Item.props`里的`valuePropName`和`trigger`设置。而`value`是注入`formStore`的`store`里对应的值，而`onChange`会注入一个自定义的方法，在其方法里会调用`formStore.updateValue`。其关系图如下所示：

![image.png](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/c726b3d76e38434d80c88ddfd353cb28~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

接下来开始手写`Field`：

```tsx
class Field extends React.Component<FieldProps, FieldState> implements FieldEntity {
  private mounted = false;

  constructor(props: FieldProps) {
    super(props);
    const { formStore } = props.fieldContext;
    // 更改formStore的initialValue
    formStore.initEntityValue(this);
  }

  // 在实例已经初始化且挂载完成后，把其自身注册到formStore里
  public componentDidMount() {
    this.mounted = true;
    this.props.fieldContext.formStore.registerField(this);
  }

  // 更新函数,其处理逻辑类似于Redux中的reducer
  public onStoreChange: FieldEntity['onStoreChange'] = (preStore, name, info) => {
    const { store } = info;
    const prevValue = preStore[this.props.name!];
    const curValue = store[this.props.name!];
    const nameMatch = namePathList && namePathList.includes(this.props.name!);
    name;

    switch (info.type) {
      default:
        if (nameMatch || (name !== undefined && prevValue !== curValue)) {
          this.reRender();
          return;
        }
        break;
    }
  };

  // 组件渲染更新函数，如果已经挂载了，则调用forceUpdate重新渲染
  public reRender() {
    if (!this.mounted) return;
    this.forceUpdate();
  }

  // 生成要通过React.cloneElement隐式混入到控件里的prop
  public getControlled = (childProps: ChildProps = {}) => {
    const {
      fieldContext,
      name,
      valuePropName = 'value',
      getValueFromEvent,
      trigger = 'onChange',
    } = this.props;
    const value = name ? this.props.fieldContext.formStore.getFieldValue(name) : undefined;
    const mergedGetValueProps = (val: any) => ({ [valuePropName]: val });

    const control = {
      ...childProps,
      ...mergedGetValueProps(value),
    };
    // 先取出用户原本定义在控件的trigger(默认为onChange)上的方法
    const originTriggerFunc: any = childProps[trigger];
    // 增强其方法
    control[trigger] = (...args: any[]) => {
      let newValue: any;
      if (getValueFromEvent) {
        newValue = getValueFromEvent(...args);
      } else {
        //如果没有定义getValueFromEvent这类从event取值方法，则调用defaultGetValueFromEvent方法取值
        // defaultGetValueFromEvent会从evnet.target[valuePropName]中取值
        newValue = defaultGetValueFromEvent(valuePropName, ...args);
      }
      // 调用updateValue更新formStore的store以及遍历调用fieldEntities里实例的onStateChange方法，
      // 也就是上面定义的onStateChange方法
      fieldContext.formStore.updateValue(name, newValue);
      if (originTriggerFunc) {
        originTriggerFunc(...args);
      }
    };

    return control;
  };

  public render() {
    const { children } = this.props;
    let returnChildNode: React.ReactNode;
    if (React.isValidElement(children)) {
      returnChildNode = React.cloneElement(children, this.getControlled(children.props));
    } else {
      returnChildNode = children;
    }
    return returnChildNode;
  }
}
```

写完三个核心部分后，我们用下面的例子测试一下：

```tsx
import Form from './Form';

const App = () => {
  return (
    <Form
      initialValues={{
        username: '123',
        is_admin: true,
      }}
    >
      <Form.Item label="用户名" name="username" initialValue="345">
        <input type="text" />
      </Form.Item>
      <Form.Item label="品牌" name="role" initialValue="saab">
        <select>
          <option value="volvo">Volvo</option>
          <option value="saab">Saab</option>
          <option value="mercedes">Mercedes</option>
          <option value="audi">Audi</option>
        </select>
      </Form.Item>
      <Form.Item label="是否是管理员" name="is_admin" valuePropName="checked">
        <input type="checkbox" />
      </Form.Item>
    </Form>
  );
};

export default App;
```

最后效果如下动图所示：

![my-antd-form-stage1.gif](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/86755095e4b541d1ad135433e01e8dad~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

效果总结：

1. 首先可以看到，两次修改表单控件数据顺利，且点击`FieldContext`里面查看`formStore`里的`store`，数据有对应的变化(要来回切换组件才看到，因为并非用`setState`或`useState`更新，所以 devtools 没有即时更新)。
2. `Form`和`Form.Item`的`initialValues`同时设置`username`的值，但刷新页面时显示`Form`中设置的值，这优先级逻辑与`Antd Form`的一致
3. 是否是管理员的勾选框所在的`Form.Item`中我们设置了`valuePropName`属性。而在运行中`formStore`里的`store`的`is_admin`值变化无误。说明该值生效。

至此我们的第一阶段算是完成了。

此阶段的代码我放在项目的分支[release_step1](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2FHitotsubashi%2Fmy-antd-form%2Ftree%2Frelease_step1)

## 第二阶段：实现用户交互功能

用户交互功能主要有三个：

1. 提交：点击提交按钮后，调用`Form.props.onFinish`
2. 重置：点击重置按钮后，重置表单值且调用`Form.props.onReset`
3. 监听变化：表单中控件的值发生变化时，调用`Form.props.onValuesChange`

接下来依次实现上面三个交互功能：

### 1. 提交

我们先定义一个`Callback`的接口用于定义回调函数：

```tsx
export interface Callbacks<Values = any> {
  onValuesChange?: (changedValues: any, values: Values) => void;
  onFinish?: (values: Values) => void;
}
```

这些回调函数将会存放到`FormStore`里，接下来我们在`FormStore`添加代码：

```tsx
// FormStore.ts
export class FormStore {
  private callbacks: Callbacks = {};

  public setCallbacks = (callbacks: Callbacks) => {
    this.callbacks = callbacks;
  };

  // 提交的时候调用formStore.submit方法触发onFinish
  public submit = () => {
    const { onFinish } = this.callbacks;
    if (onFinish) {
      onFinish(this.store);
    }
  };
}
```

然后在`Form`中添加代码：

```tsx
// Form.tsx
export interface FormProps<Values = any> extends BaseFormProps {
  initialValues?: Store;
  children?: React.ReactNode;
  // 在FormProps上添加onFinish类型的定义
  onFinish?: Callbacks<Values>['onFinish'];
}

const Form: React.FC<FormProps> = ({ onFinish }) => {
  formStore.current.setCallbacks({
    onFinish: (values: Store) => {
      if (onFinish) {
        onFinish(values);
      }
    },
  });

  return (
    <form
      onSubmit={(event: React.FormEvent<HTMLFormElement>) => {
        // 避免继续执行form的默认submit事件，继而触发跳转到action属性指定的页面，
        // 即时没有定义action，页面依旧会跳转，因此需要preventDefault来阻止
        event.preventDefault();
        // 阻止submit事件的传播。此举在于避免父级元素也是form的情况下，该事件传播到父级form元素上
        event.stopPropagation();
        // 调用formStore.submit，继而调用onFinish回调函数
        formStore.current.submit();
      }}
    >
      {wrapperNode}
    </form>
  );
};
```

这样子就可以完成提交功能了。

### 2. 重置

思路是，先在`FormStore`中写重置的方法（`resetFields`），然后在`Form`函数组件中返回的`jsx`里，在`form`元素注册`onReset`的回调函数上调用`FormStore.resetFields`即可。

```tsx
// FormStore.ts
export class FormStore {
  public resetFields = (nameList?: string[]) => {
    const prevStore = this.store;
    // 如果没有传形参，则直接重置整个store
    if (!nameList) {
      this.store = { ...this.initialValues };
      // resetWithFieldInitialValue是用于根据Form.Item上的initialValue调整store
      // 其逻辑是：遍历fieldEntities，如果实例上有定义initialValue和name，
      // 且this.initialValue[name]为undefined，则把实例上定义的initialValue赋值到store上
      this.resetWithFieldInitialValue();
      // 调用notifyObservers以遍历调用实例的更新方法
      this.notifyObservers(prevStore, undefined, { type: 'reset' });
      return;
    }
    // 如果nameList不为空，则只重置包含在nameList里的字段，逻辑与上面的相同
    nameList.forEach((name) => {
      this.store[name] = this.initialValues[name];
    });
    this.resetWithFieldInitialValue({ nameList });
    this.notifyObservers(prevStore, nameList, { type: 'reset' });
  };

  // 该方法的作用在上面说了，在真实源码中相比于下面的逻辑还多了一层判断，就是：
  // 如果存在多个name值相同的Form.Item实例定义了initialValue，则会不重置store对应的值，
  // 且跳出警告。
  private resetWithFieldInitialValue = (
    info: {
      entities?: FieldEntity[];
      nameList?: string[];
    } = {},
  ) => {
    const cache: Record<string, FieldEntity> = {};
    this.getFieldEntities().forEach((entity) => {
      const { name, initialValue } = entity.props;
      if (initialValue !== undefined) {
        cache[name!] = entity;
      }
    });

    let requiredFieldEntities: FieldEntity[];
    if (info.entities) {
      requiredFieldEntities = info.entities;
    } else if (info.nameList) {
      requiredFieldEntities = [];
      info.nameList.forEach((name) => {
        const record = cache[name];
        if (record) {
          requiredFieldEntities.push(record);
        }
      });
    } else {
      requiredFieldEntities = this.fieldEntities;
    }

    const resetWithFields = (entities: FieldEntity[]) => {
      entities.forEach((field) => {
        const { initialValue, name } = field.props;
        if (initialValue !== undefined && name !== undefined) {
          const formInitialValue = this.initialValues[name];
          if (formInitialValue === undefined) {
            this.store[name] = initialValue;
          }
        }
      });
    };

    resetWithFields(requiredFieldEntities);
  };
}
```

然后因为`notifyObservers`中调用的是`Field.onStateChange`，而`Field.onStateChange`采用与`Redux reducer`类似的`switch case`逻辑。因此我们要针对`reset`情况加处理方式，如下所示：

```tsx
// Field.tsx
export interface FieldState {
  resetCount: number;
}

class Field extends React.Component<FieldProps, FieldState> implements FieldEntity {
  public state = {
    // 定义resetCount用作React.Fragment的key值，
    // 在reset的时候更改resetCount值以达到重新渲染的目的
    resetCount: 0,
  };

  public onStoreChange: FieldEntity['onStoreChange'] = (preStore, namePathList, info) => {
    const { store } = info;
    const prevValue = preStore[this.props.name!];
    const curValue = store[this.props.name!];
    const nameMatch = namePathList && namePathList.includes(this.props.name!);

    switch (info.type) {
      // 新增处理reset情况的逻辑
      case 'reset':
        // 如果没有指定namePathList或者namePathList中包含该实例的props.name值，则重新渲染
        if (!namePathList || nameMatch) {
          this.refresh();
        }
        break;
      default:
        if (nameMatch || prevValue !== curValue) {
          this.reRender();
          return;
        }
        break;
    }
  };

  public refresh = () => {
    if (!this.mounted) return;
    // 通过递增更改resetCount
    this.setState(({ resetCount }) => ({
      resetCount: resetCount + 1,
    }));
  };

  public render() {
    const { resetCount } = this.state;
    const { children } = this.props;

    let returnChildNode: React.ReactNode;
    if (React.isValidElement(children)) {
      returnChildNode = React.cloneElement(children, this.getControlled(children.props));
    } else {
      returnChildNode = children;
    }
    // 该用React.Fragment包裹returnChildNode，且把resetCount赋值给key，
    // 以达到调用refresh更改resetCount时，让returnChildNode重新渲染的效果
    return <React.Fragment key={resetCount}>{returnChildNode}</React.Fragment>;
  }
}
```

最后在`Form`中`return`的`form`元素上注册`onReset`回调函数即可，如下所示：

```tsx
// Form.tsx
const Form: React.FC<FormProps> = ({
  initialValues,
  children,
  onValuesChange,
  onFinish,
  onReset,
}) => {
  return (
    <form
      onSubmit={(event: React.FormEvent<HTMLFormElement>) => {
        event.preventDefault();
        event.stopPropagation();
        formStore.current.submit();
      }}
      onReset={(event: React.FormEvent<HTMLFormElement>) => {
        // 调用preventDefault的目的在于阻止其默认的重置行为，
        // 因为这里的逻辑是根据Form和Form.Item的initialValue来重置
        // 而默认的重置行为重置的值可能与上述的不一致
        event.preventDefault();
        // 调用formStore.resetFields，也就是上面定义的方法
        formStore.current.resetFields();
        // 调用Form.props.onReset方法，该方法在Antd Form的教程上没暴露出来，
        // 但由于Form.props的类型--BaseFormProps继承于React.FormHTMLAttributes，
        // 而React.FormHTMLAttributes中定义了原生Form所有的属性，因此我们在Form.props可以定义encType、autoComplete这类原生的属性，当然也包括onReset
        onReset?.(event);
      }}
    >
      {wrapperNode}
    </form>
  );
};
```

以上代码即可完成重置功能。

重置功能的调用链有点复杂，因此我画了下面的流程图供大家参考：

![image.png](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/a74893328faa4f91ad621b1a42d70804~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

看到这里你可能有点疑惑： 既然在`field.onStateChange`执行之前已经重置了`store`的值。为什么要通过`field.refresh`，即递增`<React.Fragment key={resetCount}>`里的`key`值去更新？而不是直接通过`field.reRender`，即调用`forceUpdate`去更新。

首先要知道，`reRender`是用于更新控件，而`refresh`是用于生成新的控件后替换旧的控件。虽然注入的`value`值都是从已被重置的`store`里取出，都是一样的。但控件里可能不止`value`这种状态值，假设`Form.Item`里的控件是`Antd`里支持关键字搜索的`Select`，如下所示：

![antd-filter-select.gif](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/3fef47819ec54a6d9f61b5410cbe5f26~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

支持关键字搜索的`Select`内部会有一个`searchValue`的值来记录输入的关键字。如果用`reRender`作重置效果，只会更新`value`，而搜索的关键字不会消失。但如果用`refresh`作重置效果，`Select`会重新被生成，此时内部的`searchValue`为空，此时搜索的关键字就会消失。整个控件就会被重置成初始时的状态。

### 3. 监听变化

`onValuesChange`的实现思路与`onFinish`的类似：通过`FormStore.setCallbacks`把其存进`formStore`里，在`FormStore.updateValue`的逻辑里调用其即可。

根据上面的思路，我们先在`Form.tsx`上编写：

```tsx
// Form.tsx
const Form: React.FC<FormProps> = ({
  initialValues,
  children,
  onValuesChange,
  onFinish,
  onReset,
}) => {
  formStore.current.setCallbacks({
    // 把onValuesChange存进formStore.callbacks
    onValuesChange,
    onFinish: (values: Store) => {
      if (onFinish) {
        onFinish(values);
      }
    },
  });
};
复制代码
// FormStore.ts
export class FormStore {
  public updateValue = (name: string | undefined, value: any) => {
    if (name === undefined) return;
    const prevStore = this.store;
    this.store = { ...this.store, [name]: value };
    this.notifyObservers(prevStore, [name], {
      type: 'valueUpdate',
      source: 'internal',
    });

    const { onValuesChange } = this.callbacks;
    // 取出onValuesChange，如果不为空则执行且传入相应的参数
    if (onValuesChange) {
      const changedValues = { [name]: this.store[name] };
      onValuesChange(changedValues, this.getFieldsValue());
    }
  };
}
```

上面的代码既可完成监听变化的功能。

### 4. 效果展示

对于上面的三个特性，我们用下面的例子做测试，代码如下所示：

```tsx
import Form from './Form';

const App = () => {
  const onFinish = (values: any) => {
    console.log('finish', values);
  };

  const onValuesChange = (changedValues: any, values: any) => {
    console.log('onValuesChange', changedValues, values);
  };

  return (
    <Form
      initialValues={{
        username: '123',
        is_admin: true,
      }}
      onFinish={onFinish}
      onValuesChange={onValuesChange}
    >
      <Form.Item label="用户名" name="username" initialValue="345">
        <input type="text" />
      </Form.Item>
      <Form.Item label="品牌" name="role" initialValue="saab">
        <select>
          <option value="volvo">Volvo</option>
          <option value="saab">Saab</option>
          <option value="mercedes">Mercedes</option>
          <option value="audi">Audi</option>
        </select>
      </Form.Item>
      <Form.Item label="是否是管理员" name="is_admin" valuePropName="checked">
        <input type="checkbox" />
      </Form.Item>
      <Form.Item>
        <button type="submit">提交</button>
        <input type="reset" value="重置" />
      </Form.Item>
    </Form>
  );
};

export default App;
```

效果如下动图所示：

![stage2.gif](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/032ae2203b8441548e509014d272e95a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

从动图可总结出：

1. `onValuesChange`设置成功，且返回参数中有显示哪个形参发生变化。
2. `onFinish`设置成功。
3. 重置功能成功。且点击重置时，“用户名”的值置为“123”而不是“456”。因为`Form`设置的`initialValue`比`Form.Item`的优先级高。

第二阶段的所有特性至此完成，我把此阶段的所有代码放在分支[release_step2](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2FHitotsubashi%2Fmy-antd-form%2Ftree%2Frelease_step2%2F)上。

## 第三阶段：实现获取表单实例

获取**表单实例**其实就是通过`React.createRef`或者`React.useRef`创建`ref`对象。然后在`Form`实例上注入`ref`参数获取表单实例。当然在函数组件里也可以通过`Form.useForm`直接获取**表单实例**。

通常我们调用**表单实例**所做的操作例如`resetField`、`submit`其实就是对**数据状态**的读写以及对**表单字段组件**实例的更新，而我们之前在`Form`和`Form.Item`的代码里，做这两方面的操作都是通过`formStore`来进行的。那我们其实可以让**表单实例**里的方法指向`formStore`里的方法就行，这样子就可以保证。在`App.jsx`中**表单实例**进行操作时，实际上也是对`formStore.store`进行读写，这样子既可以简化设计结构，也可以保证**数据状态**的唯一性。

我们可以在`FormStore`类里新增`getForm`逻辑，如下所示：

```tsx
// FormStore.ts
export interface FormInstance {
  getFieldValue: typeof FormStore.prototype.getFieldValue;
  getFieldsValue: typeof FormStore.prototype.getFieldsValue;
  setFieldsValue: typeof FormStore.prototype.setFieldsValue;
  submit: typeof FormStore.prototype.submit;
  resetFields: typeof FormStore.prototype.resetFields;
  getInternalHooks: typeof FormStore.prototype.getInternalHooks;
}

export interface InternalHooks {
  updateValue: typeof FormStore.prototype.updateValue;
  initEntityValue: typeof FormStore.prototype.initEntityValue;
  registerField: typeof FormStore.prototype.registerField;
  setInitialValues: typeof FormStore.prototype.setInitialValues;
  setCallbacks: typeof FormStore.prototype.setCallbacks;
}

export class FormStore {
  // getForm中返回的对象就是**表单实例**
  // **表单实例**里的方法指向formStore自身的方法
  // 由于目前FormStore里所有的方法都是以箭头函数的形式编写，因此this都是指向作用域的this，也就是formStore自身
  public getForm = (): FormInstance => ({
    getFieldValue: this.getFieldValue,
    getFieldsValue: this.getFieldsValue,
    setFieldsValue: this.setFieldsValue,
    submit: this.submit,
    resetFields: this.resetFields,
    // getInternalHooks是用于暴露formStore更底层的方法
    getInternalHooks: this.getInternalHooks,
  });

  public getInternalHooks = (): InternalHooks => {
    return {
      updateValue: this.updateValue,
      initEntityValue: this.initEntityValue,
      registerField: this.registerField,
      setInitialValues: this.setInitialValues,
      setCallbacks: this.setCallbacks,
    };
  };
}
```

但这样子由于`Form.useForm`会存在一个问题，一直以来，我们都是在`Form`里创建`formStore`的。而我们调用`Form.useForm`时，却是下面这样子的：

```tsx
const App = () => {
  const [form] = Form.useForm();

  return <Form form={form}>{/* 代码省略... */}</Form>;
};
```

`Form.useForm`先于`Form`实例化之前执行。那就相当于`formStore`被创建之前就要调用`formStore.getForm()`，这可以做到吗？

我们再仔细看上面`App`逻辑，用`Form.useForm`获取**表单实例**，然后实例化`Form`时会把**表单实例**当作`form`参数从`props`注入进去。

根据上面的执行顺序，我们可以设计成这样子：把创建`formStore`的行为从`Form`挪到`useForm`方法里。`useForm`方法被调用时，如果`formStore`已被创建，则直接把之前创建的返回出去，等同于单例模式。因此，`Form.useForm`代码可以写成下面的样子：

```tsx
// FormStore.ts
export function useForm(form?: FormInstance): [FormInstance] {
  const formRef = React.useRef<FormInstance>();

  if (!formRef.current) {
    // form作为参数，若form不为空，则不会创建且会把form存入formRef里
    if (form) {
      formRef.current = form;
      // 若form为空，则创建formStore且把getForm()返回的对象存入formRef里
    } else {
      const formStore: FormStore = new FormStore();
      formRef.current = formStore.getForm();
    }
  }
  // 最后返回formRef.current
  return [formRef.current];
}
```

在`Form`组件内部可以这么写：

```tsx
// Form.tsx
// 注意此处的Form类型不再是React.FC<FormProps>，因为要考虑到ref注入的情况，所以类型改成下面这种
const Form: React.ForwardRefRenderFunction<FormInstance, FormProps> = (
  { form, initialValues, children, onValuesChange, onFinish, onReset },
  ref,
) => {
  // 不再用new FormStore()创建formStore，而是用useForm获取
  const [formInstance] = useForm(form);

  // 如果用户是通过ref获取表单实例，则通过useImperativeHandle把formInstance返回出去
  React.useImperativeHandle(ref, () => formInstance);

  const fieldContextValue = useMemo(
    // 这里以解构又组合的做法是为了防止用户在App中乱改formInstance(例如把formInstance.submit指向null)，从而影响Form和Form.Item内部调用
    () => ({
      ...formInstance,
    }),
    [formInstance],
  );

  const wrapperNode = (
    <FieldContext.Provider value={fieldContextValue}>{children}</FieldContext.Provider>
  );

  return (
    // ... 跟以前一样，无需再次展示
  );
};
```

这样子就可以实现`Form.useForm`以及`ref`获取**表单实例**的逻辑了。

注意`FieldContext`的`value`注入的不再是`{formStore: formStore.current}`，因此`Form.Item`里调用`formStore`的逻辑都要更改。不过都是替换都一样我就不再展示了，具体想了解的可以看阅读这里[/release_step3/src/Form/Field.tsx](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2FHitotsubashi%2Fmy-antd-form%2Fblob%2Frelease_step3%2Fsrc%2FForm%2FField.tsx)。

接下来我们用下面的`App`的代码来测试：

```tsx
// App.tsx
import Form from './Form';

const App = () => {
  const [form] = Form.useForm();

  const submit = () => {
    form.submit();
  };

  const reset = () => {
    form.resetFields();
  };

  const onFinish = (values: any) => {
    console.log('finish', values);
  };

  return (
    <Form
      form={form}
      initialValues={{
        username: '123',
        is_admin: true,
      }}
      onFinish={onFinish}
    >
      <Form.Item label="用户名" name="username" initialValue="345">
        <input type="text" />
      </Form.Item>
      <Form.Item label="品牌" name="role" initialValue="saab">
        <select>
          <option value="volvo">Volvo</option>
          <option value="saab">Saab</option>
          <option value="mercedes">Mercedes</option>
          <option value="audi">Audi</option>
        </select>
      </Form.Item>
      <Form.Item label="是否是管理员" name="is_admin" valuePropName="checked">
        <input type="checkbox" />
      </Form.Item>
      <Form.Item>
        {/* 部分浏览器会存在不加type则默认type为submit的情况 */}
        <button type="button" onClick={submit}>
          提交
        </button>
        <button type="button" onClick={reset}>
          重置
        </button>
      </Form.Item>
    </Form>
  );
};

export default App;
```

效果如下动图所示：

![stage3.gif](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/e20b4fdac2434ac9af93bec87999dfbc~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

可以看出：无论是重置还是提交，效果和阶段二中使用`button`触发重置和提交的效果一样。至此，阶段三的特性已经完成，我把项目代码放在分支[release_step3](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2FHitotsubashi%2Fmy-antd-form%2Ftree%2Frelease_step3%2F)上。

## 总结

至此，其实`Antd Form V4`中比较突出的特性及其涉及到的设计逻辑已经介绍完毕。看到这里你已经看完了 70%的精华了。当然还有一些比较常用的没有补充，例如：

1. `Form`和`Form.Item`的`props.children`为函数而并非`ReactNode`时的处理方式
2. `Form.Item`的`shouldUpdate`和`dependencies`的机制

这些特性我目前已经了解原理但还没下笔，之后有精力我会继续更新这篇文章补充这些特性。

# 学习原理后对日常开发有什么用？

假设有这么一个页面，表单`Form`里存在着一个日期范围选择器`DatePicker.RangePicker`，如下所示：

```jsx
import './styles.css';
import 'antd/dist/antd.css';
import { Form, Button } from 'antd';
import CustomDateRangePicker from './CustomDateRangePicker';

export default function App() {
  return (
    <Form onFinish={onFinish}>
      <Form.Item label="日期选择" name="date_range">
        <DatePicker.RangePicker />
      </Form.Item>
      <Form.Item>
        <Button htmlType="submit">提交</Button>
      </Form.Item>
    </Form>
  );
}
```

现在的要加一个需求是，要往`DatePicker.RangePicker`里加一个快捷选项的`button`，能一键选泽日期范围为最近三个月之内。那么，在开头第一阶段我们已知像`<DatePicker.RangePicker />`这类在`Form.Item`里的**控件**会被隐式处理成受控组件。因此，我们只需要在`button`被按下后调用`onChange`把最近三个月的时间范围传回去就行。

首先，基于`DatePicker.RangePicker`二次封装组件`CustomDateRangePicker`，如下所示：

```tsx
// CustomDateRangePicker.tsx
import { Button, DatePicker } from 'antd';
import type { RangePickerProps } from 'antd/lib/date-picker';
import React from 'react';
import moment from 'moment';

type Props = RangePickerProps;

const CustomDateRangePicker: React.FC<Props> = (props) => {
  // 先截取从props上截取onChange方法
  const { onChange } = props;

  // Button被点击后的回调函数
  const emitLastThreeMonthsValue = () => {
    // 先获取现在的时间
    const currentMonthMoment = moment();
    currentMonthMoment.set('date', 1);
    currentMonthMoment.set('hour', 0);
    currentMonthMoment.set('minute', 0);
    currentMonthMoment.set('second', 0);
    currentMonthMoment.set('millisecond', 0);
    // 生成代表开头的时间
    const firstMoment = moment(currentMonthMoment);
    firstMoment.subtract(2, 'months');
    // 生成代表结尾的时间
    const lastMoment = moment(currentMonthMoment);
    lastMoment.add(1, 'months');
    // 如果onChange存在则调用，注意传入参数的格式要根据RangePickerProps.onChange的格式
    onChange?.([firstMoment, lastMoment], [firstMoment.toString(), lastMoment.toString()]);
  };

  return (
    <DatePicker.RangePicker
      {...props}
      // 在renderExtraFooter上添加快捷按钮
      renderExtraFooter={() => (
        <Button size="small" type="link" onClick={emitLastThreeMonthsValue}>
          最近三个月
        </Button>
      )}
    />
  );
};

export default CustomDateRangePicker;
```

最后在`App`里把`<DatePicker.RangePicker />`替换为`<CustomDateRangePicker/>`即可。我们顺带加个`onFinish`方法来查看**数据状态**里的值，如下所示：

```js
import './styles.css';
import 'antd/dist/antd.css';
import { Form, Button } from 'antd';
import CustomDateRangePicker from './CustomDateRangePicker';

export default function App() {
  const onFinish = (values) => {
    if (!values.date_range) return;
    const [first, last] = values.date_range;
    console.log(first?.format('YYYY-M-D'), last?.format('YYYY-M-D'));
  };

  return (
    <Form onFinish={onFinish}>
      <Form.Item label="日期选择" name="date_range">
        <CustomDateRangePicker />
      </Form.Item>
      <Form.Item>
        <Button htmlType="submit">提交</Button>
      </Form.Item>
    </Form>
  );
}
```

最后效果如下动图所示：

![my-antd-form-use.gif](assets/%E4%B8%80%E6%AC%A1%E6%89%8B%E5%86%99Antd%20Form%E7%9A%84%E7%BB%8F%E5%8E%86%EF%BC%8C%E8%AE%A9%E6%88%91%E5%8F%97%E7%9B%8A%E5%8C%AA%E6%B5%85.assets/032082a53521476389635fddc194d4ab~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp)

想看详细代码可看示例链接[code sandbox](https://link.juejin.cn?target=https%3A%2F%2Fcodesandbox.io%2Fs%2Fcurrying-fog-qk1pg%3Ffile%3D%2Fsrc%2FApp.js)

# 后记

这篇文章之后会更新不断更新以补充细节。


作者：村上小树
链接：https://juejin.cn/post/7038099720400535582
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
