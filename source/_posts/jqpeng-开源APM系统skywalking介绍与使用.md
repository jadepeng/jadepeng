---
title: 开源APM系统skywalking介绍与使用
tags: ["SkyWalking","apm","java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-05-28 11:25
---
文章作者:jqpeng
原文链接: [开源APM系统skywalking介绍与使用](https://www.cnblogs.com/xiaoqi/p/skywalking-usage.html)

## 介绍

**SkyWalking** 创建与2015年，提供分布式追踪功能。从5.x开始，项目进化为一个完成功能的[Application Performance Management](https://en.wikipedia.org/wiki/Application_performance_management)系统。  
 他被用于追踪、监控和诊断分布式系统，特别是使用微服务架构，云原生或容积技术。提供以下主要功能：

- 分布式追踪和上下文传输
- 应用、实例、服务性能指标分析
- 根源分析
- 应用拓扑分析
- 应用和服务依赖分析
- 慢服务检测
- 性能优化


### 主要特性

- 多语言探针或类库
    - Java自动探针，追踪和监控程序时，不需要修改源码。
    - 社区提供的其他多语言探针
        - [.NET Core](https://github.com/OpenSkywalking/skywalking-netcore)
        - [Node.js](https://github.com/OpenSkywalking/skywalking-nodejs)
- 多种后端存储： ElasticSearch， H2
- 支持[OpenTracing](http://opentracing.io/)
    - Java自动探针支持和OpenTracing API协同工作
- 轻量级、完善功能的后端聚合和分析
- 现代化Web UI
- 日志集成
- 应用、实例和服务的告警


### 架构
![](https://skywalkingtest.github.io/page-resources/5.0/architecture.png)
### 在线体验

- 北京服务器. [前往](http://49.4.12.44:8080/)
- 香港服务器. [前往](http://159.138.0.181:8080/)


## 安装

### 安装es

新版本的skywalking使用ES作为存储，所以先安装es，注意6.X版本不行，安装5.6.8：


    wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.8.tar.gz
    tar zxvf elasticsearch-5.6.8.tar.gz
    cd elasticsearch-5.6.8/


修改配置文件，主要修改cluster.name，并增加两行配置，  
 vim config/elasticsearch.yml：


    cluster.name: CollectorDBCluster
    
    # ES监听的ip地址
    network.host: 0.0.0.0
    thread_pool.bulk.queue_size: 1000


保存，然后启动es：


    nohup bin/elasticsearch &


### 安装skywalking

先下载编译好的版本并解压：


    wget http://mirrors.hust.edu.cn/apache/incubator/skywalking/5.0.0-beta/apache-skywalking-apm-incubating-5.0.0-beta.tar.gz
    tar zxvf apache-skywalking-apm-incubating-5.0.0-beta.tar.gz 
    cd apache-skywalking-apm-incubating/


然后部署，注意skywalking会使用(8080, 10800, 11800, 12800)端口，因此先排除端口占用情况。

然后运行bin/startup.sh，windows用户为.bat文件。

一切正常的话，访问localhost:8080就能看到页面了。

#### 安装过程问题解决

1. 启动bin/startup.sh后，提示success，但是不能访问，ps 查看并无相关进程，经过检查发现是端口被占用
2. collector 不能正常启动，发现是es问题：
    - es需要使用5.x版本
    - es的集群名称需要和collector的配置文件一致


### java程序使用skywalking探针

1.拷贝apache-skywalking-apm-incubating目录下的agent目录到应用程序位置，探针包含整个目录，请不要改变目录结构  
 2.java程序启动时，增加JVM启动参数，-javaagent:/path/to/agent/skywalking-agent.jar。参数值为skywalking-agent.jar的绝对路径

在IDEA里调试程序怎么办？

![enter description here](https://raw.githubusercontent.com/jadepeng/blogpic/master/pic/2018/1527477582714.jpg "IDEA 配置")

增加VM参数即可。

agent探针配置，简单修改下agent.application\_code即可


    # 当前的应用编码，最终会显示在webui上。
    # 建议一个应用的多个实例，使用有相同的application_code。请使用英文
    agent.application_code=Your_ApplicationName
    
    # 每三秒采样的Trace数量
    # 默认为负数，代表在保证不超过内存Buffer区的前提下，采集所有的Trace
    # agent.sample_n_per_3_secs=-1
    
    # 设置需要忽略的请求地址
    # 默认配置如下
    # agent.ignore_suffix=.jpg,.jpeg,.js,.css,.png,.bmp,.gif,.ico,.mp3,.mp4,.html,.svg
    
    # 探针调试开关，如果设置为true，探针会将所有操作字节码的类输出到/debugging目录下
    # skywalking团队可能在调试，需要此文件
    # agent.is_open_debugging_class = true
    
    # 对应Collector的config/application.yml配置文件中 agent_server/jetty/port 配置内容
    # 例如：
    # 单节点配置：SERVERS="127.0.0.1:8080" 
    # 集群配置：SERVERS="10.2.45.126:8080,10.2.45.127:7600" 
    collector.servers=127.0.0.1:10800
    
    # 日志文件名称前缀
    logging.file_name=skywalking-agent.log
    
    # 日志文件最大大小
    # 如果超过此大小，则会生成新文件。
    # 默认为300M
    logging.max_file_size=314572800
    
    # 日志级别，默认为DEBUG。
    logging.level=DEBUG


一切正常的话，稍后就可以在skywalking ui看到了。

![enter description here](https://raw.githubusercontent.com/jadepeng/blogpic/master/pic/2018/1527477724637.jpg "SW UI")

可以看到累出了slow service等信息，更多的细节慢慢挖掘吧。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


