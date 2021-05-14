---
title: npm 更新package.json中依赖包版本
tags: ["jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-05-28 14:21
---
文章作者:jqpeng
原文链接: [npm 更新package.json中依赖包版本](https://www.cnblogs.com/xiaoqi/p/npm-package-update.html)

NPM可以使用npm-check-updates库更新版本

1、安装：  
 cnpm install -g npm-check-updates

2、使用：


      ncu --timeout=10000000 -u


指定--timeout参数防止超时

1. 更新全部到最新版本：
    cnpm install


为了防止版本冲突，可以先讲node\_modules删掉

