---
title: java内嵌jetty服务器
tags: ["jetty","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-01-17 15:54
---
文章作者:jqpeng
原文链接: [java内嵌jetty服务器](https://www.cnblogs.com/xiaoqi/p/6293422.html)

有的时候需要将一个简单的功能封装为服务，相比python使用flask、web.py的简洁，使用java-web显得太重量级，幸好，我们可以直接在java项目中使用jetty来搭建简易服务



1、pom.xml加入jetty依赖



    <dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-server</artifactId>
    <version>9.4.0.v20161208</version>
    </dependency>
    
    <dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-webapp</artifactId>
    <version>9.4.0.v20161208</version>
    </dependency>
    
    <dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-continuation</artifactId>
    <version>9.4.0.v20161208</version>
    </dependency>
    
    <dependency>
    <groupId>org.eclipse.jetty</groupId>
    <artifactId>jetty-jsp</artifactId>
    <version>9.1.4.v20140401</version>
    </dependency>





2、增加Server



    Serverserver=newServer(12580);





3、设置ServletContextHandler



    ServletContextHandlercontext=newServletContextHandler(server,"/");
    server.setHandler(context);





4、Context增加Servlet   
4.1 创建Servlet 继承HttpServlet，重载doGet，doPost即可



    public class XXXHandler extends HttpServlet {
    
        @Override
        protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
            JSONObject ret =  new JSONObject();
            try {
                String ttsTxt = req.getParameter("text");
    
                String outFile = System.nanoTime() + ".mp4";
                String url = xx.xxx(ttsTxt,...);
                ret.put("ret","0");
                ret.put("url",url);
            }catch (Exception ex){
                ret.put("ret","-1");
                ret.put("error",ex.getMessage());
            }
            if(req.getParameter("callback")!=null) {
                resp.getWriter().write(req.getParameter("callback")+"("+ret.toString()+")");
            }else {
                resp.getWriter().write(ret.toString());
            }
        }
    
    }





4.2 将Servlet 加入Context



    context.addServlet(xxxHandler.class,"/xxx");
    context.addServlet(Image2VideoHandler.class,"/*");





5、启动server



    server.start();
    server.join();



6、在浏览器访问http://localhost:12580/XXX 即可

