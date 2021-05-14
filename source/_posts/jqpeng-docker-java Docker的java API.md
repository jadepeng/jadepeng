---
title: docker-java Docker的java API
tags: ["docker","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-01-22 09:28
---
文章作者:jqpeng
原文链接: [docker-java Docker的java API](https://www.cnblogs.com/xiaoqi/p/docker-java.html)

## docker-java

docker-java 是 Docker的 Java 版本API  [Docker](http://docs.docker.io/ "Docker")

**当前的实现基于 Jersey 2.x 因此 classpath 不兼容老版本的 Jersey 1.x !**

开发者论坛 [docker-java](https://groups.google.com/forum/?#!forum/docker-java-dev "docker-java")

[Changelog](https://github.com/docker-java/docker-java/blob/master/CHANGELOG.md)  
  
[Wiki](https://github.com/docker-java/docker-java/wiki)

## 版本支持

Supports a subset of the Docker Remote API [v1.37](https://docs.docker.com/engine/api/v1.37/), Docker Server version since 1.12.6


    <dependency>
          <groupId>com.github.docker-java</groupId>
          <artifactId>docker-java</artifactId>
          <!-- use latest version https://github.com/docker-java/docker-java/releases -->
          <version>3.X.Y</version>
    </dependency>


当前最新的版本是3.1.0，可以点击[这里](https://github.com/docker-java/docker-java/releases)查看最新版本。


        <dependency>
            <groupId>com.github.docker-java</groupId>
            <artifactId>docker-java</artifactId>
            <version>3.1.0</version>
        </dependency>


## wiki文档

For code examples, please look at the [Wiki](https://github.com/docker-java/docker-java/wiki) or [Test cases](https://github.com/docker-java/docker-java/tree/master/src/test/java/com/github/dockerjava/core/command "Test cases")

## 配置Docker环境

系统的可配置项及默认值如下:

- `DOCKER_HOST` The Docker Host URL, e.g. `tcp://localhost:2376` or `unix:///var/run/docker.sock`
- `DOCKER_TLS_VERIFY` enable/disable TLS verification (switch between `http` and `https` protocol)
- `DOCKER_CERT_PATH` Path to the certificates needed for TLS verification
- `DOCKER_CONFIG` Path for additional docker configuration files (like `.dockercfg`)
- `api.version` The API version, e.g. `1.23`.
- `registry.url` Your registry's address.
- `registry.username` Your registry username (required to push containers).
- `registry.password` Your registry password.
- `registry.email` Your registry email.


There are three ways to configure, in descending order of precedence:

### 编程方式配置:


    DockerClientConfig config = DefaultDockerClientConfig.createDefaultConfigBuilder()
        .withDockerHost("tcp://my-docker-host.tld:2376")
        .withDockerTlsVerify(true)
        .withDockerCertPath("/home/user/.docker/certs")
        .withDockerConfig("/home/user/.docker")
        .withApiVersion("1.30") // optional
        .withRegistryUrl("https://index.docker.io/v1/")
        .withRegistryUsername("dockeruser")
        .withRegistryPassword("ilovedocker")
        .withRegistryEmail("dockeruser@github.com")
        .build();
    DockerClient docker = DockerClientBuilder.getInstance(config).build();


### 通过Properties (docker-java.properties)


    DOCKER_HOST=tcp://localhost:2376
    DOCKER_TLS_VERIFY=1
    DOCKER_CERT_PATH=/home/user/.docker/certs
    DOCKER_CONFIG=/home/user/.docker
    api.version=1.23
    registry.url=https://index.docker.io/v1/
    registry.username=dockeruser
    registry.password=ilovedocker
    registry.email=dockeruser@github.com


### 通过System Properties:


    java -DDOCKER_HOST=tcp://localhost:2375 -Dregistry.username=dockeruser pkg.Main


### 通过 Environment


    export DOCKER_HOST=tcp://localhost:2376
    export DOCKER_TLS_VERIFY=1
    export DOCKER_CERT_PATH=/home/user/.docker/certs
    export DOCKER_CONFIG=/home/user/.docker


## 测试

我们来简单测试下：


        DockerClient dockerClient = createClient();
    
        // docker info
        Info info = dockerClient.infoCmd().exec();
        System.out.print(info);
    
        // 搜索镜像
        List<SearchItem> dockerSearch = dockerClient.searchImagesCmd("busybox").exec();
        System.out.println("Search returned" + dockerSearch.toString());
    
        // pull
    
        dockerClient.pullImageCmd("busybox:latest").exec(new ResultCallback<PullResponseItem>() {
            public void onStart(Closeable closeable) {
    
            }
    
            public void onNext(PullResponseItem object) {
                System.out.println(object.getStatus());
            }
    
            public void onError(Throwable throwable) {
                throwable.printStackTrace();
            }
    
            public void onComplete() {
                System.out.println("pull finished");
            }
    
            public void close() throws IOException {
    
            }
        });
    
    
    
    
        // 创建container 并测试
    
        // create
        CreateContainerResponse container = dockerClient.createContainerCmd("busybox")
                .withCmd("/bin/bash")
                .exec();
        // start
        dockerClient.startContainerCmd(container.getId()).exec();
        dockerClient.stopContainerCmd(container.getId()).exec();
        dockerClient.removeContainerCmd(container.getId()).exec();
    
        EventsResultCallback callback = new EventsResultCallback() {
            @Override
            public void onNext(Event event) {
                System.out.println("Event: " + event);
                super.onNext(event);
            }
        };
    
        dockerClient.eventsCmd().exec(callback).awaitCompletion().close();


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


