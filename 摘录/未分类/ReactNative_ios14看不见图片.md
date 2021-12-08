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

%23%23%20Xcode%E5%8D%87%E7%BA%A712%E5%90%8E%EF%BC%8CReactNative%20ios14%E7%9C%8B%E4%B8%8D%E8%A7%81%E5%9B%BE%E7%89%87(%E9%9D%99%E6%80%81%E5%9B%BE%E7%89%87%E5%92%8C%E7%BD%91%E7%BB%9C%E5%9B%BE%E7%89%87)%0A%0A%23%23%23%20%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95%0A%23%23%23%23%201%E3%80%81%E7%AC%AC%E4%B8%80%E7%A7%8D%EF%BC%9A%E4%BF%AE%E6%94%B9node_modules%E4%B8%ADreact-native%2FLibraries%2FImage%2FRCTUIImageViewAnimates.m%E6%96%87%E4%BB%B6%0A%60%60%60%0Aif(_currentFrame)%7B%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%2F%2F275%E8%A1%8C%0A%20%20%20%20layer.contentsScale%20%3D%20self.animatedImageScale%3B%0A%20%20%20%20layer.contents%20%3D%20(__bridge%20id)_currentFrame.CGImage%3B%0A%7Delse%7B%20%20%20%20%2F%2F%E5%8A%A0%E4%B8%8A%E8%BF%99%E4%B8%AA%20%20%E4%B8%8D%E7%84%B6ios14%E4%BB%A5%E4%B8%8A%E7%9A%84%E7%B3%BB%E7%BB%9F%E7%9C%8B%E4%B8%8D%E8%A7%81%E5%9B%BE%E7%89%87%0A%20%20%20%20%5Bsuper%20displayLayer%3Alayer%5D%3B%0A%7D%0A%60%60%60%0A%23%23%23%23%202%E3%80%81%E7%AC%AC%E4%BA%8C%E7%A7%8D%EF%BC%9A%E5%8D%87%E7%BA%A7reactNative%E7%89%88%E6%9C%AC%2C%E5%A4%A7%E4%BA%8E%E6%88%96%E8%80%85%E7%AD%89%E4%BA%8E0.63%E7%89%88%E6%9C%AC
