---
title: 构建我的hexo博客
date: 2021-05-27 14:00:00
tags: 博客
categories: 博客
comments: true
typora-root-url: ../
---

## 安装Hexo环境

### 安装前准备

首先，安装Hexo环境需要提前在自己的本地安装 Git 、NodeJs，对于这两个软件熟悉的人来说应该很简单。如果不熟，可以参考以下两个链接：

* 安装git：https://blog.csdn.net/huangqqdy/article/details/83032408
* 安装nodeJs：https://www.cnblogs.com/liuqiyun/p/8133904.html

在安装好nodeJs之后，即可以执行node的内置命令：npm ，hexo 的后续安装都是基于此命令进行的。

### 安装配置Hexo

1、首先，使用windows下 CMD、PowerShell、Git-Bash 安装 Hexo 插件：

```shell
$ npm install -g hexo-cli
```

2、然后，创建你的博客文件夹`blog`，并进行初始化：

```shell
$ cd blog
$ hexo init
$ npm install
```

此时，你的命令行可能提示：`Unsupported platform for fsevents@2.3.2: wanted {"os":"darwin","arch":"any"} (current: {"os":"win32","arch":"x64"})` ，经过查阅资料，fsevents 是 MacOS 下使用的依赖，提示 Windows 下不支持，所以可以忽略。当没有其他明显的Error异常下，可以默认为时正常的。

3、接下来，给 Hexo 安装内置插件：

```shell
$ npm install hexo-deployer-git --save
$ npm install hexo-server
```

4、最后，新建一篇测试效果的文章。

```shell
$ hexo new  "my-article"
```

没错，hexo 支持直接使用命令创建一篇文章，文章会自动保存在博客根路径的： `source/_posts`目录下。然后，我们可以开始在 `source/_posts/my-article.md` 文件里，写 markdown 格式的文章了。

5、启动博客。可以使用以下命令启动 hexo内置 hexo server 服务，方便通过网页查看博客网站内容。

```shell
$ hexo s
INFO  Validating config
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```

之后，通过网页直接访问：http://localhost:4000 就能访问你的博客了。

## 配置域名

我的博客是放在阿里云服务器的，所以需要购买阿里云ECS云服务器，另外还需购买一个自定义域名。

### 购买阿里云ECS

通过注册、登录阿里云首页，直接搜索 ECS，然后找到 "新用户专享"栏目下，找适合自己的阿里云服务器。

![image-20220309155358091](/images/my_blog_init/image-20220309155358091.png)

购买服务器时，可以关注以下几个配置：

* 实例：1G
* 地域：华南(深圳)
* 云盘：40G
* 网络：专有网络
* 带宽：1Mbps

因为是新人特惠，所以我选择3年。

### 注册域名

通过阿里云首页，直接搜索 "域名注册" 进入专栏，搜索自己心仪的域名，然后就是购买了。

![image-20220309160502109](/images/my_blog_init/image-20220309160502109.png)

我注册的域名是：tinkun.top，根据后缀不同，价格不同，我选择的最便宜的哈哈。

### 域名备案

上面购买了域名，但并不意味着你马上就可以拿来用了，需要经过认证、备案、审核。具体事宜可以参考阿里云导航栏 "IPC备案" 专栏了解。注意，域名备案需要你有一台服务器，如果上面购买了服务器则忽略这句话。

域名备案需要几天时间（3-9天），请耐心等待。

