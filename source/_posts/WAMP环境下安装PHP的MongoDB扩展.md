---
title: WAMP环境下安装PHP的MongoDB扩展
date: 2016-08-16 20:52:38
tags: [MongoDB, PHP]
---
因为项目需要，我在自己的Win7环境下搭建了WAMP，并且安装了MongoDB数据库。以前没有用过NoSQL数据库，用过之后发现MongoDB真是好用啊！！！不过在环境配置过程中也着实遇到了一些坑，这篇博文记录了安装PHP的MongoDB扩展的过程，自己碰到的坑会重点标识出来。
<!-- more -->
__ _
## 下载MongoDB扩展
Windows下的php扩展都是dll文件，MongoDB的php扩展下载链接：[php for mongodb](http://pecl.php.net/package/mongo)

![1.jpg](https://raw.githubusercontent.com/lirui1992/MarkdownPhotos/master/res/1.jpg)
在选择php_mongo.dll时注意要使dll的版本号与php的版本号保持一致，这里我选择了5.5 Thread Safe (TS) x86这个版本，因为我的php版本是5.5.12，同时我安装的是WAMP 32位版本，这里要注意，*如果你安装的是32位的WAMP，那么一定要选择32为的php_mongo.dll*,否则将无法扩展。
## 安装MongoDB扩展
将下载的php_mongo.dll复制到“wamp\bin\php\php5.5.12\ext”文件夹中。然后需要修改php.ini这个配置文件来让php加载这个扩展。
找到php.ini文件后，添加：
> extension=php_mongo.dll

## 使php_mongo.dll找到libsasl.dll依赖库
libsasl.dll是在php根目录下的一个文件夹，本文的mongodb需要依赖这个dll。**由于wamp安装的过程中不会添加php的环境变量，所以我们在使用php的mongodb扩展的时候，扩展无法找到libsasl.dll的位置导致mongodb的扩展是无法使用的。**
![2.png](https://raw.githubusercontent.com/lirui1992/MarkdownPhotos/master/res/2.png)
很多地方的配置过程都忽略了这一步，我也是因为跳过了这一步，导致爬坑爬了整个下午！谨记！！！
## 测试MongoDB扩展是否配置成功
打开http://localhost/?phpinfo=1， 如果看到下图所示mongo配置成功的信息就说明mongodb的扩展配置成功了。
![3.png](https://raw.githubusercontent.com/lirui1992/MarkdownPhotos/master/res/3.png)



