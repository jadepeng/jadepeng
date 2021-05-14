---
title: 多语言业务错误日志收集监控工具Sentry 安装与使用
tags: ["Sentry","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-05-24 16:04
---
文章作者:jqpeng
原文链接: [多语言业务错误日志收集监控工具Sentry 安装与使用](https://www.cnblogs.com/xiaoqi/p/sentry.html)

Sentry 是一个实时事件日志记录和汇集的平台。其专注于错误监控以及提取一切事后处理所需信息而不依赖于麻烦的用户反馈。

Sentry是一个日志平台, 它分为客户端和服务端，客户端(目前客户端有Python, PHP,C#, Ruby等多种语言)就嵌入在你的应用程序中间，程序出现异常就向服务端发送消息，服务端将消息记录到数据库中并提供一个web节目方便查看。Sentry由python编写，源码开放，性能卓越，易于扩展，目前著名的用户有Disqus, Path, mozilla, Pinterest等。

## 通过docker安装

官方提供了 On-Premise 安装方式


    git clone https://github.com/getsentry/onpremise
    cd onpremise


然后创建volume，持久化存储


    docker volume create --name=sentry-data && docker volume create --name=sentry-postgres


创建环境变量配置文件：


    cp -n .env.example .env - create env config file


docker-compose build镜像，这一步会拉取官方镜像和redis等依赖，耐心等待	：


    docker-compose build - Build and tag the Docker services


创建secret-key，执行后得到一个key，添加到.env中的SENTRY\_SECRET\_KEY


    docker-compose run --rm web config generate-secret-key 


创建DB和初始化用户,等待创建数据库和


    docker-compose run --rm web upgrade
    Created internal Sentry project (slug=internal, id=1)
    
    Would you like to create a user account now? [Y/n]: y
    Email: 
    Password: 
    Repeat for confirmation: 
    Should this user be a superuser? [y/N]: y
    Added to organization: sentry
    Running migrations for sentry.nodestore:
     - Migrating forwards to 0001_initial.


最后-d启动，启动成功后可以访问http://server-ip:9000


    docker-compose up -d 


访问：

![sentry](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-24/1558680516195.png)

## 创建项目并集成

先创建一个project，支持多种语言，我们创建一个java的

![create project](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-24/1558680561117.png)

### DSN（Client Keys）

DSN格式为：

DSN="https://public:private@host:port/project\_id"

可以在项目的setting-Client Keys（DSN）里查看。

![DSK](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-24/1558684229783.png)

然后根据格式拼成DSN即可。

### java项目集成

有多种集成方式：

#### 基本集成方式

添加maven引用：


    	<dependency>	<groupId>io.sentry</groupId>	<artifactId>sentry</artifactId>	<version>1.7.16</version></dependency>


然后在全局异常处理里集成：


     String dsn = "http://<SENTRY_PUBLIC_KEY>:<SENTRY_PRIVATE_KEY>@host:port/<PROJECT_ID>";
     Sentry.init(dsn);


在有异常的地方：


          try {
                unsafeMethod();
            } catch (Exception e) {
                // This sends an exception event to Sentry using the statically stored instance
                // that was created in the ``main`` method.
                Sentry.capture(e);
            }


#### logback集成

logback配置：


    <configuration>
    
      <appender name="Console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
          <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
      </appender>
    
      <appender name="Sentry" class="io.sentry.logback.SentryAppender">
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
          <level>WARN</level>
        </filter>
      </appender>
    
      <root level="DEBUG">
        <appender-ref ref="Console" />
        <appender-ref ref="Sentry"/>
      </root>
    
    </configuration>


#### spring-boot集成

先添加引用：


    		<dependency>		<groupId>io.sentry</groupId>		<artifactId>sentry-spring</artifactId>		<version>1.7.16</version>	</dependency>


在Application里集成：


    import org.springframework.boot.SpringApplication;
    import org.springframework.boot.autoconfigure.EnableAutoConfiguration;
    import org.springframework.boot.autoconfigure.SpringBootApplication;
    import org.springframework.boot.web.servlet.ServletContextInitializer;
    import org.springframework.context.annotation.Bean;
    import org.springframework.stereotype.Controller;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.ResponseBody;
    import org.springframework.web.servlet.HandlerExceptionResolver;
    
    @Controller
    @EnableAutoConfiguration
    @SpringBootApplication
    public class Application {
        /*
        Register a HandlerExceptionResolver that sends all exceptions to Sentry
        and then defers all handling to the other HandlerExceptionResolvers.
        You should only register this is you are not using a logging integration,
        otherwise you may double report exceptions.
         */
        @Bean
        public HandlerExceptionResolver sentryExceptionResolver() {
            return new io.sentry.spring.SentryExceptionResolver();
        }
    
        /*
        Register a ServletContextInitializer that installs the SentryServletRequestListener
        so that Sentry events contain HTTP request information.
        This should only be necessary in Spring Boot applications. "Classic" Spring
        should automatically load the `io.sentry.servlet.SentryServletContainerInitializer`.
         */
        @Bean
        public ServletContextInitializer sentryServletContextInitializer() {
            return new io.sentry.spring.SentryServletContextInitializer();
        }
    
        @RequestMapping("/")
        @ResponseBody
        String home() {
            int x = 1 / 0;
    
            return "Hello World!";
        }
    
        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }


### 前端项目集成

前端集成更简单，引用sentry的js，然后init即可


    <script src="https://browser.sentry-cdn.com/5.1.0/bundle.min.js" crossorigin="anonymous"></script>
    Sentry.init({ dsn: 'dsn' });


* * *


> 作者：Jadepeng 出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi) 您的支持是对博主最大的鼓励，感谢您的认真阅读。 本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


