---
title: makefilen  missing separator. Stop
tags: ["jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-02-19 17:09
---
文章作者:jqpeng
原文链接: [makefilen  missing separator. Stop](https://www.cnblogs.com/xiaoqi/p/10402353.html)

makefile has a very stupid relation with tabs, all actions of every rule are identified by tabs ...... and No 4 spaces don't make a tab, only a tab makes a tab...

makefile使用tab来作为separator.如果你使用4个空格就会报错，makefile:n: \*\*\* missing separator. Stop，其中n是第几行

