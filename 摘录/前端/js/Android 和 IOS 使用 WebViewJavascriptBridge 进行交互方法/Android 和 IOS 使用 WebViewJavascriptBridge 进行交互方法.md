# Android 和 IOS 使用 WebViewJavascriptBridge 进行交互方法



## 本文采用第三方工具 WebViewJavascriptBridge

#### 注册监听事件

这段代码是固定的，必须要放到js中



```js
/*这段代码是固定的，必须要放到js中*/
function setupWebViewJavascriptBridge(callback) {

    //Android使用
    if (window.WebViewJavascriptBridge) {
        callback(WebViewJavascriptBridge)
    } else {
        document.addEventListener(
            'WebViewJavascriptBridgeReady'
            , function () {
                callback(WebViewJavascriptBridge)
            },
            false
        );
    }


    //iOS使用
    if (window.WebViewJavascriptBridge) { return callback(WebViewJavascriptBridge); }
    if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
    window.WVJBCallbacks = [callback];
    var WVJBIframe = document.createElement('iframe');
    WVJBIframe.style.display = 'none';
    WVJBIframe.src = 'wvjbscheme://__BRIDGE_LOADED__';
    document.documentElement.appendChild(WVJBIframe);
    setTimeout(function () { document.documentElement.removeChild(WVJBIframe) }, 0)
}
```

### 原生调用js

在 setupWebViewJavascriptBridge 中注册原生调用的js



```jsx
//在改function 中添加原生调起js方法
setupWebViewJavascriptBridge(function(bridge) {
    bridges = bridge;

    //注册原生调起方法
    //参数1： buttonjs 注册flag 供原生使用，要和原生统一
    //参数2： data  是原生传给js 的数据
    //参数3： responseCallback 是js 的回调，可以通过该方法给原生传数据
    bridge.registerHandler("buttonjs",function(data,responseCallback){

        document.getElementById("show").innerHTML = "buuton js" + data;
        responseCallback("button js callback");
    });
})
```

### js 调用原生方法

注：该方法应放在setupWebViewJavascriptBridge 中进行绑定



```jsx
document.getElementById('enter3').onclick = function (e) {
var data = "good hello"
//参数1： pay 注册flag 供原生使用，要和原生统一
//参数2： 是调起原生时向原生传递的参数
//参数3： 原生调用回调返回的数据
bridge.callHandler('getBlogNameFromObjC',data,function(resp){
        document.getElementById("show").innerHTML = "payInterface" + resp;
    }
 );
```

### 完整代码



```xml
<html>
<head>
    <meta content="text/html; charset=utf-8" http-equiv="content-type">
    <title>
        js调用java
    </title>
</head>

<body>
<p>
    <div id="show"></div>
</p>


<p><input type="button" id="enter3" value="payInterface" onclick="payInterface();"/></p>

</body>
<script>

        function setupWebViewJavascriptBridge(callback) {
            if (window.WebViewJavascriptBridge) {
                callback(WebViewJavascriptBridge)
            } else {
                document.addEventListener(
                    'WebViewJavascriptBridgeReady'
                    , function() {
                        callback(WebViewJavascriptBridge)
                    },
                    false
                );
            }

            if (window.WVJBCallbacks) { return window.WVJBCallbacks.push(callback); }
            window.WVJBCallbacks = [callback];
            var WVJBIframe = document.createElement('iframe');
            WVJBIframe.style.display = 'none';
            WVJBIframe.src = 'wvjbscheme://__BRIDGE_LOADED__';
            document.documentElement.appendChild(WVJBIframe);
            setTimeout(function() { document.documentElement.removeChild(WVJBIframe) }, 0)
        }

        //在改function 中添加原生调起js方法
        setupWebViewJavascriptBridge(function(bridge) {

            //注册原生调起方法
            //参数1： buttonjs 注册flag 供原生使用，要和原生统一
            //参数2： data  是原生传给js 的数据
            //参数3： responseCallback 是js 的回调，可以通过该方法给原生传数据
            bridge.registerHandler("getUserInfos",function(data,responseCallback){

                document.getElementById("show").innerHTML = "buuton js" + data;
                responseCallback("button js callback");
            });


            document.getElementById('enter3').onclick = function (e) {
            var data = "hello"
            //参数1： pay 注册flag 供原生使用，要和原生统一
            //参数2： 是调起原生时向原生传递的参数
            //参数3： 原生调用回调返回的数据
            bridge.callHandler('getBlogNameFromObjC',data,function(resp){
                    document.getElementById("show").innerHTML = "payInterface" + resp;
                }
             );
        }
        })
</script>

</html>
```

# iOS方法使用



```objectivec
#import "JsBridgeViewController.h"
#import "WKWebViewJavascriptBridge.h"
#import "WebViewJavascriptBridge.h"

@interface JsBridgeViewController ()<WKNavigationDelegate>
@property(nonatomic,strong) WKWebViewJavascriptBridge* bridge;
@property (nonatomic, strong)WKWebView * wkWebView;

@end

@implementation JsBridgeViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    [self.view addSubview:self.wkWebView];

    [WKWebViewJavascriptBridge enableLogging];
    self.bridge = [WKWebViewJavascriptBridge bridgeForWebView:self.wkWebView];
    [self.bridge setWebViewDelegate:self];
    //OC端的注册
//注册jscalloc方法，js调用oc,data为oc回调给js的参数
    [self.bridge registerHandler:@"getBlogNameFromObjC" handler:^(id data, WVJBResponseCallback responseCallback) {
        NSLog(@"js call getBlogNameFromObjC, data from js is %@", data);
        if (responseCallback) {
            // 反馈给JS
            responseCallback(@{@"blogName": @"测试blog"});
        }
    }];
------------------------------------------------------------------
//OC端的调用
//调用js中注册的showapiinner方法，不传入数据，没有回调
//[_bridge callHandler:@"showapiinner"];
//调用js中注册的showapiinner方法，传入数据str，没有回调
//[_bridge callHandler:@"showapiinner" data:str];
//调用js中注册的showapiinner方法，传入数据str，且有回调（可以获取到回调参数responData）
// [_bridge callHandler:@"showapiinner" data:str responseCallback:^(id //responseData) {
 // NSLog(@"%@",responseData);
   // }];
    [self renderButtons:self.wkWebView];
    [self loadExamplePage:self.wkWebView];
}
//native调js  （native按钮）
- (void)callHandler:(id)sender {
    
    [self.bridge callHandler:@"getUserInfos" data:@{@"name": @"测试"} responseCallback:^(id responseData) {
        NSLog(@"from js: %@", responseData);
    }];
}

- (void)renderButtons:(WKWebView*)webView {
    UIFont* font = [UIFont fontWithName:@"HelveticaNeue" size:12.0];
    
    UIButton *callbackButton = [UIButton buttonWithType:UIButtonTypeRoundedRect];
    [callbackButton setTitle:@"Call handler" forState:UIControlStateNormal];
    [callbackButton addTarget:self action:@selector(callHandler:) forControlEvents:UIControlEventTouchUpInside];
    [self.view insertSubview:callbackButton aboveSubview:webView];
    callbackButton.frame = CGRectMake(10, 400, 100, 35);
    callbackButton.titleLabel.font = font;
    
    UIButton* reloadButton = [UIButton buttonWithType:UIButtonTypeRoundedRect];
    [reloadButton setTitle:@"Reload webview" forState:UIControlStateNormal];
    [reloadButton addTarget:webView action:@selector(reload) forControlEvents:UIControlEventTouchUpInside];
    [self.view insertSubview:reloadButton aboveSubview:webView];
    reloadButton.frame = CGRectMake(110, 400, 100, 35);
    reloadButton.titleLabel.font = font;
}

- (void)loadExamplePage:(WKWebView*)webView {
    NSString* htmlPath = [[NSBundle mainBundle] pathForResource:@"demo0" ofType:@"html"];
    NSString* appHtml = [NSString stringWithContentsOfFile:htmlPath encoding:NSUTF8StringEncoding error:nil];
    NSURL *baseURL = [NSURL fileURLWithPath:htmlPath];
    [webView loadHTMLString:appHtml baseURL:baseURL];
}

- (WKWebView *)wkWebView
{
    if (!_wkWebView) {
        _wkWebView = [[WKWebView alloc] initWithFrame:self.view.bounds];
        [self.view addSubview:_wkWebView];
        
        _wkWebView.navigationDelegate = self;
        _wkWebView.scrollView.bounces = 0;
        _wkWebView.scrollView.showsVerticalScrollIndicator = 0;
        _wkWebView.scrollView.showsHorizontalScrollIndicator = 0;
    }
    return _wkWebView;
}
```

# Android方法使用



```java
package com.cfox.jsbridge;

import android.app.Activity;
import android.content.Intent;
import android.net.Uri;
import android.os.Bundle;
import android.support.design.widget.FloatingActionButton;
import android.support.design.widget.Snackbar;
import android.support.v7.app.AppCompatActivity;
import android.support.v7.widget.Toolbar;
import android.util.Log;
import android.view.View;
import android.webkit.ValueCallback;
import android.webkit.WebChromeClient;
import android.webkit.WebView;
import android.widget.Toast;

import com.cfox.jsbridge.modle.User;
import com.github.lzyzsd.jsbridge.BridgeHandler;
import com.github.lzyzsd.jsbridge.BridgeWebView;
import com.github.lzyzsd.jsbridge.BridgeWebViewClient;
import com.github.lzyzsd.jsbridge.CallBackFunction;
import com.github.lzyzsd.jsbridge.DefaultHandler;
import com.google.gson.Gson;

public class MainActivity extends AppCompatActivity {

    private BridgeWebView mWebView;

    ValueCallback<Uri> mUploadMessage;

    int RESULT_CODE = 0;

    private static final String TAG = "MainActivity";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);


        mWebView = (BridgeWebView) findViewById(R.id.webView);
        initWebView();
    }

    private void initWebView() {

        mWebView.loadUrl("file:///android_asset/demo.html");
//js调native
        mWebView.registerHandler("getBlogNameFromObjC", new BridgeHandler() {
            @Override
            public void handler(String data, CallBackFunction function) {
                Toast.makeText(MainActivity.this, "pay--->，"+ data, Toast.LENGTH_SHORT).show();
                function.onCallBack("测试blog");
            }
        });
    }
//native调js （native按钮）
    public void sendNative(View view) {
        mWebView.callHandler("getUserInfos", "hello good", new CallBackFunction() {
            @Override
            public void onCallBack(String data) {
                Toast.makeText(MainActivity.this, "buttonjs--->，"+ data, Toast.LENGTH_SHORT).show();
            }
        });

    }
}
```



作者：Kean_Qi
链接：https://www.jianshu.com/p/e37ccf32cb5b
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。