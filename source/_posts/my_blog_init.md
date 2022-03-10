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

## 配置阿里云ECS

在域名备案的时间里，可以进行阿里云ECS配置，通过配置实例密码和安全组，就可以登录服务器操作了。

### 配置ECS密码

在配置好ECS后，如果没有设置服务器登录密码是不能登录到阿里云服务器的。此时，可以设置阿里云服务器的登录密码：默认用户root，密码自己设置。

![img](/images/my_blog_init/20200919_1609.png)

### 配置安全组

阿里云的服务器默认不开放端口号，这样使得我们在网站部署完成之后仍然无法访问。

有一个基本原因是没有开启端口号，因此我们需要新建安全组并添加 80 端口，再将安全组添加到 ECS 实例中。具体操作如下。

在控制台的 ECS 实例中点击「网络与安全–>安全组–>创建安全组–>快速添加」。在访问规则的**入方向**添加如下几个端口（尤其是80端口）：

![img](/images/my_blog_init/20200919_1616.png)

然后回到 ECS 服务器实例，将刚刚配置的安全组加入到实例中：

![img](/images/my_blog_init/20200919_1620.png)

## 服务器端配置

通过上面【配置安全组】，我们配置好了我们的ECS服务器能过入网的端口，此时我们就可以通过工具登录阿里云服务器了。

接下来，我们进行服务器端的配置，方便我们从外部访问到我们的博客。

### 安装nginx

这里安装nginx做反向代理，提供博客的代理访问。

* 本地生成hexo博客静态资源文件
* 推送静态资源文件到阿里云服务器
* 外部通过nginx访问到这些静态资源

(1) 首先，安装nginx依赖

```shell
[root@xxxxx ~]# yum install -y gcc-c++
[root@xxxxx ~]# yum install -y pcre pcre-devel
[root@xxxxx ~]# yum install -y zlib zlib-devel
[root@xxxxx ~]# yum install -y openssl openssl-devel
```

(2) 然后，安装nginx服务

```shell
# 下载nginx资源包
[root@xxxxx ~]# wget -c https://nginx.org/download/nginx-1.18.0.tar.gz

# 解压nginx
[root@xxxxx ~]# tar -zxvf nginx-1.18.0.tar.gz -C /usr/local

# 安装nginx
[root@xxxxx ~]# cd /usr/local/nginx-1.18.0
[root@xxxxx nginx-1.18.0]# ./configure --prefix=/usr/local/soft/nginx --with-http_stub_status_module --with-http_ssl_module
```

注：

* –prefix 指定安装路径

* –with-http_stub_status_module 允许查看nginx状态的模块
* –with-http_ssl_module 支持https的模块

(3) 编译nginx

```shell
[root@xxxxx nginx-1.18.0]# make
[root@xxxxx nginx-1.18.0]# make install
```

之后，nginx就被安装在了路径：`/usr/local/soft/nginx`

(4) 由于 nginx 默认通过 80 端口访问，而 Linux 默认情况下不会开发该端口号，因此需要开放 linux 的 80 端口供外部访问

```shell
[root@xxxxx nginx-1.18.0]# /sbin/iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```

(5) 启动nginx

```shell
[root@xxxxx ~]# cd /usr/local/soft/nginx/sbin/
[root@xxxxx sbin]# ./nginx

# 查看nginx是否启动
[root@xxxxx sbin]# ps -ef|grep nginx
root       72683       1  0 2月28 ?       00:00:00 nginx: master process ./nginx
root     2330542 2305235  0 20:02 pts/0    00:00:00 grep --color=auto nginx
root     4073802   72683  0 3月03 ?       00:00:03 nginx: worker process
```

如上，显示出了nginx的进程号就是成功启动了。现在，就可以输入公网 IP 即可进入 nginx 的欢迎页面了

![img](/images/my_blog_init/20200919_1753.png)

### 配置nginx路由

这里的配置指的是，博客静态资源代理路径，也就是将来我们博客生成的静态文件存放的位置。

(1) 创建 hexo 博客资源存放路径：/home/tim/hexo

```shell
[root@xxxxx ~]# mkdir -p /home/tim/hexo
```

(2) 配置nginx代理路径和代理域名

```shell
[root@xxxxx ~]# cd /usr/local/soft/nginx/conf/
[root@xxxxx conf]# vim nginx.conf
user  root;                                          ---(1)用户改为 root
worker_processes  1;
... 

http {
    include       mime.types;
    default_type  application/octet-stream;
	...
	
	server {
        listen       80;
        server_name timkun.top www.timkun.top;      ---(2)将 `server_name` 改为自己的域名

		...
        location / {
            root   /home/tim/hexo;               ---(3)将location中的 `root` 项中的值改为 `/home/tim/hexo`
            index  index.html index.htm;
        }
...
```

主要修改三处位置：

* 用户改为 root

* 将 `server_name` 改为自己的域名。如果没有备案，可以先填写自己的公网 IP（在阿里云控制台的 ECS 实例中查看），访问时暂时用公网 IP 进行访问
* 将location中的 `root` 项中的值改为 `/home/tim/hexo`

修改完成之后，再次重启nginx即可：完成nginx代理资源路径。

### 安装NodeJs

安装 node.js的命令如下：

```shell
[root@xxxxx /]# cd ~
[root@xxxxx ~]# curl -sL https://rpm.nodesource.com/setup_10.x | bash -
[root@xxxxx ~]# yum install -y nodejs
```

过查看版本号验证是否安装成功：

```shell
[root@xxxxx ~]# node -v
[root@xxxxx ~]# npm -v
```

### 安装Git服务

(1) 为了使我们能够在本地向服务器实现自动部署，需要在服务器端另外新建一个 Git 用户。然后使用公钥连接成功之后，就可以方便地随时进行自动部署了。

```shell
[root@xxxxx ~]# yum install -y git
[root@xxxxx ~]# git --version
git version 2.27.0
```

(2) 创建git用户，并在root角色下给git用户设置密码。

```shell
[root@xxxxx ~]# adduser git

# 修改git用户密码
[root@xxxxx ~]# passwd git
更改用户 git 的密码 。
新的 密码：
重新输入新的 密码：
passwd：所有的身份验证令牌已经成功更新。
```

(3) 给git用户角色增加root执行权限（后续可以再git角色使用：`sudo command` 执行命令了）

```shell
vim /etc/sudoers
```

进入文件后，后按 `i` 键由命令模式切换到编辑模式。如下图所示，在 root 下添加一行 Git 信息：

![image-20220310121031491](/images/my_blog_init/image-20220310121031491.png)

修改结束后，先按 `Esc` 由编辑模式切换到命令模式，再输入`:wq` 命令保存并退出编辑器。

以上我们就完成了 Git 用户的创建，接下来我们向 Git 用户添加公钥，就像配置 Github 那样。

### 设置git用户免登录

通过配置，使得我们在本地电脑：在免登录状态，上传和部署博客到服务器端。包括直接在本地部署博客时，hexo会自动提交静态资源到阿里云服务器，并部署。具体操作流程如下：

- 先在本地的`C:\Users\用户名.ssh`目录生成公钥`id_rsa.pub`和私钥`id_rsa`；
- 然后使用 FTP 上传工具，将公钥文件`id_rsa.pub`上传到服务器端的 `.ssh` 文件夹；
- 最后将公钥文件`id_rsa.pub`内容拷贝到 `authorized_keys` 文件中。

当然，如果你本地以前生成过公钥文件`id_rsa.pub`，即可直接传到服务器进行配置。接下来将介绍具体的配置步骤。

(1) 在你的本地生成ssh秘钥。（本地指的是你的 windows 或 Mac 电脑）

```shell
# windows下生成ssh秘钥
# 直接在本地打开cmd或其他命令行客户端，输入：ssh-keygen -t rsa -C “邮箱名”
Microsoft Windows [版本 10.0.19043.1586]
(c) Microsoft Corporation。保留所有权利。
C:\Users\TIM> ssh-keygen -t rsa -C "your.emall@gmall.com"
... 一路回车(默认即可)...
```

最终：C:\Users\用户名\.ssh 目录下，会看到有两个文件：id_rsa, id_rsa.pub。有 `.pub` 后缀的文件就是公钥，另一个文件则是密钥

```shell
# macOs下生成ssh秘钥
# 打开Mac系统自带的命令行终端，输入：ssh-keygen
$ $ cd ~
$ cd .ssh
$ ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/Users/smyhvae/.ssh/id_rsa):
```

最终：C:\Users\用户名\.ssh 里生成两个文件：公钥文件 `id_rsa.pub`、私钥文件`id_rsa`

**注意：`.ssh` 为隐藏文件夹，你可能需要显示隐藏文件夹之后才可以看到**

(2) 登录到阿里云服务器 Git 用户角色下，并创建 .ssh 文件夹。

```shell
[root@xxxxx ~]# su - git
[git@xxxxx ~]# mkdir .ssh
```

(3) 将本地生成的ssh公钥文件`id_rsa.pub` 文件上传到服务器端的`/home/git/.ssh`目录下：

这一步可以使用FTP客户端，或者服务器端安装RZ内嵌命令执行上传。注意：上传目标路径是：`/home/git/.ssh/` 下面。

```shell
[git@xxxxx ~]# cd ~ && ll -a
-rw-r--r-- 1 root root 753 2月  28 19:33 id_rsa.pub
```

(4) 再次登录到服务器，用 Git 用户身份在 `.ssh` 文件夹内新建 `authorized_keys` 文件，并将公钥内容拷贝到该文件中：

```shell
[git@xxxxx ~]# cd ~/.ssh
[git@xxxxx .ssh]# cp id_rsa.pub authorized_keys
[git@xxxxx .ssh]# cat id_rsa.pub >> ~/.ssh/authorized_keys
```

修改文件权限：

```shell
[git@xxxxx .ssh]# chmod 600 ~/.ssh/authorized_keys
[git@xxxxx .ssh]# chmod 700 ~/.ssh
```

(5) 确保设置了正确的SELinux上下文：

```shell
[git@xxxxx .ssh]# restorecon -Rv ~/.ssh
```

现在我们来验证一下，在 Windows / MaxOs 输入如下命令，是否能正常连接到远程服务器：（不用输入密码，就能直接连上）

```shell
$ ssh git@120.79.xx.xx（你的公网 IP）

Welcome to Alibaba Cloud Elastic Compute Service !

Updates Information Summary: available
    11 Security notice(s)
         1 Critical Security notice(s)
         2 Important Security notice(s)
         8 Moderate Security notice(s)
Run "dnf upgrade-minimal --security" to apply all updates.
Last login: Thu Mar 10 10:23:46 2022 from 59.40.1xx.1x
[git@xxxxx ~]$
```

如上，我们直接使用ssh命令就登录到了服务器git角色下。相同的，如果命令换成：`ssh root@120.79.xx.xx` 则需要登录密码才能登录。

### 配置Hexo的Git仓库

在上几节中，我们创建了 hexo 博客资源存放路径：/home/tim/hexo，以及安装了：Nginx、Git、NodeJs，接下来我们将创建git的裸仓库（不明白的可以网上搜索：git bare库，具体目的就是只作为一个分享仓库，不作为实际代码仓库），

(1) 登录ECS服务器，使用 **Git 用户** 创建 git 仓库，并新建 post-receive 钩子文件：

```shell
[root@xxxxx ~]$ su - git
[git@xxxxx ~]$ git init --bare hexo.git

# 新建文件
[git@xxxxx ~]$ touch ~/hexo.git/hooks/post-receive

# 修改文件权限
[git@xxxxx ~]$ chmod +x ~/hexo.git/hooks/post-receive

# 编辑钩子文件，并输入以下内容：git --work-tree=/home/tim/hexo --git-dir=/home/git/hexo.git checkout -f
[git@xxxxx ~]$ vim ~/hexo.git/hooks/post-receive
git --work-tree=/home/tim/hexo --git-dir=/home/git/hexo.git checkout -f
```

修改完成后，先按 `Esc` 由编辑模式切换到命令模式，再输入 `:wq` 命令保存并退出编辑器。

(2) 修改文件权限：

```shell
[git@xxxxx ~]$ sudo chmod -R 777 /home/tim/hexo
```

到此，我们就完成了服务端的配置。

## 本地部署Hexo博客

经过前面的准备工作，接下来我们就可以进行hexo博客部署了，通过部署后就可以通过：http://120.79.xx.xx(公网ip) 或 http://timkun.top 直接访问到我们的博客了。

### 更新Hexo配置文件

至此，准备工作都差不多已经完成，接下来需要修改 `_config.yml` 文件。进入本地计算机 `blog` 文件夹的根目录，找到 `_config.yml` 文件并打开。以下，我将列出一些可以修改的地方(可以按关键字全文搜索并修改)，其他地方胖友们可以自行探索。

```shell
$ vim _config.yml
# Site
title: 蒂姆的个人博客
subtitle: 'Tim`s blog'
description: ''
keywords:
author: Tim
language: zh-CN
timezone: ''

# URL 配置自己的域名，如果域名未备案则填写ECS公网IP
url: https://timkun.top
... 

# Extensions
#theme: 修改主题，这里直接修改没用，需要安装主题扩展插件
theme: hexo-theme-fluid

# Deployment
## Docs: https://hexo.io/docs/one-command-deployment
deploy:
  type: git
  repo: git@120.79.xx.xx:/home/git/hexo.git
  branch: master
```

### Hexo 部署和发布

后续编写的文章都需要放在`blog\source\_posts`目录下，执行如下三个命令，就可以将文章发布到服务器端了。

```shell
$ hexo clean
$ hexo generate
$ hexo deploy
```

或者可以使用快捷命令：

```shell
$ hexo hexo cl && hexo g && hexo d
```

此时，**在浏览器中输入自己的公网 IP，你就可以打开你的博客网站了，Nice!!!**

![image-20220310121206358](/images/my_blog_init/image-20220310121206358.png)

### 域名DNS解析

当我们的域名备案成功之后，我们就有能力使用域名登陆自己的博客了。在此之前，需要在阿里云 `控制台-域名` 中设置域名解析。

点击“解析”：

![image-20220310121532904](/images/my_blog_init/image-20220310121532904.png)

在DNS解析设置里同时添加两条指向公网ip的主机记录：一条`@`记录，一条`www`记录。如下：

![image-20220310121807800](/images/my_blog_init/image-20220310121807800.png)

此时，在浏览器输入`www.timkun.top`或者`timkun.top`，就可以打开我的博客网站了。它们都是基于http协议的，等同于`http://www.timkun.top`

## 配置https访问

在上面的步骤中，网站域名只支持了 http，还没有支持https。所以，当我输入`https://www.timkun.top`时，网站是不打开的。那要怎么让网站域名支持 https呢？我们可以为域名添加免费证书，添加证书后，网站将变成安全的 https。

整体流程如下:![img](/images/my_blog_init/20200920_1511.png)

### 购买SSL证书

在阿里云首页搜索：“SSL证书”，点击【选购SSL证书】进入到购买页面，在【SSL证书服务】项选择 “DV单域名证书【免费使用】”，直接购买。**注意，选择其他选项会触发收费项目。**

![image-20220310143613593]1(/images/my_blog_init/image-20220310143613593.png)

按照步骤的流程点击之后，域名解析里会**自动**多出下面这一条解析：

![image-20220310144533455](/images/my_blog_init/image-20220310144533455.png)

### 下载SSL证书

在上面购买且自动生成解析配置之后，则可以到【控制台】-【SSL证书（应用安全）】栏目下看到自己的免费SSL证书了。

![image-20220310144839229](/images/my_blog_init/image-20220310144839229.png)

点击下面的【下载】按钮，再次选择【Nginx】下载，即可下载SSL证书秘钥包到你的本地机器。

![image-20220310145106475](/images/my_blog_init/image-20220310145106475.png)

### 配置SSL证书

按照上面的步骤下载完成后，会得到一个`7xxxxx_timkun.top_nginx.zip`压缩包，将压缩包解压后，会看到两个文件：`7xxxxx_timkun.top.pem`、`7xxxxx_timkun.top.key`。我们本地将它解压。

然后，将他们上传到我们的服务器，并配置到 nginx 代理的证书配置上，使我们的网站拥有 https 访问能力。

```shell
[git@xxxxx ~]$ cd /usr/local/soft/nginx/
[git@xxxxx nginx]$ mkdir cert

# 然后使用rz 或者 ftp将上述解压的：7xxxxx_timkun.top.pem、7xxxxx_timkun.top.key 上传到 /cert文件夹下
[git@xxxxx nginx]$ cd cert
[git@xxxxx cert]$ ll
总用量 8
-rw-r--r-- 1 root root 1679 3月   1 11:51 7xxxxx_timkun.top.key
-rw-r--r-- 1 root root 3813 3月   1 11:51 7xxxxx_timkun.top.pem
```

最后，也是重要的一步：修改nginx配置文件，开放443端口，且配置SSL证书文件地址。

```shell
[git@xxxxx ~]$ cd /usr/local/soft/nginx/conf
[git@xxxxx conf]$ vim nginx.conf

# 整理修改如下：去除这块代码的注释，并且配置
server {
        listen       443 ssl;
        server_name  timkun.top www.timkun.top;                                  --1、配置上我们的域名/公网IP

        ssl_certificate      /usr/local/soft/nginx/cert/7xxxxx_timkun.top.pem;   --2、配置上pem文件的地址
        ssl_certificate_key  /usr/local/soft/nginx/cert/7xxxxx_timkun.top.key;   --3、配置上key文件的地址

        ssl_session_cache    shared:SSL:1m;
        ssl_session_timeout  5m;

        ssl_ciphers  HIGH:!aNULL:!MD5;
        ssl_prefer_server_ciphers  on;

        location / {
            root   /home/dingmk/hexo;
            index  index.html index.htm;
        }
    }
```

最后的最后，重启nginx

```shell
[git@xxxxx ~]$ /usr/local/soft/nginx/sbin
[git@xxxxx sbin]$ ./nginx -s reload
```

然后在网页访问：https://timkun.top  或者  https://www.timkun.top 即可访问到我们的博客了！！！

### HTTP重定向

如果还是按照http访问的话，我们可以强制让http请求重定向到https上，具体做法是：当http访问时，nginx转发https访问。

继续修改 nginx 文件，修改原有端口 80 的监听，加一行配置：

```shell
server {
	listen 80;
	server_name timkun.top www.timkun.top;
	return 301 https://timkun.top$request_uri;
	...
}
```

修改之后，当用户使用 http 协议访问网站时，会自动进行 301 跳转，以 https 协议访问网站。

## 博客主题

如果你按照上面的步骤搭建了属于自己的博客，会发现和我的不一样，这是由于需要更换主题。我的主题是：[hexo-theme-fluid](https://github.com/fluid-dev/hexo-theme-fluid) ，你也可以去 Github 搜索其他的主题。

关于更换主题，其实很简单，如我使用的主题 [hexo-theme-fluid](https://github.com/fluid-dev/hexo-theme-fluid) ，点击进入github源码页面，直接点击下载zip包到本地，并解压备用。

![image-20220310152121939](/images/my_blog_init/image-20220310152121939.png)

然后，将解压的文件夹，放置到你博客的 `/blog/themes/` 文件夹下：

![image-20220310152401667](/images/my_blog_init/image-20220310152401667.png)

然后，修改自己博客根路径下的配置文件：`/blog/_config.yml`

```shell
$ vim _config.yml
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
#theme: landscape 
theme: hexo-theme-fluid
```

之后就大功告成了！

最后，其他的配置可以参考主题【hexo-theme-fluid】的官方文档：*https://hexo.fluid-dev.com/docs/guide/*

## 源码管理

为了我们编写好的博客文档更好的管理，需要将我们生成的`/blog/` 文件夹下所有的东西都托管到代码仓库中心，可以是 GitHub、Gitee或其他仓库，这里我放在了GitHub，Gitee从GitHub不定时同步。

具体步骤这里不细说了，就是登陆GItHub，创建新仓库，并将`/blog/`下的内容都提交上去。**注意：提交前，请先执行`hexo cl`清除无关文件。**

最后，祝各位愉快的写博客啦~~~

## 参考链接

[hexo+阿里云搭建博客网站](https://www.qianguyihao.com/post/2020-09-19-hexo-aliyun-blog/)

[我终于拥有自己的独立博客了](https://penghh.fun/2020/10/21/2020-10-21-post01/)













































