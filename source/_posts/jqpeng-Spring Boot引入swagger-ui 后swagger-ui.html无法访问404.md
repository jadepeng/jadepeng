---
title: Spring Boot引入swagger-ui 后swagger-ui.html无法访问404
tags: ["swagger","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-08-22 22:05
---
文章作者:jqpeng
原文链接: [Spring Boot引入swagger-ui 后swagger-ui.html无法访问404](https://www.cnblogs.com/xiaoqi/p/swagger-ui-404.html)

最近给graphserver增加swagger，记录下过程与问题解决。

Swagger 是一个规范和完整的框架，用于生成、描述、调用和可视化 RESTful 风格的 Web 服务，后端集成下Swagger，然后就可以提供一个在线文档地址给前端同学。

## 引入 Swagger

pom中加入相关配置：


            <dependency>
                <groupId>io.springfox</groupId>
                <artifactId>springfox-swagger2</artifactId>
                <version>2.9.2</version>
            </dependency>
            <dependency>
                <groupId>io.springfox</groupId>
                <artifactId>springfox-swagger-ui</artifactId>
                <version>2.9.2</version>
            </dependency>


增加`Swagger2Config`, 添加`@EnableSwagger2`,可以通过定义Docket bean实现自定义。


    @Configuration
    @EnableSwagger2
    @Profile("swagger")
    @ComponentScan("xxx.controller")
    public class Swagger2Config {
    
        @Bean
        public Docket createRestApi() {
            return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .enable(true)
                .select()
                .apis(RequestHandlerSelectors.basePackage("xxx.controller"))
                .paths(PathSelectors.any())
                .build();
        }
    
        private ApiInfo apiInfo() {
            return new ApiInfoBuilder()
                .title("XXX Rest Server")
                .description("XXXRest接口")
                .contact(new Contact("contract", "url", "email"))
                .version("1.0")
                .build();
        }
    }


## swagger-ui.html 404问题

项目中有web配置，因此怀疑是这些配置影响了，搜索下发现这位仁兄有类似经历：`https://www.cnblogs.com/pangguoming/p/10551895.html`

于是在WebMvcConfig 配置中，override `addResourceHandlers`


    
    @Configuration
    public class WebMvcConfig implements WebMvcConfigurer {
    
        @Override
        public void addResourceHandlers(ResourceHandlerRegistry registry) {
            registry.addResourceHandler("/**").addResourceLocations("classpath:/static/");
            registry.addResourceHandler("swagger-ui.html")
                .addResourceLocations("classpath:/META-INF/resources/");
            registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/");
        }


搞定收工。

## 延伸阅读

server端有了swagger，前端如何更优先的调用？

参见：[Vue 使用typescript， 优雅的调用swagger API](https://www.cnblogs.com/xiaoqi/p/generator-swagger-2-ts-2.html),笔者提供了一个开源npm库，可以为前端生成调用axios调用代码。

## 参考

- [https://www.cnblogs.com/pangguoming/p/10551895.html](https://www.cnblogs.com/pangguoming/p/10551895.html)


