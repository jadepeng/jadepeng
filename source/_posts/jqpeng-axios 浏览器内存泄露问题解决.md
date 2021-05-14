---
title: axios 浏览器内存泄露问题解决
tags: ["axios","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-11-04 11:53
---
文章作者:jqpeng
原文链接: [axios 浏览器内存泄露问题解决](https://www.cnblogs.com/xiaoqi/p/axios-memory-leaks.html)

## 现象

业务页面，频繁切换下一条，内存飙涨，导致卡顿，之前怀疑是音频播放器的锅，修改后问题依旧，于是排查网络请求。

到axios issues搜索，发现`memory leaks`帖子不少，典型的在这里[Axios doesn't address memory leaks?](https://github.com/axios/axios/issues/3217):

这里提到`0.19.2 `版本没有问题，但是升级到`0.20.0`后，出现问题。

## 两种解决方案：

- 降级到`0.19.2 `
- 在新版本里，不要直接使用axios，而是先创建一个instance



    const axios = axios.create({...}) // instead of axios.get(), post(), put() etc.


排查业务代码，发现每次请求都是创建一个 instance，抛开版本问题，每次创建实例肯定会存在内存问题，最好还是先创建个single instance，后面复用：


    import axios, { AxiosRequestConfig, AxiosResponse } from 'axios'
    
    // 创建一个实例
    const axiosInstance = axios.create() 
    
      save(parameters: {
        'data': ResultSaveParam,
        $queryParameters?: any,
        $domain?: string
      }): Promise<AxiosResponse<ApiResult>> {
         .... // 使用axiosInstance
        return axiosInstance.request(config)
      }
    


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


