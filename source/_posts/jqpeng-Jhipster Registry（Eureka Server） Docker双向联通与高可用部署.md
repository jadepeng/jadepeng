---
title: Jhipster Registry（Eureka Server） Docker双向联通与高可用部署
tags: ["Jhipster","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-07-05 09:56
---
文章作者:jqpeng
原文链接: [Jhipster Registry（Eureka Server） Docker双向联通与高可用部署](https://www.cnblogs.com/xiaoqi/p/9266722.html)

使用Compose来编排这个Eureka Server集群：

## peer1配置：


    server:
        port: 8761
    
    eureka:
        instance:
            hostname: eureka-peer-1
        server:
            # see discussion about enable-self-preservation:
            # https://github.com/jhipster/generator-jhipster/issues/3654
            enable-self-preservation: false
            registry-sync-retry-wait-ms: 500
            a-sgcache-expiry-timeout-ms: 60000
            eviction-interval-timer-in-ms: 30000
            peer-eureka-nodes-update-interval-ms: 30000
            renewal-threshold-update-interval-ms: 15000
        client:
            fetch-registry: true
            register-with-eureka: true
            service-url:
                defaultZone: http://admin:${spring.security.user.password:admin}@eureka-peer-2:8762/eureka/


## peer2配置：


    server:
        port: 8762
    
    eureka:
        instance:
            hostname: eureka-peer-2
        server:
            # see discussion about enable-self-preservation:
            # https://github.com/jhipster/generator-jhipster/issues/3654
            enable-self-preservation: false
            registry-sync-retry-wait-ms: 500
            a-sgcache-expiry-timeout-ms: 60000
            eviction-interval-timer-in-ms: 30000
            peer-eureka-nodes-update-interval-ms: 30000
            renewal-threshold-update-interval-ms: 15000
        client:
            fetch-registry: true
            register-with-eureka: true
            service-url:
                defaultZone: http://admin:${spring.security.user.password:admin}@eureka-peer-1:8761/eureka/


## 构建Image

使用官方的DockerFile：


    FROM openjdk:8-jre-alpine
    
    ENV SPRING_OUTPUT_ANSI_ENABLED=ALWAYS \
        JAVA_OPTS="" \
        JHIPSTER_SLEEP=0
    
    VOLUME /tmp
    EXPOSE 8761
    CMD echo "The application will start in ${JHIPSTER_SLEEP}s..." && \
        sleep ${JHIPSTER_SLEEP} && \
        java ${JAVA_OPTS} -Djava.security.egd=file:/dev/./urandom -jar /app.war
    
    # add directly the war
    ADD *.war /app.war


构建Image并push到registry，这里是192.168.86.8:5000/registry-dev

## 编写compose文件：


    version: "3" 
    services:
      eureka-peer-1 :
        image: 192.168.86.8:5000/registry-dev:latest
        links:
          - eureka-peer-2
        ports:
          - "8761:8761"
        environment:
          spring.profiles.active: oauth2,peer1,swagger
        entrypoint:
          - java
          - -Dspring.profiles.active=oauth2,peer1,swagger
          - -Djava.security.egd=file:/dev/./urandom
          - -jar
          - /app.war
      eureka-peer-2:
        image: 192.168.86.8:5000/registry-dev:latest
        links:
          - eureka-peer-1
        expose:
          - "8762"
        ports:
          - "8762:8762"
        environment:
          spring.profiles.active: oauth2,peer2,swagger
        entrypoint:
          - java
          - -Dspring.profiles.active=oauth2,peer2,swagger
          - -Djava.security.egd=file:/dev/./urandom
          - -jar
          - /app.war


启动即可。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


