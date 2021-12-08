# react native webview 的使用

#### 安装

```
yarn add react-native-webview 
```

#### 渲染远程 HTML 内容

```
import React, { Component } from 'react'; 
import { View } from 'react-native'; 
import { WebView } from 'react-native-webview'; 
export default class App extends Component { 
    render(){
        return (
            <View>
                <WebView source={{ uri: 'https://www.baidu.com' }} /> 
            </View>
        )
    }
} 
```

#### 渲染 HTML 字符串

```
<WebView 
originWhitelist={['*']} 
source={{ html: `<h1>这里是一个标题</h1>` }} 
/> 
```

#### 根据内容自动计算高度

往往我们无法提前预知 `HTML` 内容的高度，所以我们无法给其设置一个固定的高度，而是需要根据内容来设置其高度。

为了达到这个目的，我们需要用到 `injectedJavaScript` 和 `onMessage`

##### injectedJavaScript

这个属性的作用是设置一个在网页加载之前执行的 js 代码。

##### onMessage

在 `webview` 内部的网页中调用 `window.postMessage` 方法时可以触发此属性对应的函数，从而实现网页和 `RN` 之间的数据交换。 设置此属性的同时会在 `webview` 中注入一个`postMessage` 的全局函数并覆盖可能已经存在的同名实现。

网页端的 `window.postMessage` 只发送一个参数 `data`，此参数封装在 `RN` 端的 `event` 对象中，即 `event.nativeEvent.data`。`data` 只能是一个字符串。

```
import React, { Component } from 'react';
import { View, ScrollView, SafeAreaView } from 'react-native';
import { WebView } from 'react-native-webview';

export default class App extends Component {
  constructor(props) {
    super(props);

    this.state = { webViewHeight: 0 };
  }

  {/* 根据内容计算 WebView 高度 */}
  onWebViewMessage = (event) => {
    this.setState({ webViewHeight: Number(event.nativeEvent.data) });
  }

  render() {
    const html = `<p>这里是一个标题</p>`
    const injectedJavaScript = `window.ReactNativeWebView.postMessage(document.documentElement.scrollHeight)`
    return (
      <View style={{ flex: 1 }}>
        <View style={{ height: 100, backgroundColor: 'yellow' }}></View>
        <View style={{ height: this.state.webViewHeight }}>
          <WebView
            originWhitelist={['*']}
            source={{ html }}
            injectedJavaScript={{ injectedJavaScript }}
            onMessage={this.onWebViewMessage}
          />
        </View>
        <View style={{ height: 100, backgroundColor: 'green' }}></View>
      </View>
    )
  }
}
```

#### 采用响应式布局

%23%23%23%23%20%E5%AE%89%E8%A3%85%20%0A%60%60%60bash%20%0Ayarn%20add%20react-native-webview%20%0A%60%60%60%20%0A%23%23%23%23%20%E6%B8%B2%E6%9F%93%E8%BF%9C%E7%A8%8B%20HTML%20%E5%86%85%E5%AE%B9%20%0A%60%60%60%0Aimport%20React%2C%20%7B%20Component%20%7D%20from%20'react'%3B%20%0Aimport%20%7B%20View%20%7D%20from%20'react-native'%3B%20%0Aimport%20%7B%20WebView%20%7D%20from%20'react-native-webview'%3B%20%0Aexport%20default%20class%20App%20extends%20Component%20%7B%20%0A%20%20%20%20render()%7B%0A%20%20%20%20%20%20%20%20return%20(%0A%20%20%20%20%20%20%20%20%20%20%20%20%3CView%3E%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%3CWebView%20source%3D%7B%7B%20uri%3A%20'https%3A%2F%2Fwww.baidu.com'%20%7D%7D%20%2F%3E%20%0A%20%20%20%20%20%20%20%20%20%20%20%20%3C%2FView%3E%0A%20%20%20%20%20%20%20%20)%0A%20%20%20%20%7D%0A%7D%20%0A%60%60%60%0A%23%23%23%23%20%E6%B8%B2%E6%9F%93%20HTML%20%E5%AD%97%E7%AC%A6%E4%B8%B2%0A%60%60%60%0A%3CWebView%20%0AoriginWhitelist%3D%7B%5B'*'%5D%7D%20%0Asource%3D%7B%7B%20html%3A%20%60%3Ch1%3E%E8%BF%99%E9%87%8C%E6%98%AF%E4%B8%80%E4%B8%AA%E6%A0%87%E9%A2%98%3C%2Fh1%3E%60%20%7D%7D%20%0A%2F%3E%20%0A%60%60%60%20%0A%23%23%23%23%20%E6%A0%B9%E6%8D%AE%E5%86%85%E5%AE%B9%E8%87%AA%E5%8A%A8%E8%AE%A1%E7%AE%97%E9%AB%98%E5%BA%A6%20%0A%E5%BE%80%E5%BE%80%E6%88%91%E4%BB%AC%E6%97%A0%E6%B3%95%E6%8F%90%E5%89%8D%E9%A2%84%E7%9F%A5%20%60HTML%60%20%E5%86%85%E5%AE%B9%E7%9A%84%E9%AB%98%E5%BA%A6%EF%BC%8C%E6%89%80%E4%BB%A5%E6%88%91%E4%BB%AC%E6%97%A0%E6%B3%95%E7%BB%99%E5%85%B6%E8%AE%BE%E7%BD%AE%E4%B8%80%E4%B8%AA%E5%9B%BA%E5%AE%9A%E7%9A%84%E9%AB%98%E5%BA%A6%EF%BC%8C%E8%80%8C%E6%98%AF%E9%9C%80%E8%A6%81%E6%A0%B9%E6%8D%AE%E5%86%85%E5%AE%B9%E6%9D%A5%E8%AE%BE%E7%BD%AE%E5%85%B6%E9%AB%98%E5%BA%A6%E3%80%82%0A%0A%E4%B8%BA%E4%BA%86%E8%BE%BE%E5%88%B0%E8%BF%99%E4%B8%AA%E7%9B%AE%E7%9A%84%EF%BC%8C%E6%88%91%E4%BB%AC%E9%9C%80%E8%A6%81%E7%94%A8%E5%88%B0%20%60injectedJavaScript%60%20%E5%92%8C%20%60onMessage%60%0A%0A%23%23%23%23%23%20injectedJavaScript%0A%E8%BF%99%E4%B8%AA%E5%B1%9E%E6%80%A7%E7%9A%84%E4%BD%9C%E7%94%A8%E6%98%AF%E8%AE%BE%E7%BD%AE%E4%B8%80%E4%B8%AA%E5%9C%A8%E7%BD%91%E9%A1%B5%E5%8A%A0%E8%BD%BD%E4%B9%8B%E5%89%8D%E6%89%A7%E8%A1%8C%E7%9A%84%20js%20%E4%BB%A3%E7%A0%81%E3%80%82%0A%0A%23%23%23%23%23%20onMessage%0A%0A%E5%9C%A8%20%60webview%60%20%E5%86%85%E9%83%A8%E7%9A%84%E7%BD%91%E9%A1%B5%E4%B8%AD%E8%B0%83%E7%94%A8%20%60window.postMessage%60%20%E6%96%B9%E6%B3%95%E6%97%B6%E5%8F%AF%E4%BB%A5%E8%A7%A6%E5%8F%91%E6%AD%A4%E5%B1%9E%E6%80%A7%E5%AF%B9%E5%BA%94%E7%9A%84%E5%87%BD%E6%95%B0%EF%BC%8C%E4%BB%8E%E8%80%8C%E5%AE%9E%E7%8E%B0%E7%BD%91%E9%A1%B5%E5%92%8C%20%60RN%60%20%E4%B9%8B%E9%97%B4%E7%9A%84%E6%95%B0%E6%8D%AE%E4%BA%A4%E6%8D%A2%E3%80%82%20%E8%AE%BE%E7%BD%AE%E6%AD%A4%E5%B1%9E%E6%80%A7%E7%9A%84%E5%90%8C%E6%97%B6%E4%BC%9A%E5%9C%A8%20%60webview%60%20%E4%B8%AD%E6%B3%A8%E5%85%A5%E4%B8%80%E4%B8%AA%60%20postMessage%60%20%E7%9A%84%E5%85%A8%E5%B1%80%E5%87%BD%E6%95%B0%E5%B9%B6%E8%A6%86%E7%9B%96%E5%8F%AF%E8%83%BD%E5%B7%B2%E7%BB%8F%E5%AD%98%E5%9C%A8%E7%9A%84%E5%90%8C%E5%90%8D%E5%AE%9E%E7%8E%B0%E3%80%82%0A%0A%E7%BD%91%E9%A1%B5%E7%AB%AF%E7%9A%84%20%60window.postMessage%60%20%E5%8F%AA%E5%8F%91%E9%80%81%E4%B8%80%E4%B8%AA%E5%8F%82%E6%95%B0%20%60data%60%EF%BC%8C%E6%AD%A4%E5%8F%82%E6%95%B0%E5%B0%81%E8%A3%85%E5%9C%A8%20%60RN%60%20%E7%AB%AF%E7%9A%84%20%60event%60%20%E5%AF%B9%E8%B1%A1%E4%B8%AD%EF%BC%8C%E5%8D%B3%20%60event.nativeEvent.data%60%E3%80%82%60data%60%20%E5%8F%AA%E8%83%BD%E6%98%AF%E4%B8%80%E4%B8%AA%E5%AD%97%E7%AC%A6%E4%B8%B2%E3%80%82%0A%0A%60%60%60%0Aimport%20React%2C%20%7B%20Component%20%7D%20from%20'react'%3B%0Aimport%20%7B%20View%2C%20ScrollView%2C%20SafeAreaView%20%7D%20from%20'react-native'%3B%0Aimport%20%7B%20WebView%20%7D%20from%20'react-native-webview'%3B%0A%0Aexport%20default%20class%20App%20extends%20Component%20%7B%0A%20%20constructor(props)%20%7B%0A%20%20%20%20super(props)%3B%0A%0A%20%20%20%20this.state%20%3D%20%7B%20webViewHeight%3A%200%20%7D%3B%0A%20%20%7D%0A%0A%20%20%7B%2F*%20%E6%A0%B9%E6%8D%AE%E5%86%85%E5%AE%B9%E8%AE%A1%E7%AE%97%20WebView%20%E9%AB%98%E5%BA%A6%20*%2F%7D%0A%20%20onWebViewMessage%20%3D%20(event)%20%3D%3E%20%7B%0A%20%20%20%20this.setState(%7B%20webViewHeight%3A%20Number(event.nativeEvent.data)%20%7D)%3B%0A%20%20%7D%0A%0A%20%20render()%20%7B%0A%20%20%20%20const%20html%20%3D%20%60%3Cp%3E%E8%BF%99%E9%87%8C%E6%98%AF%E4%B8%80%E4%B8%AA%E6%A0%87%E9%A2%98%3C%2Fp%3E%60%0A%20%20%20%20const%20injectedJavaScript%20%3D%20%60window.ReactNativeWebView.postMessage(document.documentElement.scrollHeight)%60%0A%20%20%20%20return%20(%0A%20%20%20%20%20%20%3CView%20style%3D%7B%7B%20flex%3A%201%20%7D%7D%3E%0A%20%20%20%20%20%20%20%20%3CView%20style%3D%7B%7B%20height%3A%20100%2C%20backgroundColor%3A%20'yellow'%20%7D%7D%3E%3C%2FView%3E%0A%20%20%20%20%20%20%20%20%3CView%20style%3D%7B%7B%20height%3A%20this.state.webViewHeight%20%7D%7D%3E%0A%20%20%20%20%20%20%20%20%20%20%3CWebView%0A%20%20%20%20%20%20%20%20%20%20%20%20originWhitelist%3D%7B%5B'*'%5D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20source%3D%7B%7B%20html%20%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20injectedJavaScript%3D%7B%7B%20injectedJavaScript%20%7D%7D%0A%20%20%20%20%20%20%20%20%20%20%20%20onMessage%3D%7Bthis.onWebViewMessage%7D%0A%20%20%20%20%20%20%20%20%20%20%2F%3E%0A%20%20%20%20%20%20%20%20%3C%2FView%3E%0A%20%20%20%20%20%20%20%20%3CView%20style%3D%7B%7B%20height%3A%20100%2C%20backgroundColor%3A%20'green'%20%7D%7D%3E%3C%2FView%3E%0A%20%20%20%20%20%20%3C%2FView%3E%0A%20%20%20%20)%0A%20%20%7D%0A%7D%0A%60%60%60%20%0A%23%23%23%23%20%E9%87%87%E7%94%A8%E5%93%8D%E5%BA%94%E5%BC%8F%E5%B8%83%E5%B1%80%0A%60%60%60%0A%3CWebView%0A%20%20originWhitelist%3D%7B%5B'*'%5D%7D%0A%20%20source%3D%7B%7B%20html%3A%20%60%0A%20%20%3Chtml%3E%0A%20%20%20%20%3Chead%3E%0A%20%20%20%20%20%20%3Cmeta%20name%3D%22viewport%22%20content%3D%22width%3Ddevice-width%2C%20initial-scale%3D1.0%22%3E%0A%20%20%20%20%3C%2Fhead%3E%0A%0A%20%20%20%20%3Cbody%3E%0A%20%20%20%20%20%20%3Cp%3E%E8%BF%99%E9%87%8C%E6%98%AF%E4%B8%80%E4%B8%AA%E6%A0%87%E9%A2%98%3C%2Fp%3E%0A%20%20%20%20%3C%2Fbody%3E%0A%20%20%3C%2Fhtml%3E%0A%20%20%60%20%7D%7D%0A%20%20injectedJavaScript%3D'window.ReactNativeWebView.postMessage(document.documentElement.scrollHeight)'%0A%20%20onMessage%3D%7Bthis.onWebViewMessage%7D%0A%2F%3E%0A%60%60%60
