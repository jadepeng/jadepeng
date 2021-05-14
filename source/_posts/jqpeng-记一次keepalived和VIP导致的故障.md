---
title: 记一次keepalived和VIP导致的故障
tags: ["keepalived","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-08-27 10:33
---
文章作者:jqpeng
原文链接: [记一次keepalived和VIP导致的故障](https://www.cnblogs.com/xiaoqi/p/keepalived.html)

## 起因

nginx服务器采用的keepalived+vip实现的双活，最近由于一台服务器有问题，更换了一台nginx：

操作：

- 停止有问题服务器keepalived和nginx
- 新服务器部署keepalived和nginx


更换后一切正常，但是过了几个小时，出现大面积的不能访问。

## keepalived 升级

检查nginx正常，重启keepalived后OK，怀疑可能是keepalived的问题，于是编译安装最新的keepalived：


    curl --progress http://keepalived.org/software/keepalived-1.2.15.tar.gz | tar xz
    cd keepalived-1.2.15
    ./configure --prefix=/usr/local/keepalived-1.2.15
    make
    sudo make install


升级后，一切正常,。

## 再出故障，最终定位

一晚过去无异常，第二天又出现部分域名不能访问，检查服务一切正常，因此怀疑是VIP导致的问题，检查之前有问题服务器的ip：


    ip addr


果不其然：


    2: eno1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
        link/ether 90:b1:1c:2a:92:e4 brd ff:ff:ff:ff:ff:ff
        inet 172.31.161.42/32 scope global eno1
           valid_lft forever preferred_lft forever
        inet 172.31.161.41/32 scope global eno1
           valid_lft forever preferred_lft forever
        inet 172.31.161.38/24 brd 172.31.161.255 scope global eno1
           valid_lft forever preferred_lft forever
        inet 172.31.161.42/0 scope global eno1::1
           valid_lft forever preferred_lft forever
    3: eno2: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu


41,42 是我们设置的VIP，竟然还在这个有问题服务器的网卡上，这就导致一个机房内，有2台服务器绑定相同的vip。

解决方案, 通过`ip addr delete`删除绑定的vip


    ip addr delete 172.31.161.42/32 dev eno1
    ip addr delete 172.31.161.41/32 dev eno1
    ip addr delete 172.31.161.41/0 dev eno1


顺道介绍如何给网卡绑定vip：


    ip addr add 172.31.161.41/32 dev eno1


## 溯源与问题总结

问题的根源在于，keepalived为网卡停止后，keepalived为网卡绑定的VIP并没有移除，导致多台机器出现同样的ip。

切记： `停止keepalived，vip不会自动删除，需要手动清理`

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


