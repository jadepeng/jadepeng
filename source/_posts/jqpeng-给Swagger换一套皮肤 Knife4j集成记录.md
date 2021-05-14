---
title: 给Swagger换一套皮肤 Knife4j集成记录
tags: ["Knife4j","swagger","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-11-02 11:19
---
文章作者:jqpeng
原文链接: [给Swagger换一套皮肤 Knife4j集成记录](https://www.cnblogs.com/xiaoqi/p/Knife4j.html)

Swagger有一套经典的UI，但是并不是很好用，之前有看到Knife4j，界面美观、功能完善，因此尝试集成。

demo参考示例地址：[knife4j-spring-boot-demo](https://gitee.com/xiaoym/swagger-bootstrap-ui-demo/tree/master/knife4j-spring-boot-demo)

Knife4j前身是swagger-bootstrap-ui,是一个为Swagger接口文档赋能的工具

根据官方文档，集成非常方便。

## maven引用

第一步,是在项目的`pom.xml`文件中引入`knife4j`的依赖,如下：


    <dependencies>
        <dependency>
            <groupId>com.github.xiaoymin</groupId>
            <artifactId>knife4j-spring-boot-starter</artifactId>
            <version>2.0.6</version>
        </dependency>
    </dependencies>
    


当前最新版是2.0.6

如果你想使用bom的方式引入,请参考[Maven Bom方式引用](https://doc.xiaominfo.com/knife4j/mavenbom.html)

## 创建Swagger配置文件

新建Swagger的配置文件`SwaggerConfiguration.java`文件,创建springfox提供的Docket分组对象,代码如下：


    @Configuration
    @EnableSwagger2
    @EnableKnife4j
    @Import(BeanValidatorPluginsConfiguration.class)
    public class SwaggerConfiguration {
    
        @Bean(value = "defaultApi2")
        public Docket defaultApi2() {
            Docket docket=new Docket(DocumentationType.SWAGGER_2)
                    .apiInfo(apiInfo())
                    //分组名称
                    .groupName("2.X版本")
                    .select()
                    //这里指定Controller扫描包路径
                    .apis(RequestHandlerSelectors.basePackage("com.swagger.bootstrap.ui.demo.new2"))
                    .paths(PathSelectors.any())
                    .build();
            return docket;
        }
    
    }
    


以上有两个注解需要特别说明，如下表：


| 注解 | 说明 |
| --- | --- |
| `@EnableSwagger2` | 该注解是Springfox-swagger框架提供的使用Swagger注解，该注解必须加 |
| `@EnableKnife4j` | 该注解是`knife4j`提供的增强注解,Ui提供了例如动态参数、参数过滤、接口排序等增强功能,如果你想使用这些增强功能就必须加该注解，否则可以不用加 |


## spring-security 免认证

将`/**/doc.html/**` 加入：


        private String[] getSwaggerUrl() {
            List<String> urls = new ArrayList<String>();
            urls.add("/**/swagger-resources/**");
            urls.add("/**/webjars/**");
            urls.add("/**/doc.html/**");
            urls.add("/**/v2/**");
            urls.add("/**/swagger-ui.html/**");
            return urls.toArray(new String[urls.size()]);
        }http.authorizeRequests() .antMatchers(getSwaggerUrl()).permitAll()


## 测试访问

在浏览器输入地址：`http://host:port/doc.html`

可以设置全局参数：

![全局参数](https://gitee.com/jadepeng/pic/raw/master/pic/2020/11/2/1604286718931.png)

支持在线调试

![在线调试](https://gitee.com/jadepeng/pic/raw/master/pic/2020/11/2/1604286643402.png)

离线文档支持导出md、pdf等

![导出文档](https://gitee.com/jadepeng/pic/raw/master/pic/2020/11/2/1604286760474.png)

## 最后

前端如何更优雅的调用api呢？参考：

[Vue 使用typescript， 优雅的调用swagger API](https://www.cnblogs.com/xiaoqi/p/generator-swagger-2-ts-2.html)

后面有空，可以将这个集成到knife4j

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


