---
title: Vue 使用typescript， 优雅的调用swagger API
tags: ["jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-05-22 18:31
---
文章作者:jqpeng
原文链接: [Vue 使用typescript， 优雅的调用swagger API](https://www.cnblogs.com/xiaoqi/p/generator-swagger-2-ts-2.html)

Swagger 是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务，后端集成下Swagger，然后就可以提供一个在线文档地址给前端同学。

![swagger](https://gitee.com/jadepeng/pic/raw/master/pic/2020/5/22/1590142373032.png)

前端如何优雅的调用呢？

## 入门版

根据文档，用axios自动来调用


    // 应用管理相关接口
    import axios from '../interceptors.js'
    
    // 获取应用列表
    export const getList = (data) => {
      return axios({
        url: '/app/list?sort=createdDate,desc',
        method: 'get',
        params: data
      })
    }


这里的问题是，有多少个接口，你就要编写多少个函数，且数据结构需要查看文档获取。

## 进阶版本

使用typescript，编写API，通过Type定义数据结构，进行约束。

问题： 还是需要手写

## 优雅版本

swagger 其实是一个json-schema描述文档，我们可以基于此，自动生成。

很早之前，写过一个插件 [generator-swagger-2-t](https://www.npmjs.com/package/generator-swagger-2-ts), 简单的实现了将swagger生成typescript api。

今天，笔者对这个做了升级，方便支持后端返回的泛型数据结构。

### 安装

需要同时安装 [Yeoman](http://yeoman.io) 和 -swagger-2-ts


    npm install -g generator-swagger-2-ts


然后cd到你的工作目录，执行:


    yo swagger-2-ts


按提示

- 输入swagger-ui 地址，例如http://192.168.86.8:8051/swagger-ui.html
- 可选生成js 或者 typescript
- 可以自定义生成的api class名称、api文件名
- API 支持泛型


也可以通过命令行直接传递参数


     yo swagger-2-ts --swaggerUrl=http://localhost:8080/swagger-ui.html --className=API --type=typescript --outputFile=api.ts


- swaggerUrl: swagger ui url swaggerui地址
- className： API class name 类名
- type： typescript or javascipt
- outputFile: api 文件保存路径


### 生成代码demo：


    
    
    export type AccountUserInfo = {
      disableTime?: string
      isDisable?: number
      lastLoginIp?: string
      lastLoginPlace?: string
      lastLoginTime?: string
      openId?: string
    }
    
    
    export type BasePayloadResponse = {
      data?: object
      desc?: string
      retcode?: string
    
    }
    
    /**
     * User Account Controller
     * @class UserAccountAPI
     */
    export class UserAccountAPI {
    /**
      * changeUserState
      * @method
      * @name UserAccountAPI#changeUserState
      
      * @param  accountUserInfo - accountUserInfo 
      
      * @param $domain API域名,没有指定则使用构造函数指定的
      */
      changeUserState(parameters: {
        'accountUserInfo': AccountUserInfo,
        $queryParameters?: any,
        $domain?: string
      }): Promise<AxiosResponse<BasePayloadResponse>> {
    
        let config: AxiosRequestConfig = {
          baseURL: parameters.$domain || this.$defaultDomain,
          url: '/userAccount/changeUserState',
          method: 'PUT'
        }
    
        config.headers = {}
        config.params = {}
    
        config.headers[ 'Accept' ] = '*/*'
        config.headers[ 'Content-Type' ] = 'application/json'
    
        config.data = parameters.accountUserInfo
        return axios.request(config)
      }
    
      _UserAccountAPI: UserAccountAPI = null;
    
      /**
      * 获取 User Account Controller API
      * return @class UserAccountAPI
      */
      getUserAccountAPI(): UserAccountAPI {
        if (!this._UserAccountAPI) {
          this._UserAccountAPI = new UserAccountAPI(this.$defaultDomain)
        }
        return this._UserAccountAPI
      }
    }
    
    
    /**
     * 管理系统接口描述
     * @class API
     */
    export class API {
      /**
       *  API构造函数
       * @param domain API域名
       */
      constructor(domain?: string) {
        this.$defaultDomain = domain || 'http://localhost:8080'
      }
    }
    
    


### 使用


    import { API } from './http/api/manageApi'
    // in main.ts
    let api = new API("/api/")
    api.getUserAccountAPI().changeUserState({
      isDisable: 1
      openId: 'open id'
    })


## Vue中最佳实践

### main.ts 全局定义


    import { API } from './http/api/manageApi'
    
    Vue.prototype.$manageApi = new API('/api/')


### 增加.d.ts

增加types文件，方便使用智能提示


    import { API } from '@/http/api/manageApi'
    import { MarkAPI } from '@/http/api/mark-center-api'
    declare module "vue/types/vue" {
      interface Vue {
        $manageApi: API
        $markApi: MarkAPI
      }
    }


### 实际使用

现在可以在vue里直接调用了。

![vscode调用](https://gitee.com/jadepeng/pic/raw/master/pic/2020/5/22/1590143121653.png)


    this.$manageApi
          .getUserAccountAPI().changeUserState({isDisable: 1， openId: 'open id'})


## 开源地址

[https://github.com/jadepeng/generator-swagger-2-ts](https://github.com/jadepeng/generator-swagger-2-ts)

欢迎star！

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


