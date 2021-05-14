---
title: Docker启用TLS进行安全配置
tags: ["docker","TLS","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-08-08 20:08
---
文章作者:jqpeng
原文链接: [Docker启用TLS进行安全配置](https://www.cnblogs.com/xiaoqi/p/docker-tls.html)

之前开启了docker的2375 Remote API，接到公司安全部门的要求，需要启用授权，翻了下官方文档

[Protect the Docker daemon socket](https://docs.docker.com/engine/security/https/)

## 启用TLS

在docker服务器，生成CA私有和公共密钥


    $ openssl genrsa -aes256 -out ca-key.pem 4096
    Generating RSA private key, 4096 bit long modulus
    ............................................................................................................................................................................................++
    ........++
    e is 65537 (0x10001)
    Enter pass phrase for ca-key.pem:
    Verifying - Enter pass phrase for ca-key.pem:
    
    $ openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
    Enter pass phrase for ca-key.pem:
    You are about to be asked to enter information that will be incorporated
    into your certificate request.
    What you are about to enter is what is called a Distinguished Name or a DN.
    There are quite a few fields but you can leave some blank
    For some fields there will be a default value,
    If you enter '.', the field will be left blank.
    -----
    Country Name (2 letter code) [AU]:
    State or Province Name (full name) [Some-State]:Queensland
    Locality Name (eg, city) []:Brisbane
    Organization Name (eg, company) [Internet Widgits Pty Ltd]:Docker Inc
    Organizational Unit Name (eg, section) []:Sales
    Common Name (e.g. server FQDN or YOUR name) []:$HOST
    Email Address []:Sven@home.org.au


有了CA后，可以创建一个服务器密钥和证书签名请求(CSR)


> $HOST 是你的服务器ip



    $ openssl genrsa -out server-key.pem 4096
    Generating RSA private key, 4096 bit long modulus
    .....................................................................++
    .................................................................................................++
    e is 65537 (0x10001)
    
    $ openssl req -subj "/CN=$HOST" -sha256 -new -key server-key.pem -out server.csr
    


接着，用CA来签署公共密钥:


    $ echo subjectAltName = DNS:$HOST,IP:$HOST:127.0.0.1 >> extfile.cnf
    
     $ echo extendedKeyUsage = serverAuth >> extfile.cnf
     


生成key：


    $ openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem \
      -CAcreateserial -out server-cert.pem -extfile extfile.cnf
    Signature ok
    subject=/CN=your.host.com
    Getting CA Private Key
    Enter pass phrase for ca-key.pem:
    


创建客户端密钥和证书签名请求:


    
    $ openssl genrsa -out key.pem 4096
    Generating RSA private key, 4096 bit long modulus
    .........................................................++
    ................++
    e is 65537 (0x10001)
    
    $ openssl req -subj '/CN=client' -new -key key.pem -out client.csr
    


修改`extfile.cnf`：


    echo extendedKeyUsage = clientAuth > extfile-client.cnf


生成签名私钥：


    $ openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem \
      -CAcreateserial -out cert.pem -extfile extfile-client.cnf
    Signature ok
    subject=/CN=client
    Getting CA Private Key
    Enter pass phrase for ca-key.pem:


将Docker服务停止，然后修改docker服务文件


    [Unit]
    Description=Docker Application Container Engine
    Documentation=http://docs.docker.io
    
    [Service]
    Environment="PATH=/opt/kube/bin:/bin:/sbin:/usr/bin:/usr/sbin"
    ExecStart=/opt/kube/bin/dockerd  --tlsverify --tlscacert=/root/docker/ca.pem --tlscert=/root/docker/server-cert.pem --tlskey=/root/docker/server-key.pem -H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375
    ExecStartPost=/sbin/iptables -I FORWARD -s 0.0.0.0/0 -j ACCEPT
    ExecReload=/bin/kill -s HUP $MAINPID
    Restart=on-failure
    RestartSec=5
    LimitNOFILE=infinity
    LimitNPROC=infinity
    LimitCORE=infinity
    Delegate=yes
    KillMode=process
    
    [Install]
    WantedBy=multi-user.target
    


然后重启服务


    systemctl daemon-reload
    systemctl restart docker.service 



    重启后查看服务状态：
    
    systemctl status docker.service
    ● docker.service - Docker Application Container Engine
       Loaded: loaded (/etc/systemd/system/docker.service; enabled; vendor preset: enabled)
       Active: active (running) since Thu 2019-08-08 19:22:26 CST; 1 min ago


已经生效。

使用证书连接：

复制`ca.pem`,`cert.pem`,`key.pem`三个文件到客户端

`docker --tlsverify --tlscacert=ca.pem --tlscert=cert.pem --tlskey=key.pem -H=$HOST:2375 version`连接即可

## docker-java 启用TLS

项目里使用docker的java客户端`docker-java`调用docker，为了支持TLS，在创建客户端时，需要增加TLS设置。

首先将`ca.pem cert.pem key.pem `这三个文件拷贝到本地，例如`E:\\docker\\"`,

然后`DefaultDockerClientConfig`里`withDockerTlsVerify`设为true，并设置`certpath`为刚拷贝的目录。


    DefaultDockerClientConfig.Builder builder =
                    DefaultDockerClientConfig.createDefaultConfigBuilder()
                        .withDockerHost("tcp://" + server + ":2375")
                        .withApiVersion("1.30");
                if (containerConfiguration.getDockerTlsVerify()) {
                    builder = builder.withDockerTlsVerify(true)
                        .withDockerCertPath("E:\\docker\\");
                }return  DockerClientBuilder.getInstance(builder.build()).build()		


大工搞定。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


