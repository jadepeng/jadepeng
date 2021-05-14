---
title: java应用诊断和在线debug利器bistoury介绍与在K8S环境使用
tags: ["Bistoury","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-04-09 20:31
---
文章作者:jqpeng
原文链接: [java应用诊断和在线debug利器bistoury介绍与在K8S环境使用](https://www.cnblogs.com/xiaoqi/p/Bistoury.html)

* * *

## Bistoury介绍

`Bistoury` 是去哪儿网开源的一个对应用透明，无侵入的java应用诊断工具，用于提升开发人员的诊断效率和能力，可以让开发人员无需登录机器或修改系统，就可以从日志、内存、线程、类信息、调试、机器和系统属性等各个方面对应用进行诊断，提升开发人员诊断问题的效率和能力。

`Bistoury` 集成了Alibaba开源的[arthas](https://github.com/alibaba/arthas)和唯品会开源的[vjtools](https://github.com/vipshop/vjtools)，因此arthas和vjtools相关功能都可以在`Bistoury`中使用。  
 Arthas和vjtools通过命令行或类似的方式使用，Bistoury在保留命令行界面的基础上，还对很多命令提供了图形化界面，方面用户使用。

`Bistoury` 英文解释是外科手术刀，含义也就不言而喻了。

## Screenshots

通过命令行界面查看日志，使用arthas和vjtools的各项功能  
![console](https://gitee.com/jadepeng/bistoury/raw/master/docs/image/console.png)

在线debug，在线应用调试神器  
![debug](https://gitee.com/jadepeng/bistoury/raw/master/docs/image/debug_panel.png)

线程级cpu监控，帮助你掌握线程级cpu使用率  
![jstack_dump](https://gitee.com/jadepeng/bistoury/raw/master/docs/image/jstack.png)

在web界面查看JVM运行信息，以及各种其它信息  
![jvm](https://gitee.com/jadepeng/bistoury/raw/master/docs/image/jvm.png)

动态给方法添加监控  
![monitor](https://gitee.com/jadepeng/bistoury/raw/master/docs/image/monitor.png)

线程dump  
![thread_dump](https://gitee.com/jadepeng/bistoury/raw/master/docs/image/thread_dump.png)

## Bistoury架构分析

Bistoury核心组件包含agent，proxy，ui：

- agent ： 与需要诊断的应用部署到一起，负责具体的诊断命令执行，通过域名连接proxy
- proxy：agent的代理，agent启动时会通过ws和proxy连接注册，proxy可以部署多个，推荐使用域名负载
- ui：ui提供图形化和命令行界面，接收从用户传来的命令，传递命令给proxy，接收从proxy传来的结果并展示给用户。


![](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586431812716.png)

一次命令执行的数据流向为 ui -&gt; proxy -&gt; agent -&gt; proxy -&gt; ui

具体分析一下：

- proxy 先启动，将自己地址注册到zk
- agent通过域名访问proxy，随机分配到一个proxy，在proxy注册自己
- UI 访问一个具体的应用时，通过zk拿到所有的proxy，然后依次检查app对应的agent是否在该proxy，如果在，web网页连接这个proxy
- web上输入一个命令:web-&gt;proxy-&gt;agent-&gt;proxy-&gt;ui


具体参见 [https://github.com/qunarcorp/bistoury/blob/master/docs/cn/design/design.md](https://github.com/qunarcorp/bistoury/blob/master/docs/cn/design/design.md)

bistoury原理分析： [https://www.jianshu.com/p/f7202e490156](https://www.jianshu.com/p/f7202e490156)

总结下就是使用类似skywalking那样的agent技术，来监测和协助运行在JVM上的程序。

## Bistoury快速开始

官方有一个快速开始文档： [https://github.com/qunarcorp/bistoury/blob/master/docs/cn/quick\_start.md](https://github.com/qunarcorp/bistoury/blob/master/docs/cn/quick_start.md)

可以下载release包快速启动，就可以体验了。

首先我们将快速启动包 bistoury-quick-start.tar.gz 拷贝到想要安装的位置。

然后解压启动包：


    tar -zxvf bistoury-quick-start.tar.gz
    cd bistoury


最后是启动 Bistoury，因为 Bistoury 会用到 jstack 等操作，为了保证所有功能可用，需要使用和待诊断 JAVA 应用相同的用户启动。

假设应用进程 id 为 1024

- 如果应用以本人用户启动，可以直接运行



    ./quick_start.sh -p 1024 start


- 如果应用以其它帐号启动，比如 tomcat，需要指定一下用户然后运行



    sudo -u tomcat ./quick_start.sh -p 1024 start


- 停止运行



    ./quick_start.sh stop


## Bistoury 在docker运行

官方的git仓库里，有一个docker分支，翻阅后找到相关文档。

官方的快速启动命令：


    #!/bin/bash
    #创建网络
    echo "start create network"
    docker network create --subnet=172.19.0.0/16 bistoury
    #mysql 镜像
    echo "start run mysql image"
    docker run --name mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root -d -i --net bistoury --ip 172.19.0.7  registry.cn-hangzhou.aliyuncs.com/bistoury/bistoury-db
    #zk 镜像
    echo "start run zk image"
    docker run -d -p 2181:2181 -it --net bistoury --ip 172.19.0.2 registry.cn-hangzhou.aliyuncs.com/bistoury/zk:latest
    sleep 30
    #proxy 镜像
    echo "start run proxy module"
    docker run -d -p 9880:9880 -p 9881:9881 -p 9090:9090 -i --net bistoury --ip 172.19.0.3 registry.cn-hangzhou.aliyuncs.com/bistoury/bistoury-proxy --real-ip $1 --zk-address 172.19.0.2:2181 --proxy-jdbc-url jdbc:mysql://172.19.0.7:3306/bistoury
    #ui 镜像
    echo "start run ui module"
    docker run -p 9091:9091  -it -d --net bistoury --ip 172.19.0.4 registry.cn-hangzhou.aliyuncs.com/bistoury/bistoury-ui --zk-address 172.19.0.2:2181 --ui-jdbc-url jdbc:mysql://172.19.0.7:3306/bistoury
    #boot 镜像
    echo "start run demo application"
    docker  run -it -d  -p 8686:8686 -i --net bistoury --ip 172.19.0.5 registry.cn-hangzhou.aliyuncs.com/bistoury/bistoury-demo --proxy-host $1:9090
    docker  run -it -d  -p 8687:8686 -i --net bistoury --ip 172.19.0.6 registry.cn-hangzhou.aliyuncs.com/bistoury/bistoury-demo --proxy-host $1:9090
    


上面的命令不能直接运行，`$1`是需要替换成当前服务器IP，然后再运行就OK了。

## Bistoury 在生产环境运行

官方推荐部署方式：

- ui 独立部署，推荐部署在多台机器，并提供独立的域名
- proxy 独立部署，推荐部署在多台机器，并提供独立的域名
- agent 需要和应用部署在同一台机器上。推荐在测试环境全环境自动部署，线上环境提供单机一键部署，以及应用下所有机器一键部署
- 独立的应用中心，管理所有功能内部应用和机器信息，这是一个和 Bistoury 相独立的系统，Bistoury 从中拿到不断更新的应用和机器信息


这里有个关键的点，应用中心，Bistoury内置了一个简单的应用中心，Bistoury里代码对应bistoury-application，ui和proxy都通过这个工程获取应用信息，官方默认实现了一个mysql版本的：

![application](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586433126733.png)

使用mysql的缺点是，你需要ui界面里手动维护应用以及应用的服务器，做个demo还OK，生产环境肯定不行。更优雅的方式是，用户系统应该在启动时自动注册到注册中心上，汇报自己的应用、机器信息（ip、域名等）、端口等信息。当然这个对大部分微服务架构来说，注册中心是标配的，因此实现一套bistoury-application-api接口即可。

## bistoury-application-k8s(Bistoury on K8S)

我们项目组所有的应用都部署在K8S环境，因此要实现一个`bistoury-application-k8s`。

拷贝`bistoury-application-mysql`项目，建立`bistoury-application-k8s`

简单对应下：

- 一个应用对应一个deployment，对应一个application
- 一个deployment里有n个pod，对应applicationServer


所以，我们只需要调用调用K8S API 获取deployment和pod即可。

首先引入相关jar包：


       <dependency>
                <groupId>io.kubernetes</groupId>
                <artifactId>client-java</artifactId>
                <version>8.0.0</version>
                <scope>compile</scope>
            </dependency>


初始化ApiClient


    			ApiClient defaultClient = Configuration.getDefaultApiClient();
                defaultClient.setBasePath(k8sApiServer);
                ApiKeyAuth BearerToken = (ApiKeyAuth) defaultClient.getAuthentication("BearerToken");
                BearerToken.setApiKey(k8sToken);
                BearerToken.setApiKeyPrefix("Bearer");
                defaultClient.setVerifyingSsl(false);


### 获取deployment

区分下是获取所有namespace，还是获取指定的namespace


       private List<V1Deployment> getDeployments() throws ApiException {
            AppsV1Api appsV1Api = new AppsV1Api(k8SConfiguration.getApiClient());
            return k8SConfiguration.isAllNamespace()
                    ? appsV1Api.listDeploymentForAllNamespaces(false, null, null, null, 0, null, null, 120, false).getItems()
                    : getNamespacesDeployments(k8SConfiguration.getAllowedNamespace());
        }
    
        List<V1Deployment> getNamespacesDeployments(List<String> namespaces) {
            AppsV1Api appsV1Api = new AppsV1Api(k8SConfiguration.getApiClient());
            List<V1Deployment> deploymentList = new ArrayList<>();
            for (String nameSpace : namespaces) {
                try {
                    deploymentList.addAll(appsV1Api.listNamespacedDeployment(nameSpace, null, null, null, null, null, 0, null, 120, false).getItems());
                } catch (ApiException e) {
                    logger.error("get " + nameSpace + "'s deployment error", e);
                }
            }
            return deploymentList;
        }


转换为application：


        private List<Application> getApplications(List<V1Deployment> applist) {
            return applist.stream().map(this::getApplication).collect(Collectors.toList());
        }
    
        private Application getApplication(V1Deployment deployment) {
            Application application = new Application();
            application.setCreateTime(deployment.getMetadata().getCreationTimestamp().toDate());
            application.setCreator(deployment.getMetadata().getName());
            application.setGroupCode(deployment.getMetadata().getNamespace());
            application.setName(deployment.getMetadata().getName());
            application.setStatus(1);
            application.setCode(getAppCode(deployment.getMetadata().getNamespace(), deployment.getMetadata().getName()));
            return application;
        }


### 获取pod

获取pod相对麻烦点，需要先获取到V1Deployment，拿到部署的lableSelector，然后根据lableSelector选择pod：


     public List<AppServer> getAppServerByAppCode(final String appCode) {
            Preconditions.checkArgument(!Strings.isNullOrEmpty(appCode), "app code cannot be null or empty");
    
            try {
                V1Deployment deployment = getDeployMent(appCode);
                String nameSpace = appCode.split(APPCODE_SPLITTER)[0];
                Map<String, String> labelMap = Objects.requireNonNull(deployment.getSpec()).getSelector().getMatchLabels();
                StringBuilder lableSelector = new StringBuilder();
                labelMap.entrySet().stream().forEach(e -> {
                    if (lableSelector.length() > 0) {
                        lableSelector.append(",");
                    }
                    lableSelector.append(e.getKey()).append("=").append(e.getValue());
                });
    
                CoreV1Api coreV1Api = new CoreV1Api(k8SConfiguration.getApiClient());
                V1PodList podList = coreV1Api.listNamespacedPod(nameSpace, null, false, null,
                        null, lableSelector.toString(), 200, null, 600, false);
    
                return podList.getItems().stream().map(pod -> {
                    AppServer server = new AppServer();
                    server.setAppCode(appCode);
                    server.setHost(pod.getMetadata().getName());
                    server.setIp(pod.getStatus().getPodIP());
                    server.setLogDir(k8SConfiguration.getAppLogPath());
                    server.setAutoJMapHistoEnable(true);
                    server.setAutoJStackEnable(true);
                    server.setPort(8080);
                    return server;
                }).collect(Collectors.toList());
    
            } catch (ApiException e) {
                logger.error("get deployment's pod  error", e);
            }
    
            return null;
    
        }


最后，修改ui和proxy工程，将原来的mysql替换为k8s：

![修改pom](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586434230879.png)

## 应用引入bistoury agent

这块相对比较容易：

在需要调试的应用的Dockerfile里增加：


    COPY  --from=hub.xfyun.cn/abkdev/bistoury-agent:2.0.11  /home/q/bistoury  /opt/bistoury


然后修改应用的启动脚本，在最前面增加：


    BISTOURY_APP_LIB_CLASS="org.springframework.web.servlet.DispatcherServlet"
    
    # default proxy
    PROXY="bistoury-bistoury-proxy.incubation:9090"
    AGENT_JAVA_HOME="/usr/local/openjdk-8/"
    
    # env
    if [[ -n $PROXY_HOST ]]; then
        PROXY=$PROXY_HOST
    fi
    
    TEMP=`getopt -o : --long proxy-host:,app-class:,agent-java-home: -- "$@"`
    
    eval set -- "$TEMP"
    
    while true; do
      case "$1" in
        --proxy-host )
          PROXY="$2"; shift 2 ;;
        --app-class )
          BISTOURY_APP_LIB_CLASS="$2"; shift 2 ;;
        --agent-java-home )
          AGENT_JAVA_HOME="$2"; shift 2 ;;
        * ) break ;;
      esac
    done
    
    
    echo "proxy host: "$PROXY_HOST
    echo "app class: "$BISTOURY_APP_LIB_CLASS
    echo "agent java home: "$AGENT_JAVA_HOME
    


在最后面增加：


    APP_PID=`$AGENT_JAVA_HOME/bin/jps -l|awk '{if($2!="sun.tools.jps.Jps"){print $1 ;{exit}} }'`
    
    echo "app pid: "$APP_PID
    
    /opt/bistoury/agent/bin/bistoury-agent.sh -j $AGENT_JAVA_HOME -p $APP_PID -c $BISTOURY_APP_LIB_CLASS -s $PROXY -f start


## 集成测试

部署一个测试应用 agent-debug-demo，部署到jx namespace：

![ agent-debug-demo](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586434610449.png)


    {
      "kind": "Deployment",
      "apiVersion": "extensions/v1beta1",
      "metadata": {
        "name": "agent-debug-demo",
        "namespace": "jx",
        "annotations": {
          "deployment.kubernetes.io/revision": "2"
        }
      },
      "spec": {
        "replicas": 1,
        "selector": {
          "matchLabels": {
            "app": "agent-debug-demo",
            "draft": "draft-app"
          }
        },
        "template": {
          "metadata": {
            "creationTimestamp": null,
            "labels": {
              "app": "agent-debug-demo",
              "draft": "draft-app"
            }
          },
          "spec": {
            "containers": [
              {
                "name": "springboot-rest-demo",
                "image": "hub.xxx.cn/abkdev/springboot-rest-demo:dev-113",
                "ports": [
                  {
                    "containerPort": 8080,
                    "protocol": "TCP"
                  }
                ],
                "env": [
                  {
                    "name": "SPRING_PROFILES_ACTIVE",
                    "value": "dev"
                  },
                  {
                    "name": "PROXY_HOST",
                    "value": "$PROXY_HOST:9090"
                  }
                ],
                "resources": {},
                "terminationMessagePath": "/dev/termination-log",
                "terminationMessagePolicy": "File",
                "imagePullPolicy": "IfNotPresent"
              }
            ],
            "restartPolicy": "Always",
            "terminationGracePeriodSeconds": 10,
            "dnsPolicy": "ClusterFirst",
            "securityContext": {},
            "schedulerName": "default-scheduler"
          }
        },
        "strategy": {
          "type": "RollingUpdate",
          "rollingUpdate": {
            "maxUnavailable": 1,
            "maxSurge": 1
          }
        },
        "revisionHistoryLimit": 2147483647,
        "progressDeadlineSeconds": 2147483647
      },
      "status": {
        "observedGeneration": 2,
        "replicas": 1,
        "updatedReplicas": 1,
        "unavailableReplicas": 1,
        "conditions": [
          {
            "type": "Available",
            "status": "True",
            "lastUpdateTime": "2020-04-09T01:32:42Z",
            "lastTransitionTime": "2020-04-09T01:32:42Z",
            "reason": "MinimumReplicasAvailable",
            "message": "Deployment has minimum availability."
          }
        ]
      }
    }


部署后：

![K8S Dashboard](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586434889358.png)

打开ui，查看：

应用名称显示为： namespace名称-部署名称

![bistoury](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586434848921.png)

![ThreadDump](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586434991429.png)

在线调试：

先选择应用：

![选择应用](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586435034455.png)

点击Debug，然后选择需要调试的类，

测试工程源代码为：


    @SpringBootApplication
    @Controller
    public class RestPrometheusApplication {
    @Autowiredprivate MeterRegistry registry;
    @Autowiredprivate Environment env;
    @GetMapping(path = "/", produces = "application/json")@ResponseBodypublic Map<String, Object> landingPage() {	Counter.builder("mymetric").tag("foo", "bar").register(registry).increment();	String profile = "default";	if(env.getActiveProfiles().length > 0){		profile = env.getActiveProfiles()[0];	}
    	return singletonMap("hello", ""+ profile);}
    public static void main(String[] args) {	SpringApplication.run(RestPrometheusApplication.class, args);}
    
    }


因此，我们输入RestPrometheusApplication筛选：

![选择Class](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586435105776.png)

然后点击调试，可以看到，反编译出来了源代码：

![在线debug1](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586435149849.png)

在landingPage最后一行加一个端点，然后点击添加端点，最后访问该POD对应的服务,该pod对应的ip是170.22.149.37，因此我们访问：


    curl http://170.22.149.37:8080
    {"hello":"dev"}


再回到UI，可以看到成员变量，局部变量和调用堆栈等信息。

![在线debug2](https://gitee.com/jadepeng/pic/raw/master/pic/2020/4/9/1586435287216.png)

well down！

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


