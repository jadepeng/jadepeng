---
title: conda docker镜像
tags: ["conda","docker","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-09-10 16:36
---
文章作者:jqpeng
原文链接: [conda docker镜像](https://www.cnblogs.com/xiaoqi/p/conda-docker.html)

之前的python环境，使用ubuntu安装pip来安装python依赖，但是遇到缺少某些库的版本，比如一个项目需要用到faiss，pip只有最新的1.5.3版本，但是这个版本使用了较新的CPU指令，在老服务器上运行报错：

Illegal instruction (core dumped) - in new version of FAISS #885

github上提示安装旧版本：


> If anyone else is struggling and wanna go back to previous working version, use: conda install faiss-cpu=1.5.1 -c pytorch -y


遗憾的是，下面的命令不成功，没有1.5.1版本：


    pip install faiss-cpu==1.5.1


转而投向conda。

首先，下载最新的conda安装命令：


    wget https://repo.anaconda.com/archive/Anaconda3-2019.07-Linux-x86_64.sh


然后构建conda的基础镜像，还是以ubuntu:16.04为底包，Dockerfile如下：


    from ubuntu:16.04
    RUN apt-get update && apt-get install -y --no-install-recommends \
          bzip2 \
          g++ \
          git \
          graphviz \
          libgl1-mesa-glx \
          libhdf5-dev \
          openmpi-bin \
          wget && \
        rm -rf /var/lib/apt/lists/*
    
    RUN sed -i 's/archive.ubuntu.com/mirrors.ustc.edu.cn/g' /etc/apt/sources.list
    RUN apt-get update
    
    ADD ./Anaconda3-2019.07-Linux-x86_64.sh ./anaconda.sh
    
    ENV LANG=C.UTF-8 LC_ALL=C.UTF-8
    ENV PATH /opt/conda/bin:$PATH
    RUN  /bin/bash ./anaconda.sh -b -p /opt/conda  && rm ./anaconda.sh && ln -s /opt/conda/etc/profile.d/conda.sh /etc/profile.d/conda.sh  && echo ". /opt/conda/etc/profile.d/conda.sh" >> ~/.bashrc && echo "conda activate base" >> ~/.bashrc && find /opt/conda/ -follow -type f -name '*.a' -delete && find /opt/conda/ -follow -type f -name '*.js.map' -delete &&  /opt/conda/bin/conda clean -afy
    
    
    CMD [ "/bin/bash" ]


构建：


    docker build -t conda3:1.0 .


后面，就可以以`conda3:1.0 .`为基础镜像构建需要的镜像，比如我们需要安装faiss-cpu 1.5.1版本


    from conda3:1.0
    
    RUN conda install pytorch -y
    RUN conda install faiss-cpu=1.5.1 -c pytorch -y
    
    
    CMD [ "/bin/bash" ]


构建：


    docker build -t conda-faiss:1.0 .


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


