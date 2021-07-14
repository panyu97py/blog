# mac homebrew安装的mysql忘记密码

试了网上很多种方法都不行，还是暴力解决吧！

1. 通过vim修改配置文件, 命令行输入一下命令
    `vim /usr/local/etc/my.cnf`
    按 i 进编辑，添加 skip-grant-tables 到文件最后一行
    按  esc 后按:wq 退出
2. 重启
    `brew services restart mysql`
    `mysql -u root -p`需要输入密码，直接回车,进入mysql



```php
flush privileges; // —刷新
use mysql;
alter user 'root'@'localhost' identified by 'Root!123'; // 你的密码
```

修改密码成功后的图如下所示：



![img](https:////upload-images.jianshu.io/upload_images/6550466-d515639789d0c8e1.png?imageMogr2/auto-orient/strip|imageView2/2/w/1072/format/webp)

image.png

1. 恢复my.cnf，把加的那个skip-grant-tables删除
2. 重启  `brew services restart mysql`

再重新进入输入你改的密码就好了



作者：noyanse
链接：https://www.jianshu.com/p/c5a023a28a5b
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。