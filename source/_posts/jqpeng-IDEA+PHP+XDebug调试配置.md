---
title: IDEA+PHP+XDebug调试配置
tags: ["xdebug","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-06-19 14:38
---
文章作者:jqpeng
原文链接: [IDEA+PHP+XDebug调试配置](https://www.cnblogs.com/xiaoqi/p/xdbug.html)

## XDebug调试配置

临时需要调试服务器上的PHP web程序，因此安装xdebug，下面简单记录

### 安装xdebug

#### 下载最新并解压


    wget https://xdebug.org/files/xdebug-2.5.4.tgz
    tar zxvf xdebug-2.5.4.tgz 
    cd xdebug-2.5.4/


#### 编译

按照README里的步骤安装


    ./configure --enable-xdebug
    ···
    
    报错
    >checking Check for supported PHP versions... configure: error: not supported. Need a PHP version >= 5.5.0 and < 7.2.0 (found 5.3.10-1ubuntu3.21)
    
    
    原来服务器上的php版本比较低：
    >PHP 5.3.10-1ubuntu3.26 with Suhosin-Patch (cli) (built: Feb 13 2017 20:37:53) 
    Copyright (c) 1997-2012 The PHP Group
    Zend Engine v2.3.0, Copyright (c) 1998-2012 Zend Technologies
    
    最稳妥起见，下载老版本的xdebug，下载2.2.2版本
    
    ``` bash
    wget https://xdebug.org/files/xdebug-2.2.2.tgz
    tar zxvf xdebug-2.2.2.tgz 
    cd xdebug-2.2.2/
    ./configure --enable-xdebug
    make


make完成后，modules下面就有了编译好的xdebug.so:


    root@nginx01:/opt/research/xdebug-2.2.2# ll modules/
    total 808
    drwxr-xr-x 2 root root   4096 Jun 19 14:17 ./
    drwxr-xr-x 9 root root   4096 Jun 19 13:10 ../
    -rw-r--r-- 1 root root    939 Jun 19 13:09 xdebug.la
    -rwxr-xr-x 1 root root 814809 Jun 19 13:09 xdebug.so*


#### 配置

修改php.ini，服务器使用的php5-fpm，配置文件在/etc/php5/fpm/php.ini

修改，增加xdebug配置信息


    zend_extension="/opt/research/xdebug-2.2.2/modules/xdebug.so"
    xdebug.remote_enable = On
    xdebug.remote_handler = dbgp
    xdebug.remote_port = 9001 #端口9001
    xdebug.remote_connect_back = 1 
    #xdebug.remote_host= 192.168.xxx.xxx
    xdebug.idekey = PHPSTORM
    xdebug.remote_log = /opt/research/xdebug-2.2.2/xdebug.log


### IDEA 配置

#### 配置xdebug端口为9001

在设置里搜索XDEBUG，配置端口9001  
![enter description here](https://ooo.0o0.ooo/2017/06/19/5947709e37cf4.jpg "1497854105384")

#### 调试配置

在RUN-Edit Configuratins里，新增PHP Web Application  
![enter description here](https://ooo.0o0.ooo/2017/06/19/5947707361211.jpg "1497854062372")

Server新增服务器地址，Debugger设置为Xdebug，将服务器上的绝对地址，映射到本地

![XDEBUG配置](https://ooo.0o0.ooo/2017/06/19/59477034b6e7f.jpg "1497853999274")

然后就可以启动调试了

