---
title: 阿里开源JDK dragonwell8在容器环境使用
tags: ["dragonwell8","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-09-29 15:30
---
文章作者:jqpeng
原文链接: [阿里开源JDK dragonwell8在容器环境使用](https://www.cnblogs.com/xiaoqi/p/dragonwell8-docker.html)

Alibaba Dragonwell 是阿里巴巴的Open JDK 发行版，提供长期支持。 阿里宣传称在阿里生产环境实现了应用。Alibaba Dragonwell兼容 Java SE 标准，因此可以方便切换。

## 下载

可以到https://github.com/alibaba/dragonwell8/releases?spm=5176.cndragonwell.0.0.4c5a7568DpYPsp 下载，当前的最新版是8.4.4 GA

![ali jdk8](https://gitee.com/jadepeng/pic/raw/master/pic/2020/9/29/1601363568094.png)

阿里也提供了镜像下载地址，可以加速下载：


> 8.4.4GA



| File name | 中国大陆 | United States |
| --- | --- | --- |
| Alibaba\_Dragonwell\_8.4.4-GA\_Linux\_x64.tar.gz | [download](https://dragonwell.oss-cn-shanghai.aliyuncs.com/8/8.4.4-GA/Alibaba_Dragonwell_8.4.4-GA_Linux_x64.tar.gz) | [download](https://github.com/alibaba/dragonwell8/releases/download/dragonwell-8.4.4_jdk8u262-ga/Alibaba_Dragonwell_8.4.4-GA_Linux_x64.tar.gz) |
| Alibaba\_Dragonwell\_8.4.4-GA\_source.tar.gz | [download](https://dragonwell.oss-cn-shanghai.aliyuncs.com/8/8.4.4-GA/Alibaba_Dragonwell_8.4.4-GA_source.tar.gz) | [download](https://github.com/alibaba/dragonwell8/releases/download/dragonwell-8.4.4_jdk8u262-ga/Alibaba_Dragonwell_8.4.4-GA_source.tar.gz) |
| Alibaba\_Dragonwell\_8.4.4-Experimental\_Windows\_x64.tar.gz | [download](https://dragonwell.oss-cn-shanghai.aliyuncs.com/8/8.4.4-GA/Alibaba_Dragonwell_8.4.4-Experimental_Windows_x64.tar.gz) | [download](https://github.com/alibaba/dragonwell8/releases/download/dragonwell-8.4.4_jdk8u262-ga/Alibaba_Dragonwell_8.4.4-Experimental_Windows_x64.tar.gz) |
| java8-api-8.4.4-javadoc.jar | [download](https://dragonwell.oss-cn-shanghai.aliyuncs.com/8/8.4.4-GA/java8-api-8.4.4-javadoc.jar) | [download](https://github.com/alibaba/dragonwell8/releases/download/dragonwell-8.4.4_jdk8u262-ga/java8-api-8.4.4-javadoc.jar) |
| java8-api-8.4.4-sources.jar | [download](https://dragonwell.oss-cn-shanghai.aliyuncs.com/8/8.4.4-GA/java8-api-8.4.4-sources.jar) | [download](https://github.com/alibaba/dragonwell8/releases/download/dragonwell-8.4.4_jdk8u262-ga/java8-api-8.4.4-sources.jar) |
| java8-api-8.4.4.jar | [download](https://dragonwell.oss-cn-shanghai.aliyuncs.com/8/8.4.4-GA/java8-api-8.4.4.jar) | [download](https://github.com/alibaba/dragonwell8/releases/download/dragonwell-8.4.4_jdk8u262-ga/java8-api-8.4.4.jar) |



> [SHA256 checksum](https://github.com/alibaba/dragonwell8/releases/tag/dragonwell-8.4.4_jdk8u262-ga)


## 使用


    wget https://dragonwell.oss-cn-shanghai.aliyuncs.com/8/8.4.4-GA/Alibaba_Dragonwell_8.4.4-GA_Linux_x64.tar.gz
    tar zxvf Alibaba_Dragonwell_8.4.4-GA_Linux_x64.tar.gz 
    cd jdk8u262-b10/
     bin/java -version


输出：


    openjdk version "1.8.0_262"
    OpenJDK Runtime Environment (Alibaba Dragonwell 8.4.4) (build 1.8.0_262-b10)
    OpenJDK 64-Bit Server VM (Alibaba Dragonwell 8.4.4) (build 25.262-b10, mixed mode)


## 容器环境使用

我们需要准备一个基础镜像包，我们以openjdk:8-jdk-slim为基础包，把jdk替换为dragonwell8


     FROM openjdk:8-jdk-slim
     MAINTAINER jadepeng
     
     RUN sed -i 's#http://deb.debian.org#https://mirrors.163.com#g' /etc/apt/sources.list \
         && apt-get update\
         && apt-get install -y procps unzip curl bash tzdata ttf-dejavu \
         && rm -rf /var/cache/apt/ \
         && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
         && echo "Asia/Shanghai" > /etc/timezone
     
     RUN rm -r -f /usr/local/openjdk-8
     
     ADD jdk8u262-b10  /usr/local/openjdk-8


构建：


    docker build -t dragonwell8:8.4.4 .


java应用就可以用`dragonwell8:8.4.4 `为底包执行了。

## dragonwell8 协程使用

dragonwell8的一大特色就是协程（Coroutine ）

官方有一个demo：

以一个pingpong 1,000,000次的程序为例，这是一个需要阻塞，切换密集型的应用。


               o      .   _______ _______
                  \_ 0     /______//______/|   @_o
                    /\_,  /______//______/     /\
                   | \    |      ||      |     / |
    
    static final ExecutorService THREAD_POOL = Executors.newCachedThreadPool();
    
    public static void main(String[] args) throws Exception {
        BlockingQueue<Byte> q1 = new LinkedBlockingQueue<>(), q2 = new LinkedBlockingQueue<>();
        THREAD_POOL.submit(() -> pingpong(q2, q1)); // thread A
        Future<?> f = THREAD_POOL.submit(() -> pingpong(q1, q2)); // thread B
        q1.put((byte) 1);
        System.out.println(f.get() + " ms");
    }
    
    private static long pingpong(BlockingQueue<Byte> in, BlockingQueue<Byte> out) throws Exception {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 1_000_000; i++) out.put(in.take());
        return System.currentTimeMillis() - start;
    }


运行，查看执行时间:


    $java PingPong
    13212 ms
    
    // 开启Wisp2
    $java -XX:+UseWisp2 -XX:ActiveProcessorCount=1 PingPong
    882 ms


开启Wisp2后整体运行效率提升了近十多倍，只需要启动时增加参数`-XX:+UseWisp2`即可。不用修改一行代码，即可享受协程带来的优势。

随后可以通过jstack观察到起来的线程都以协程的方式在运行了。


     - Coroutine [0x7f6d6d60fb20] "Thread-978" #1076 active=1 steal=0 steal_fail=0 preempt=0 park=0/-1
            at java.dyn.CoroutineSupport.unsafeSymmetricYieldTo(CoroutineSupport.java:138)
    --
     - Coroutine [0x7f6d6d60f880] "Thread-912" #1009 active=1 steal=0 steal_fail=0 preempt=0 park=0/-1
            at java.dyn.CoroutineSupport.unsafeSymmetricYieldTo(CoroutineSupport.java:138)
    
    ...
    


可以看到最上方的frame上的方法是协程切换，因为线程调用了sleep，yield出了执行权。

开启Wisp2后，Java线程不再简单地映射到内核级线程，而是对应到一个协程，JVM在少量内核线上调度大量协程执行，以减少内核的调度开销，提高web服务器的性能。

在阿里的生产环境，性能指标：

- 在复杂的业务应用中(tomcat + 大量基于netty的中间件)我们获得了大约10%的性能提升。
- 在中间件应用(数据库代理，MQ)中我们获得大约20%的性能提升。


更多Wisp的设计实现与API相关信息，请参考[Wisp文档](https://github.com/alibaba/dragonwell8/wiki/Wisp%E6%96%87%E6%A1%A3)。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


