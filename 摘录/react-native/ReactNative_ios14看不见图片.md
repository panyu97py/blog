# ReactNative ios14看不见图片

## Xcode升级12后，ReactNative ios14看不见图片(静态图片和网络图片)

### 解决方法

#### 1、第一种：修改node_modules中react-native/Libraries/Image/RCTUIImageViewAnimates.m文件

```
if(_currentFrame){                   //275行
    layer.contentsScale = self.animatedImageScale;
    layer.contents = (__bridge id)_currentFrame.CGImage;
}else{    //加上这个  不然ios14以上的系统看不见图片
    [super displayLayer:layer];
}
```

#### 2、第二种：升级reactNative版本,大于或者等于0.63版本