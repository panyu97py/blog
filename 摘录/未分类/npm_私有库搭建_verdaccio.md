# npm 私有库搭建 verdaccio

### 安装

```
npm install -g verdaccio
```

### 安装过程出现的问题

#### 找不到python或者版本不符

> 解决方法：安装`2.7`版本的 `python`

官网下载 `Python-2.7.18.tgz` 后，解压

```
tar -zxvf Python-2.7.18.tgz
```

配置安装路径

```
cd Python-2.7.18
 ./configure  --prefix=/usr/local
```

安装

```
make&&sudo make install
```

完成后`npm`配置`python`环境

```
npm config set python python2.7
```

#### leveldown 安装错误

> 升级 `node` 版本

#### node 环境配置

##### 设置全局安装的默认目录

```
npm config set prefix "node_global"
```

##### 获取全局安装的默认目录

```
npm config get prefix
```

##### 设置局部安装的默认缓存目录

```
npm config set cache "node_cache"
```

##### 获取局部安装的默认缓存目录

```
npm config get cache
```

%23%23%23%20%E5%AE%89%E8%A3%85%0A%0A%60%60%60%0Anpm%20install%20-g%20verdaccio%0A%60%60%60%0A%0A%23%23%23%20%E5%AE%89%E8%A3%85%E8%BF%87%E7%A8%8B%E5%87%BA%E7%8E%B0%E7%9A%84%E9%97%AE%E9%A2%98%0A%0A%23%23%23%23%20%E6%89%BE%E4%B8%8D%E5%88%B0python%E6%88%96%E8%80%85%E7%89%88%E6%9C%AC%E4%B8%8D%E7%AC%A6%0A%0A%3E%20%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95%EF%BC%9A%E5%AE%89%E8%A3%85%602.7%60%E7%89%88%E6%9C%AC%E7%9A%84%20%60python%60%0A%0A%E5%AE%98%E7%BD%91%E4%B8%8B%E8%BD%BD%20%60Python-2.7.18.tgz%60%20%E5%90%8E%EF%BC%8C%E8%A7%A3%E5%8E%8B%0A%0A%60%60%60%0Atar%20-zxvf%20Python-2.7.18.tgz%0A%60%60%60%0A%E9%85%8D%E7%BD%AE%E5%AE%89%E8%A3%85%E8%B7%AF%E5%BE%84%0A%60%60%60%0Acd%20Python-2.7.18%0A%20.%2Fconfigure%20%20--prefix%3D%2Fusr%2Flocal%0A%60%60%60%0A%E5%AE%89%E8%A3%85%0A%60%60%60%0Amake%26%26sudo%20make%20install%0A%60%60%60%0A%E5%AE%8C%E6%88%90%E5%90%8E%60npm%60%E9%85%8D%E7%BD%AE%60python%60%E7%8E%AF%E5%A2%83%0A%60%60%60%0Anpm%20config%20set%20python%20python2.7%0A%60%60%60%0A%0A%23%23%23%23%20leveldown%20%E5%AE%89%E8%A3%85%E9%94%99%E8%AF%AF%0A%0A%3E%20%E5%8D%87%E7%BA%A7%20%60node%60%20%E7%89%88%E6%9C%AC%0A%0A%23%23%23%23%20node%20%E7%8E%AF%E5%A2%83%E9%85%8D%E7%BD%AE%0A%0A%23%23%23%23%23%20%E8%AE%BE%E7%BD%AE%E5%85%A8%E5%B1%80%E5%AE%89%E8%A3%85%E7%9A%84%E9%BB%98%E8%AE%A4%E7%9B%AE%E5%BD%95%0A%60%60%60%0Anpm%20config%20set%20prefix%20%22node_global%22%0A%60%60%60%0A%0A%23%23%23%23%23%20%E8%8E%B7%E5%8F%96%E5%85%A8%E5%B1%80%E5%AE%89%E8%A3%85%E7%9A%84%E9%BB%98%E8%AE%A4%E7%9B%AE%E5%BD%95%0A%60%60%60%0Anpm%20config%20get%20prefix%0A%60%60%60%0A%23%23%23%23%23%20%E8%AE%BE%E7%BD%AE%E5%B1%80%E9%83%A8%E5%AE%89%E8%A3%85%E7%9A%84%E9%BB%98%E8%AE%A4%E7%BC%93%E5%AD%98%E7%9B%AE%E5%BD%95%0A%60%60%60%0Anpm%20config%20set%20cache%20%22node_cache%22%0A%60%60%60%0A%23%23%23%23%23%20%E8%8E%B7%E5%8F%96%E5%B1%80%E9%83%A8%E5%AE%89%E8%A3%85%E7%9A%84%E9%BB%98%E8%AE%A4%E7%BC%93%E5%AD%98%E7%9B%AE%E5%BD%95%0A%60%60%60%0Anpm%20config%20get%20cache%0A%60%60%60
