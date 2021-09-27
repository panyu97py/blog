“React 和 Vue 哪个更好？” 论坛上经常看到这样的问题，然后评论区就直接开战了。也有朋友转行做前端，问我该学React还是Vue。几年前，可能确实有必要考虑下到底该选择哪一个，毕竟前端圈子这么乱，谁又知道Vue能走多远？React会不会不维护了呢？可现在两者生态都很不错，Vue确实好用，React学习成本也没有传闻中那么高，template很好用，jsx也更灵活。可以两者都去玩玩，根据个人喜好和项目需要来选择用哪个。而如果能够结合两者的优点，那岂不是很有趣？

 我刚转前端的时候，用的是vue版本好像还是1.0，那时候的感觉就是数据绑定比jquery操作dom省事儿太多了。后来又接触了React，之后的项目大部分用React写的，现在偶尔也用vue，总体感觉就是vue单文件组件结构比较清晰，模板指令也很好用，而jsx更加灵活，之前有在react状态管理部分做一些尝试，可以像普通function一样去更新状态，也一直想在jsx中加上类似vue里面的模板指令，直到前几天比较闲，总算实现了 “条件渲染”和 “列表渲染”，结果还算不错，过程也挺有趣。

 先来看看结果吧，以前要根据不同状态来控制模块是否显示，我们大概要写这样的代码：

```
render(){
    const visible = true
    return(
        <div>
            {
                visible ? <div>content<div>
                        : ''
            }
        </div>
    )
}
```

 现在可以这么玩：

```
render(){
    const visible = true
    return(
        <div>
            <div r-if = {visible}>content</div>
        </div>
    )
}
```

 另一种常见的场景就是根据一个数组来渲染出一个列表，一般是这么写：

```
render(){
    const list = [1, 2, 3, 4, 5]
    return(
        <div>
            {
                list.map((item,index)=>(
                	<div key={index}>{item}</div>
                ))
            }
        </div>
    )
}
```

 现在可以更简洁：

```
render(){
    const list = [1, 2, 3, 4, 5]
    return(
        <div>
            <div r-for = {item in list}>{item}</div>
        </div>
    )
}
```

 以上代码会自动设置key，值为当前元素的索引。如果你想要自定义key，也可以加上，改成

```
<div r-for = {(item,index) in list} key = {index+1}>{item}</div>
```

 结果还算不错吧，代码更简短，语义也比较明确，体验也不必vue里面的模板指令差，个人感觉在“”里面写js有点奇怪。而在{}里面写就很自然，就是普通的js代码块嘛。

 至于实现方法嘛，其实很简单，总共才几十行代码，就是写了一个babel插件。

 我们写的jsx也是通过babel转译成普通js代码的，然后才能在浏览器中运行，而babel编译主要分为三个阶段：解析、转换、生成目标代码。解析部分就是将源代码解析成抽象语法树ast，转换过程是对ast做一些处理，而生成目标代码部分就是将ast再转换成js代码。babel-plugin就是在转换部分做一些工作。

 比如对于以下jsx：

```
<div r-if = { visible }>{content}</div>
```

 转换成的ast结构大概是：

```
{
   	type: 'CallExpression',
    callee: {},
    arguments: {
        properties: []
    }
}
```

 目标代码为：

```
React.createElement(
    'div',
    {'r-if': visible},
    content
)
```

 React.createElement()方法调用对应ast中的CallExpression, React和createElement在callee中可以找到，可以以此来找出createElement(), r-if 等属性以及第三个参数content在arguments的properties数组中能找到。有了这些信息，我们就可以遍历ast，找到那些callee为React.createElement的CallExpression, 然后判断arguments中如果出现了r-if, 就对ast做以下修改：首先移除r-if属性，避免死循环；然后在CallExpression对应的节点外面再套一层ifStatement,  如此一来，转换后的ast生成的目标代码大致如下：

```
if(visible){
    React.createElement(
        'div',
        {'r-if': visible},
        content
    )
}
```

 至此，我们的目标就已经达到了，至于r-for列表渲染，原理类似，先找出有r-for属性的CallExpression, 然后构造一个map方法对应的CallExpression, 当前CallExpression作为参数传给map方法即可。需要注意的是，在同一次遍历中解析出 r-if 和 r-for 的话，需要把map放在外层，ifStatement放在里面，如果觉得这样做结构比较混乱，可以拆分成不同的插件。

 最后再说一下babel-plugin的写法，其实也就是一个方法:

```
module.exports = function ({ types: t }) {
  return {
    visitor: {
      CallExpression(path) {
          // 在这里通过修改path来修改ast。
      },
      Identifier(path) {
            
      }
    }
  }
}
```

 types类型为babel-types, 提供了一些类似loadash的操作方法，比如做一些判断、构造节点。visitor里面写对应类型节点的遍历方法, 比如遍历标识符类型的就写在Identifier中，方法调用就写在CallExpression中。

 本文中提到的r-if 和 r-for 已经写成了一个插件，可以在[github仓库](https://github.com/panyu97py/babel-plugin-react-directive)中找到,同时也发布到了npm仓库，可以直接安装：

```
yarn add --dev babel-plugin-react-directive
```

 然后在.babelrc中配置即可：

```
{
    "plugins": [
        "react-directive"
    ]
}
```

 我想要的目的已经达到了，但这并未结束，才刚刚开始，还可以实现其他的一些指令，比如r-if 是模块渲染或者不渲染的，我们经常也会遇到这种需求：只是单纯的控制元素可见或者不可见，但元素还是占用空间的，也就是控制visibility, 这也可以写成一个指令。而babel能做的远远不止如此，无聊的时候可以好好玩一玩。