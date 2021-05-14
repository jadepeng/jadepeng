---
title: Spring boot web程序static资源放在jar外部
tags: ["Spring Boot","java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-04-04 10:27
---
文章作者:jqpeng
原文链接: [Spring boot web程序static资源放在jar外部](https://www.cnblogs.com/xiaoqi/p/8715860.html)

spring boot程序的static目录默认在resources/static目录， 打包为jar的时候，会把static目录打包进去，这样会存在一些问题：

- static文件过多，造成jar包体积过大
- 临时修改不方便


查看官方文档，可以发现，static其实是可以外置的。

## 方法1 直接修改配置文件


    spring.resources.static-locations=file:///E://resources/static


## 自定义Configuration方法


    @Configuration
    public class StaticResourceConfiguration extends WebMvcConfigurerAdapter {
        @Override
        public void addResourceHandlers(ResourceHandlerRegistry registry) {
            registry.addResourceHandler("/**").addResourceLocations("file:/path/to/my/dropbox/");
        }
    }


推荐使用方法1，安全无害

相关阅读：[Spring Boot配置文件放在jar外部](http://www.cnblogs.com/xiaoqi/p/6955288.html)

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


