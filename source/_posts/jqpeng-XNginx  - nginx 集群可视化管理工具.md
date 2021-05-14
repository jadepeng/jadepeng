---
title: XNginx  - nginx 集群可视化管理工具
tags: ["XNginx","jqpeng"]
categories: ["博客","jqpeng"]
date: 2018-09-28 17:30
---
文章作者:jqpeng
原文链接: [XNginx  - nginx 集群可视化管理工具](https://www.cnblogs.com/xiaoqi/p/xnginx.html)

之前团队的nginx管理，都是运维同学每次去修改配置文件，然后重启，非常不方便，一直想找一个可以方便管理nginx集群的工具，翻遍web，未寻到可用之物，于是自己设计开发了一个。

## 效果预览

1. 集群group管理界面


![Xnginx pic1](https://www.github.com/jadepeng/blogpic/raw/master/pic/28/1538116402607.png)

可以管理group的节点，配置文件，修改后可以一键重启所有节点，且配置文件出错时会提示错误，不会影响线上服务。

2.集群Node节点管理

![集群节点](https://www.github.com/jadepeng/blogpic/raw/master/pic/28/1538116456272.png)

3 .集群Node节点日志查看

![集群日志管理](https://www.github.com/jadepeng/blogpic/raw/master/pic/28/1538116799984.png)

1. 生成的配置文件预览


![配置文件](https://www.github.com/jadepeng/blogpic/raw/master/pic/28/1538116495120.png)

1. vhost管理


![Vhost管理](https://www.github.com/jadepeng/blogpic/raw/master/pic/28/1538116669140.png)

## 设计思路

数据结构：  
 一个nginxGroup，拥有多个NginxNode，共享同一份配置文件。

分布式架构：Manager节点+agent节点+web管理  
 每个nginx机器部署一个agent，agent启动后自动注册到manager，通过web可以设置agent所属group，以及管理group的配置文件。

配置文件变更后，manager生成配置文件，分发给存活的agent，检验OK后，控制agent重启nginx。

## 关键技术点

### 分布式管理

一般分布式可以借助zookeeper等注册中心来实现，作为java项目，其实使用EurekaServer就可以了：

manager加入eureka依赖：


        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter</artifactId>
        </dependency>
    
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>


然后在入口程序添加 @EnableEurekaServer

agent 添加注册配置：


    eureka:instance:	prefer-ip-address: trueclient:	service-url:		defaultZone: http://admin:admin@ip:3002/eureka/


manager 节点获取存活的agent，可以通过EurekaServerContextHolder来获取注册的agent，同时可以通过定时任务自动发现新节点。


    public class NginxNodeDiscover {
    
        private static final String AGENT_NAME = "XNGINXAGENT";
    
        private PeerAwareInstanceRegistry getRegistry() {
            return getServerContext().getRegistry();
        }
    
        private EurekaServerContext getServerContext() {
            return EurekaServerContextHolder.getInstance().getServerContext();
        }
    
        @Autowired
        NginxNodeRepository nginxNodeRepository;
    
        @Scheduled(fixedRate = 60000)
        public void discoverNginxNode() {
            List<String> nodes = getAliveAgents();
            nodes.stream().forEach(node->{
                if(!nginxNodeRepository.findByAgent(node).isPresent()){
                    NginxNode nginxNode = new NginxNode();
                    nginxNode.setAgent(node);
                    nginxNode.setName(node);
                    nginxNodeRepository.save(nginxNode);
                }
            });
        }
    
        public List<String> getAliveAgents() {
            List<String> instances = new ArrayList<>();
            List<Application> sortedApplications = getRegistry().getSortedApplications();
            Optional<Application> targetApp = sortedApplications.stream().filter(a->a.getName().equals(AGENT_NAME)).findFirst();
            if(targetApp.isPresent()){
                Application app = targetApp.get();
                for (InstanceInfo info : app.getInstances()) {
                    instances.add(info.getHomePageUrl());
                }
            }
            return instances;
        }
    }


### RPC调用

manager 需要控制agent，按最简单的方案，agent提供rest服务，从Eureka获取地址后直接调用就可以了，另外可以借助feign来方便调用。

定义接口：


    public interface NginxAgentManager {
    
        @RequestLine("GET /nginx/start")
         RuntimeBuilder.RuntimeResult start() ;
    
        @RequestLine("GET /nginx/status")
         RuntimeBuilder.RuntimeResult status() ;
    
        @RequestLine("GET /nginx/reload")
         RuntimeBuilder.RuntimeResult reload() ;
    
        @RequestLine("GET /nginx/stop")
         RuntimeBuilder.RuntimeResult stop();
    
        @RequestLine("GET /nginx/testConfiguration")
         RuntimeBuilder.RuntimeResult testConfiguration();
    
        @RequestLine("GET /nginx/kill")
         RuntimeBuilder.RuntimeResult kill() ;
    
        @RequestLine("GET /nginx/restart")
         RuntimeBuilder.RuntimeResult restart() ;
    
        @RequestLine("GET /nginx/info")
         NginxInfo info();
    
        @RequestLine("GET /nginx/os")
         OperationalSystemInfo os() ;
    
        @RequestLine("GET /nginx/accesslogs/{lines}")
        List<NginxLoggerVM> getAccesslogs(@Param("lines") int lines);
    
        @RequestLine("GET /nginx/errorlogs/{lines}")
        List<NginxLoggerVM> getErrorLogs(@Param("lines") int lines);
    
    }


agent 实现功能：


    @RestController
    @RequestMapping("/nginx")
    public class NginxResource {
    
       ...
    
        @PostMapping("/update")
        @Timed
        public String  update(@RequestBody NginxConf conf){
            if(conf.getSslDirectives()!=null){
                for(SslDirective sslDirective : conf.getSslDirectives()){
                    nginxControl.conf(sslDirective.getCommonName(),sslDirective.getContent());
                }
            }
            return updateConfig(conf.getConf());
        }
    
        @GetMapping("/accesslogs/{lines}")
        @Timed
        public List<NginxLoggerVM> getAccesslogs(@PathVariable Integer lines) {
            return nginxControl.getAccessLogs(lines);
        }
    
    
    }


manager 调用;

先生成一个Proxy实例，其中nodeurl是agent节点的url地址


        public NginxAgentManager getAgentManager(String nodeUrl){
            return Feign.builder()
                .options(new Request.Options(1000, 3500))
                .retryer(new Retryer.Default(5000, 5000, 3))
                .requestInterceptor(new HeaderRequestInterceptor())
                .encoder(new GsonEncoder())
                .decoder(new GsonDecoder())
                .target(NginxAgentManager.class, nodeUrl);
        }


然后调用就简单了，比如要启动group：


    public void start(String groupId){
            operateGroup(groupId,((conf, node) -> {
                NginxAgentManager manager = getAgentManager(node.getAgent());
    
                String result = manager.update(conf);
                if(!result.equals("success")){
                    throw new XNginxException("node "+ node.getAgent()+" update config file failed!");
                }
    
                RuntimeBuilder.RuntimeResult runtimeResult =   manager.start();
                if(!runtimeResult.isSuccess()){
                    throw new XNginxException("node "+ node.getAgent()+" start failed,"+runtimeResult.getOutput());
                }
            }));
        }
    
        public void operateGroup(String groupId,BiConsumer<NginxConf,NginxNode> action){
    
            List<String> alivedNodes = nodeDiscover.getAliveAgents();
            if(alivedNodes.size() == 0){
                throw new XNginxException("no alived agent!");
            }
            List<NginxNode> nginxNodes = nodeRepository.findAllByGroupId(groupId);
            if(nginxNodes.size() ==0){
                throw new XNginxException("the group has no nginx Nodes!");
            }
    
            NginxConf conf = nginxConfigService.genConfig(groupId);
    
            for(NginxNode node : nginxNodes){
    
                if(!alivedNodes.contains(node.getAgent())){
                    continue;
                }
    
                action.accept(conf, node);
           }
        }


### Nginx 配置管理

nginx的核心是各种Directive（指令），最核心的是vhost和Location。

我们先来定义VHOST：


    public class VirtualHostDirective implements Directive {
    private Integer port = 80;private String aliases;private boolean enableSSL;private SslDirective sslCertificate;private SslDirective sslCertificateKey;private List<LocationDirective> locations;
    private String root;private  String index;private String access_log;
    }


其中核心的LocationDirective，设计思路是passAddress存储location的目标地址，可以是url，也可以是upstream，通过type来区分，同时如果有upstream，则通过proxy来设置负载信息。


    public class LocationDirective {
    public static final String PROXY = "PROXY";public static final String UWSGI = "UWSGI";public static final String FASTCGI = "FASTCGI";public static final String COMMON = "STATIC";
    private String path;
    private String type = COMMON;
    private ProxyDirective proxy;
    private List<String> rewrites;
    private String advanced;
    private String passAddress;}


再来看ProxyDirective，通过balance来区分是普通的url还是upstream，如果是upstream，servers存储负载的服务器。


    public class ProxyDirective implements Directive {
    public static final String BALANCE_UPSTREAM = "upstream";public static final String BALANCE_URL = "url";
    
    private String name;
    private String strategy;
    /** *  Upstream balance type : upsteam,url */private String balance = BALANCE_UPSTREAM;
    private List<UpstreamDirectiveServer> servers;}


### 历史数据导入

已经有了配置信息，可以通过解析导入系统，解析就是常规的文本解析，这里不再赘述。

核心思想就是通过匹配大括号，将配置文件分成block，然后通过正则等提取信息,比如下面的代码拆分出server{...}


    private List<String> blocks() {
            List<String> blocks = new ArrayList<>();
            List<String> lines = Arrays.asList(fileContent.split("\n"));
    
            AtomicInteger atomicInteger = new AtomicInteger(0);
            AtomicInteger currentLine = new AtomicInteger(1);
            Integer indexStart = 0;
            Integer serverStartIndex = 0;
            for (String line : lines) {
                if (line.contains("{")) {
                    atomicInteger.getAndIncrement();
                    if (line.contains("server")) {
                        indexStart = currentLine.get() - 1;
                        serverStartIndex = atomicInteger.get() - 1;
                    }
                } else if (line.contains("}")) {
                    atomicInteger.getAndDecrement();
                    if (atomicInteger.get() == serverStartIndex) {
                        if (lines.get(indexStart).trim().startsWith("server")) {
                            blocks.add(StringUtils.join(lines.subList(indexStart, currentLine.get()), "\n"));
                        }
                    }
                }
                currentLine.getAndIncrement();
            }
            return blocks;
        }


### 配置文件生成

配置文件生成，一般是通过模板引擎，这里也不例外，使用了Velocity库。


        public static StringWriter mergeFileTemplate(String pTemplatePath, Map<String, Object> pDto) {
            if (StringUtils.isEmpty(pTemplatePath)) {
                throw new NullPointerException("????????????");
            }
            StringWriter writer = new StringWriter();
            Template template;
            try {
                template = ve.getTemplate(pTemplatePath);
            } catch (Exception e) {
                throw new RuntimeException("????????", e);
            }
            VelocityContext context = VelocityHelper.convertDto2VelocityContext(pDto);
            try {
                template.merge(context, writer);
            } catch (Exception e) {
                throw new RuntimeException("????????", e);
            }
            return writer;
        }


定义模板：


    #if(${config.user})user ${config.user};#end
    #if(${config.workerProcesses}== 0 )
    worker_processes auto;
    #else
    worker_processes  ${config.workerProcesses};
    #end
    pid        /opt/xnginx/settings/nginx.pid;
    
    events {
        multi_accept off;
        worker_connections ${config.workerConnections};
    }
    
    ...
    


生成配置文件;


        public static StringWriter buildNginxConfString(ServerConfig serverConfig, List<VirtualHostDirective> hostDirectiveList, List<ProxyDirective> proxyDirectiveList) {
            Map<String,Object> map = new HashMap<>();
            map.put("config",serverConfig);
            map.put("upstreams", proxyDirectiveList);
            map.put("hosts",hostDirectiveList);
            return VelocityHelper.mergeFileTemplate(NGINX_CONF_VM, map);
        }


### 管理web

管理web基于[ng-alain框架](https://www.cnblogs.com/xiaoqi/p/angular-ng-alain.html)，typescript+angular mvvm开发起来，和后端没有本质区别

开发相对简单，这里不赘述。

## 小结

目前只实现了基本的管理功能，后续可根据需要再继续补充完善，比如支持业务、负责人等信息管理维护。

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


