---
title: Docker Client （another java docker client api）
tags: ["docker","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-01-22 15:30
---
文章作者:jqpeng
原文链接: [Docker Client （another java docker client api）](https://www.cnblogs.com/xiaoqi/p/java-Docker-Client.html)

前一篇提到了docker-java，这里介绍另一个docker client 库，[Docker Client](https://github.com/spotify/docker-client)

## 版本兼容

兼容17.03.1~ce - 17.12.1~ce (点 [here][1]查看).

## 下载jar包

点击 [via Maven][maven-search]搜索和下载最新的jar包.

pom.xml配置如下：


    <dependency>
      <groupId>com.spotify</groupId>
      <artifactId>docker-client</artifactId>
      <version>LATEST-VERSION</version>
    </dependency>


当前最新的是8.15.0


    <dependency><groupId>com.spotify</groupId><artifactId>docker-client</artifactId><version>8.15.0</version>
    </dependency>


## 使用举例


    // Create a client based on DOCKER_HOST and DOCKER_CERT_PATH env vars
    final DockerClient docker = DefaultDockerClient.fromEnv().build();
    
    // Pull an image
    docker.pull("busybox");
    
    // Bind container ports to host ports
    final String[] ports = {"80", "22"};
    final Map<String, List<PortBinding>> portBindings = new HashMap<>();
    for (String port : ports) {
        List<PortBinding> hostPorts = new ArrayList<>();
        hostPorts.add(PortBinding.of("0.0.0.0", port));
        portBindings.put(port, hostPorts);
    }
    
    // Bind container port 443 to an automatically allocated available host port.
    List<PortBinding> randomPort = new ArrayList<>();
    randomPort.add(PortBinding.randomPort("0.0.0.0"));
    portBindings.put("443", randomPort);
    
    final HostConfig hostConfig = HostConfig.builder().portBindings(portBindings).build();
    
    // Create container with exposed ports
    final ContainerConfig containerConfig = ContainerConfig.builder()
        .hostConfig(hostConfig)
        .image("busybox").exposedPorts(ports)
        .cmd("sh", "-c", "while :; do sleep 1; done")
        .build();
    
    final ContainerCreation creation = docker.createContainer(containerConfig);
    final String id = creation.id();
    
    // Inspect container
    final ContainerInfo info = docker.inspectContainer(id);
    
    // Start container
    docker.startContainer(id);
    
    // Exec command inside running container with attached STDOUT and STDERR
    final String[] command = {"sh", "-c", "ls"};
    final ExecCreation execCreation = docker.execCreate(
        id, command, DockerClient.ExecCreateParam.attachStdout(),
        DockerClient.ExecCreateParam.attachStderr());
    final LogStream output = docker.execStart(execCreation.id());
    final String execOutput = output.readFully();
    
    // Kill container
    docker.killContainer(id);
    
    // Remove container
    docker.removeContainer(id);
    
    // Close the docker client
    docker.close();


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


