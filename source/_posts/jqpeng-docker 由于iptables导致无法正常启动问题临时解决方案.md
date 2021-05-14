---
title: docker 由于iptables导致无法正常启动问题临时解决方案
tags: ["docker","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-10-12 00:01
---
文章作者:jqpeng
原文链接: [docker 由于iptables导致无法正常启动问题临时解决方案](https://www.cnblogs.com/xiaoqi/p/13800447.html)

docker安装新的ca证书后无法正常启动，  
 表现为`/sbin/iptables --wait -t filter -N DOCKER-ISOLATION-STAGE-2` hang住，  
 日志有报错 `xtables contention detected while running ..`

解决历程：

1. 重启大法，先stop再start无果
2. iptables 重装大法，无果
3. docker升级到最新版本无果
4. 确定还是iptables的问题，暂时没时间详细排查，于是docker禁用iptables，先卸载iptables，docker启动设置iptables false


