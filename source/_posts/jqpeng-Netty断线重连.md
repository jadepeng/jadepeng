---
title: Netty断线重连
tags: ["netty","java","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-06-01 09:52
---
文章作者:jqpeng
原文链接: [Netty断线重连](https://www.cnblogs.com/xiaoqi/p/6927387.html)



# Netty断线重连

最近使用Netty开发一个中转服务，需要一直保持与Server端的连接，网络中断后需要可以自动重连，查询官网资料，实现方案很简单，核心思想是在channelUnregistered钩子函数里执行重连。



## 创建连接

需要把configureBootstrap重构为一个函数，方便后续复用



1. EventLoopGroup group = new NioEventLoopGroup(); 
* * *
2. private volatile Bootstrap bootstrap; 
* * *
3. * * *
4. public void init(String host, int port) throws RobotException { 
* * *
5. this.serverIp = host; 
* * *
6. this.serverPort = port; 
* * *
7. try { 
* * *
8. // 创建并初始化 Netty 客户端 Bootstrap 对象 
* * *
9. bootstrap = configureBootstrap(new Bootstrap(),group); 
* * *
10. bootstrap.option(ChannelOption.TCP\_NODELAY, true); 
* * *
11. doConnect(bootstrap); 
* * *
12. } 
* * *
13. catch(Exception ex){ 
* * *
14. ex.printStackTrace(); 
* * *
15. throw new RobotException("connect remote control server error!",ex.getCause()); 
* * *
16. } 
* * *
17. } 
* * *
18. * * *
19. Bootstrap configureBootstrap(Bootstrap b, EventLoopGroup g) { 
* * *
20. b.group(g).channel(NioSocketChannel.class) 
* * *
21. .remoteAddress(serverIp, serverPort) 
* * *
22. .handler(new ChannelInitializer&lt;SocketChannel&gt;() { 
* * *
23. @Override 
* * *
24. public void initChannel(SocketChannel channel) throws Exception { 
* * *
25. ChannelPipeline pipeline = channel.pipeline(); 
* * *
26. // 编解码器 
* * *
27. pipeline.addLast(protoCodec); 
* * *
28. // 请求处理 
* * *
29. pipeline.addLast(RobotClient.this); 
* * *
30. } 
* * *
31. }); 
* * *
32. * * *
33. return b; 
* * *
34. } 
* * *
35. * * *
36. void doConnect(Bootstrap b) { 
* * *
37. try { 
* * *
38. * * *
39. ChannelFuture future = b.connect(); 
* * *
40. future.addListener(new ChannelFutureListener() { 
* * *
41. @Override 
* * *
42. public void operationComplete(ChannelFuture future) throws Exception { 
* * *
43. if (future.isSuccess()) { 
* * *
44. System.out.println("Started Tcp Client: " + serverIp); 
* * *
45. } else { 
* * *
46. System.out.println("Started Tcp Client Failed: "); 
* * *
47. } 
* * *
48. if (future.cause() != null) { 
* * *
49. future.cause().printStackTrace(); 
* * *
50. } 
* * *
51. * * *
52. } 
* * *
53. }); 
* * *
54. } catch (Exception e) { 
* * *
55. e.printStackTrace(); 
* * *
56. } 
* * *
57. } 
* * *






## 断线重连

来看断线重连的关键代码：



1. @ChannelHandler.Sharable 
* * *
2. public class RobotClient extends SimpleChannelInboundHandler&lt;RobotProto&gt; { 
* * *
3. @Override 
* * *
4. public void channelUnregistered(ChannelHandlerContext ctx) throws Exception { 
* * *
5. // 状态重置 
* * *
6. isConnected = false; 
* * *
7. this.serverStatus = -1; 
* * *
8. * * *
9. final EventLoop loop = ctx.channel().eventLoop(); 
* * *
10. loop.schedule(new Runnable() { 
* * *
11. @Override 
* * *
12. public void run() { 
* * *
13. doConnect(configureBootstrap(new Bootstrap(), loop)); 
* * *
14. } 
* * *
15. }, 1, TimeUnit.SECONDS); 
* * *
16. } 
* * *
17. } 
* * *




需要注意，Client类需要添加@ChannelHandler.Sharable注解，否则重连时会报错

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。




