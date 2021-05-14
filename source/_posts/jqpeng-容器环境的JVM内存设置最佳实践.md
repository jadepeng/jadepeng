---
title: 容器环境的JVM内存设置最佳实践
tags: ["docker","jvm","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-01-18 16:54
---
文章作者:jqpeng
原文链接: [容器环境的JVM内存设置最佳实践](https://www.cnblogs.com/xiaoqi/p/container-jvm.html)

Docker和K8S的兴起，很多服务已经运行在容器环境，对于java程序，JVM设置是一个重要的环节。这里总结下我们项目里的最佳实践。

## Java Heap基础知识

默认情况下，jvm自动分配的heap大小取决于机器配置，比如我们到一台64G内存服务器：


    java -XX:+PrintFlagsFinal -version | grep -Ei "maxheapsize|maxram"
        uintx DefaultMaxRAMFraction                     = 4                                   {product}
        uintx MaxHeapSize                              := 16875782144                         {product}
     uint64_t MaxRAM                                    = 137438953472                        {pd product}
        uintx MaxRAMFraction                            = 4                                   {product}
       double MaxRAMPercentage                          = 25.000000                           {product}
    java version "1.8.0_192"
    Java(TM) SE Runtime Environment (build 1.8.0_192-b12)
    Java HotSpot(TM) 64-Bit Server VM (build 25.192-b12, mixed mode)


可以看到，JVM 分配的最大MaxHeapSize为 16G，计算公式如下：


    MaxHeapSize = MaxRAM * 1 / MaxRAMFraction


MaxRAMFraction 默认是4，意味着，每个jvm最多使用25%的机器内存。

但是需要注意的是，JVM实际使用的内存会比heap内存大：


    JVM内存  = heap 内存 + 线程stack内存 (XSS) * 线程数 + 启动开销（constant overhead）


默认的XSS通常在256KB到1MB，也就是说每个线程会分配最少256K额外的内存，constant overhead是JVM分配的其他内存。

我们可以通过`-Xmx` 指定最大堆大小。


    java -XX:+PrintFlagsFinal -Xmx1g -version | grep -Ei "maxheapsize|maxram"
        uintx DefaultMaxRAMFraction                     = 4                                   {product}
        uintx MaxHeapSize                              := 1073741824                          {product}
     uint64_t MaxRAM                                    = 137438953472                        {pd product}
        uintx MaxRAMFraction                            = 4                                   {product}
       double MaxRAMPercentage                          = 25.000000                           {product}
    java version "1.8.0_192"
    Java(TM) SE Runtime Environment (build 1.8.0_192-b12)
    Java HotSpot(TM) 64-Bit Server VM (build 25.192-b12, mixed mode)


此外，还可以使用`XX:MaxRAM`来指定。


    java -XX:+PrintFlagsFinal -XX:MaxRAM=1g -version | grep -Ei 


但是指定-Xmx或者MaxRAM需要了解机器的内存，更好的方式是设置`MaxRAMFraction`，以下是不同的Fraction对应的可用内存比例：


    +----------------+-------------------+
    | MaxRAMFraction | % of RAM for heap |
    |----------------+-------------------|
    |              1 |              100% |
    |              2 |               50% |
    |              3 |               33% |
    |              4 |               25% |
    +----------------+-------------------+


## 容器环境的Java Heap

容器环境，由于java获取不到容器的内存限制，只能获取到服务器的配置：


    $ docker run --rm alpine free -m
                 total     used     free   shared  buffers   cached
    Mem:          1998     1565      432        0        8     1244
    $ docker run --rm -m 256m alpine free -m
                 total     used     free   shared  buffers   cached
    Mem:          1998     1552      445        1        8     1244


这样容易引起不必要问题，例如限制容器使用100M内存，但是jvm根据服务器配置来分配初始化内存，导致java进程超过容器限制被kill掉。为了解决这个问题，可以设置-Xmx或者MaxRAM来解决，但就想第一部分描述的一样，这样太不优雅了！

为了解决这个问题，Java 10 引入了 +`UseContainerSupport`（默认情况下启用），通过这个特性，可以使得JVM在容器环境分配合理的堆内存。 并且，在JDK8U191版本之后，这个功能引入到了JDK 8，而JDK 8是广为使用的JDK版本。

### UseContainerSupport

-XX:+UseContainerSupport允许JVM 从主机读取cgroup限制，例如可用的CPU和RAM，并进行相应的配置。这样当容器超过内存限制时，会抛出OOM异常，而不是杀死容器。  
 该特性在Java 8u191 +，10及更高版本上可用。

注意，在191版本后，-XX:{Min|Max}RAMFraction 被弃用，引入了`-XX:MaxRAMPercentage`，其值介于0.0到100.0之间，默认值为25.0。

## 最佳实践

拉取最新的openjdk:8-jre-alpine作为底包，截止这篇博客，最新的版本是212，&gt;191


    docker run -it --rm openjdk:8-jre-alpine java -version
    openjdk version "1.8.0_212"
    OpenJDK Runtime Environment (IcedTea 3.12.0) (Alpine 8.212.04-r0)
    OpenJDK 64-Bit Server VM (build 25.212-b04, mixed mode)


我们构建一个基础镜像，dockerfile如下：


    FROM openjdk:8-jre-alpine
    MAINTAINER jadepeng
    
    RUN echo "http://mirrors.aliyun.com/alpine/v3.6/main" > /etc/apk/repositories \
        && echo "http://mirrors.aliyun.com/alpine/v3.6/community" >> /etc/apk/repositories \
        && apk update upgrade \
        && apk add --no-cache procps unzip curl bash tzdata \
        && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
        && echo "Asia/Shanghai" > /etc/timezone
    
    RUN apk add --update ttf-dejavu && rm -rf /var/cache/apk/*


在应用的启动参数，设置 -XX:+UseContainerSupport，设置-XX:MaxRAMPercentage=75.0，这样为其他进程（debug、监控）留下足够的内存空间，又不会太浪费RAM。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


