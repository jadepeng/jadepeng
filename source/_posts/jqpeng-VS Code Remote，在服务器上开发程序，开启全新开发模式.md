---
title: VS Code Remote，在服务器上开发程序，开启全新开发模式
tags: ["jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-05-08 11:19
---
文章作者:jqpeng
原文链接: [VS Code Remote，在服务器上开发程序，开启全新开发模式](https://www.cnblogs.com/xiaoqi/p/vs-code-remote.html)

一直使用Idea开发java 程序，头疼的是太太太占用内存了，笔记本电脑经常卡爆，在服务器开发的话又太麻烦，VS Code Remote的带来，解决了这一烦恼。下面来实战一下。

## VS Code Remote

2019 年 5 月 3 日，在 PyCon 2019 大会上，微软发布了 VS Code Remote，开启了远程开发的新时代  
 。

![VS](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-8/1557283868532.png)

Visual Studio Code Remote 允许开发者将容器，远程计算机，或 Windows Subsystem for Linux (WSL) 作为完整的开发环境。你可以：

- 在部署相同的操作系统上进行开发，或者使用更大或更专业的硬件。
- 把开发环境作为沙箱，以避免影响本地计算机配置。
- 让新手轻松上手，让每个人都保持一致的开发环境。
- 使用原本在本地环境不可用的工具或运行时，或者管理它们的多个版本。
- 在 WSL 里开发 Linux 应用。
- 从多台不同的计算机访问现有的开发环境。
- 调试在其他位置（比如客户网站或云端）运行的应用程序。


所有以上的功能，并不需要在你的本地开发环境有源代码。通过 VS Code Remote，轻松连接上远程环境，在本地进行开发。

下面来实战。

## 安装vs code insiders

需要先安装最新的内部体验版，[https://code.visualstudio.com/insiders/](https://code.visualstudio.com/insiders/)

然后安装Remote Development插件

[https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)

为了简单起见，我们采用SSH模式。需要先在windows机器安装OpenSSH

## windows 10 安装OpenSSH

使用 PowerShell 安装 OpenSSH

若要安装使用 PowerShell 的 OpenSSH，请首先以管理员身份启动 PowerShell。 若要确保 OpenSSH 功能以安装方式提供：

PowerShell复制


    Get-WindowsCapability -Online | ? Name -like 'OpenSSH*'
    
    # This should return the following output:
    
    Name  : OpenSSH.Client~~~~0.0.1.0
    State : NotPresent
    Name  : OpenSSH.Server~~~~0.0.1.0
    State : NotPresent
    


然后，安装服务器和/或客户端功能：

PowerShell复制


    # Install the OpenSSH Client
    Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0
    
    # Install the OpenSSH Server
    Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
    
    # Both of these should return the following output:
    
    Path          :
    Online        : True
    RestartNeeded : False


## SSH 认证

- 先ssh-keygen生车密钥
- 然后ssh-copy-id 到服务器



     ssh-copy-id root@YOUR-SERVER-IP
    /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/c/Users/jqpeng/.ssh/id_ed25519.pub"
    The authenticity of host 'YOUR-SERVER-IP' can't be established.
    ECDSA key fingerprint is SHA256:HRwsmslg5ge+JYcOjW6zRtUxrFeWJ5V2AojlIvLaykc.
    Are you sure you want to continue connecting (yes/no)? yes
    /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filterout any that are already installed
    /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
    root@YOUR-SERVER-IP's password:
    
    Number of key(s) added: 1
    
    Now try logging into the machine, with:   "ssh 'root@YOUR-SERVER-IP'"
    and check to make sure that only the key(s) you wanted were added.


## 使用VS code inside 开发程序

准备工作：

- 确保服务器已有JDK，mvn，没有的话先安装好
- 将代码签出到服务器一个目录


打开VS code，命令行：

![enter description here](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-8/1557284388601.png)

选择`connect to host`:

然后输入root@YOUR\_SERVETR\_IP

![enter description here](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-8/1557284444999.png)

回车，VS 会自动在服务器准备相关环境。

搞定后，点击文件打开文件夹，VS Code会列出服务器的目录，选择项目所在地址打开即可。

![enter description here](https://markdown.xiaoshujiang.com/img/spinner.gif "[[[1557284521346]]]")

接下来安装必要的语言插件，打开一个java文件，vs code会自动图惨案安装一些插件，把java相关的安装好：

![enter description here](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-8/1557284624056.png)

## 调试程序

打开包含main的java文件，点击调试菜单，会自动生成一个启动文件，配置下即可：


    {
        // 使用 IntelliSense 了解相关属性。 
        // 悬停以查看现有属性的描述。
        // 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
        "version": "0.2.0",
        "configurations": [
            {
                "type": "java",
                "name": "AimindWebApplication",
                "request": "launch",
                "mainClass": "com.xxx.xxx.XXXWebApplication"
            }
        ]
    }


然后启动。

惊喜的发现，在main函数上方，自动出现了RUN|DEBUG，见下图，点击debug即可启动调试

![自动识别的main](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-8/1557285054203.png)

在调试控制台可以看到对应的输出。

![调试控制台](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-8/1557285150474.png)

## 内存占用

之前IDEA启动调试后，内存占用2G+，VS code呢？400M+！

![VS CODE remote 内存占用](https://www.github.com/jadepeng/blogpic/raw/master/images/2019/5-8/1557285229259.png)

把耗费计算资源、内存的都放到服务器上去执行了，本地只需要负责View，所以资源占用极小。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


