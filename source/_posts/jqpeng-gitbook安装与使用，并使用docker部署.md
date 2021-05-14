---
title: gitbook安装与使用，并使用docker部署
tags: ["docker","gitbook","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-01-04 17:03
---
文章作者:jqpeng
原文链接: [gitbook安装与使用，并使用docker部署](https://www.cnblogs.com/xiaoqi/p/gitbook-setup-and-usage-publish-with-docker.html)

本文简单介绍如何安装并使用gitbook，最后如何使用docker构建书籍镜像。

# 1. 前置条件

需要Nodejs环境，安装npm，国内用户再安装cnpm


    npm install -g cnpm --registry=https://registry.npm.taobao.org


# 2. 安装gitbook


    cnpm install -g gitbook-cli
    gitbook -V 
    CLI version: 2.3.2
    Installing GitBook 3.2.3
    gitbook@3.2.3 ..\AppData\Local\Temp\tmp-20544doJtj1hfVp40\node_modules\gitbook
    ├── escape-string-regexp@1.0.5
    ├── escape-html@1.0.3
     。。。。
    GitBook version: 3.2.3


# 3. gitbook使用

## 3.1 生成目录和图书结构


    mkdir docker-start
    gitbook init
    warn: no summary file in this book
    info: create README.md
    info: create SUMMARY.md
    info: initialization is finished


编辑SUMMARY.md，输入：


    * [简介](README.md)
    * [1.Docker入门](chapter1/README.md)
     - [1.1 什么是Docker](chapter1/section1.md)
     - [1.2 Docker基本概念](chapter1/section2.md)
     - [1.3 安装Docker](chapter1/section3.md)
     - [1.4 使用Docker镜像](chapter1/section4.md)
     - [1.5 操作容器](chapter1/section5.md)
     - [1.6 访问仓库](chapter1/section6.md)
     - [1.6 数据管理](chapter1/section7.md)
    * [2.使用Docker部署web应用](chapter2/README.md)
     - [2.1 编写DockerFile](chapter2/section1.md)
     - [2.2 编写web应用](chapter2/section2.md)
     - [2.3 构建镜像](chapter2/section3.md)
     - [2.4 运行web应用](chapter2/section4.md)
     - [2.5 分享镜像](chapter2/section5.md)
    * [结束](end/README.md)


再次执行：


    gitbook init
    info: create chapter1/README.md
    info: create chapter1/section1.md
    info: create chapter1/section2.md
    info: create chapter1/section3.md
    info: create chapter1/section4.md
    info: create chapter1/section5.md
    info: create chapter1/section6.md
    info: create chapter1/section7.md
    info: create chapter2/README.md
    info: create chapter2/section1.md
    info: create chapter2/section2.md
    info: create chapter2/section3.md
    info: create chapter2/section4.md
    info: create chapter2/section5.md
    info: create end/README.md
    info: create SUMMARY.md
    info: initialization is finished


## 3.2 生成图书

使用：


    gitbook serve .
    Live reload server started on port: 35729
    Press CTRL+C to quit ...
    
    info: 7 plugins are installed
    info: loading plugin "livereload"... OK
    info: loading plugin "highlight"... OK
    info: loading plugin "search"... OK
    info: loading plugin "lunr"... OK
    info: loading plugin "sharing"... OK
    info: loading plugin "fontsettings"... OK
    info: loading plugin "theme-default"... OK
    info: found 16 pages
    info: found 15 asset files
    info: >> generation finished with success in 4.0s !
    
    Starting server ...
    Serving book on http://localhost:4000


访问 [http://localhost:4000](http://localhost:4000) ，就可以看到图书了

![enter description here](http://oyqmmpkcm.bkt.clouddn.com/1515048112496.jpg "1515048112496")

编辑生成的md，gitbook会自动Restart，

![enter description here](http://oyqmmpkcm.bkt.clouddn.com/1515048702182.jpg "1515048702182")

在当前目录下，会生成一个\_book目录 ，里面是生成的静态html，可以发布到服务器直接使用。

# 4. 使用docker发布gitbook书籍

首先 将\_book目录里的内容拷贝到一个新目录。

然后编写Dockerfile


    FROM nginx
    WORKDIR /usr/share/nginx/html
    ADD . /usr/share/nginx/html
    EXPOSE 80


build：


    docker build -t docker-start-web .
    Sending build context to Docker daemon  4.766MB
    Step 1/4 : FROM nginx
     ---> 3f8a4339aadd
    Step 2/4 : WORKDIR /usr/share/nginx/html
    Removing intermediate container a4232f4b6b62
     ---> 91a66299ecad
    Step 3/4 : ADD . /usr/share/nginx/html
     ---> 9a9fef80da3b
    Step 4/4 : EXPOSE 80
     ---> Running in 59f2b829aba6
    Removing intermediate container 59f2b829aba6
     ---> b92c92688046
    Successfully built b92c92688046
    Successfully tagged docker-start-web:latest


执行：


    docker run -p 4000:80 --name docker-start-web -d docker-start-web
    f91cf4446b3746c665476b3dd214446a941d838fa9a3ad47680190bb08c9aa48


访问服务器ip:4000就可以查看到了。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


