# ios css border-radius元素 overflow:hidden失效问题

## 问题

父级设置圆角属性失效 父元素使用border-radius和overflow:hidden做成圆形，子元素如果使用了transform属性，则父元素的overflow:hidden会失效。

## 解决

```
-webkit-backface-visibility: hidden;
-webkit-transform: translate3d(0, 0, 0);
```

%23%23%20%E9%97%AE%E9%A2%98%0A%E7%88%B6%E7%BA%A7%E8%AE%BE%E7%BD%AE%E5%9C%86%E8%A7%92%E5%B1%9E%E6%80%A7%E5%A4%B1%E6%95%88%20%E7%88%B6%E5%85%83%E7%B4%A0%E4%BD%BF%E7%94%A8border-radius%E5%92%8Coverflow%3Ahidden%E5%81%9A%E6%88%90%E5%9C%86%E5%BD%A2%EF%BC%8C%E5%AD%90%E5%85%83%E7%B4%A0%E5%A6%82%E6%9E%9C%E4%BD%BF%E7%94%A8%E4%BA%86transform%E5%B1%9E%E6%80%A7%EF%BC%8C%E5%88%99%E7%88%B6%E5%85%83%E7%B4%A0%E7%9A%84overflow%3Ahidden%E4%BC%9A%E5%A4%B1%E6%95%88%E3%80%82%0A%0A%23%23%20%E8%A7%A3%E5%86%B3%0A%60%60%60%0A-webkit-backface-visibility%3A%20hidden%3B%0A-webkit-transform%3A%20translate3d(0%2C%200%2C%200)%3B%0A%60%60%60
