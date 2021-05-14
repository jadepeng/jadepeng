---
title: 使用websocket-sharp来创建c#版本的websocket服务
tags: ["c#","websocket","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-06-22 17:34
---
文章作者:jqpeng
原文链接: [使用websocket-sharp来创建c#版本的websocket服务](https://www.cnblogs.com/xiaoqi/p/websocket-sharp.html)

当前有一个需求，需要网页端调用扫描仪，javascript不具备调用能力，因此需要在机器上提供一个ws服务给前端网页调用扫描仪。而扫描仪有一个c#版本的API，因此需要寻找一个c#的websocket库。

java里有大名鼎鼎的netty，通过搜索，c#可以选择[websocket-sharp](https://github.com/sta/websocket-sharp)来实现websocket Server。

### 使用websocket-sharp创建websocket server###


    using System;
    using WebSocketSharp;
    using WebSocketSharp.Server;
    
    namespace Example
    {
      public class Laputa : WebSocketBehavior
      {
        protected override void OnMessage (MessageEventArgs e)
        {
          var msg = e.Data == "BALUS"
                    ? "I've been balused already..."
                    : "I'm not available now.";
    
          Send (msg);
        }
      }
    
      public class Program
      {
        public static void Main (string[] args)
        {
          var wssv = new WebSocketServer ("ws://dragonsnest.far");
          wssv.AddWebSocketService<Laputa> ("/Laputa");
          wssv.Start ();
          Console.ReadKey (true);
          wssv.Stop ();
        }
      }
    }


#### Step 1

Required namespace.


    using WebSocketSharp.Server;


The `WebSocketBehavior` and `WebSocketServer` 两个类需要引用 `WebSocketSharp.Server` namespace.

#### Step 2

编写处理类，需要继承 `WebSocketBehavior` class.

例如，如果你要创建一个echo Service,


    using System;
    using WebSocketSharp;
    using WebSocketSharp.Server;
    
    public class Echo : WebSocketBehavior
    {
      protected override void OnMessage (MessageEventArgs e)
      {
        Send (e.Data);
      }
    }


再提供一个 chat service,


    using System;
    using WebSocketSharp;
    using WebSocketSharp.Server;
    
    public class Chat : WebSocketBehavior
    {
      private string _suffix;
    
      public Chat ()
        : this (null)
      {
      }
    
      public Chat (string suffix)
      {
        _suffix = suffix ?? String.Empty;
      }
    
      protected override void OnMessage (MessageEventArgs e)
      {
        Sessions.Broadcast (e.Data + _suffix);
      }
    }


可以通过继承`WebSocketBehavior`类来自定义Service.

通过重载 `WebSocketBehavior.OnMessage (MessageEventArgs)` 方法, 来处理消息

同时你也可以重载 `WebSocketBehavior.OnOpen ()`, `WebSocketBehavior.OnError (ErrorEventArgs)`, 和 `WebSocketBehavior.OnClose (CloseEventArgs)` 方法,来处理websocket连接事件。

通过`WebSocketBehavior.Send` 方法来给客户端发送消息。

If you would like to get the sessions in the service, you should access the `WebSocketBehavior.Sessions` property (returns a `WebSocketSharp.Server.WebSocketSessionManager`).

The `WebSocketBehavior.Sessions.Broadcast` method can send data to every client in the service.

#### Step 3

创建 `WebSocketServer` 对象.


    var wssv = new WebSocketServer (4649);
    wssv.AddWebSocketService<Echo> ("/Echo");
    wssv.AddWebSocketService<Chat> ("/Chat");
    wssv.AddWebSocketService<Chat> ("/ChatWithNyan", () => new Chat (" Nyan!"));


#### Step 4

启动 WebSocket server.


    wssv.Start ();


#### Step 5

停止 WebSocket server.


    wssv.Stop (code, reason);


### 测试Demo

`目的`：对外提供一个websocket服务，让网页端的js可以调用扫描仪

#### 服务端代码


     class Program
        {
            static void Main(string[] args)
            {
                var wssv = new WebSocketServer(10086);
                wssv.AddWebSocketService<ScannerHandler>("/scan");
                wssv.Start();
                if (wssv.IsListening)
                {
                    Console.WriteLine("Listening on port {0}, and providing WebSocket services:", wssv.Port);
                    foreach (var path in wssv.WebSocketServices.Paths)
                        Console.WriteLine("- {0}", path);
                }
    
                Console.WriteLine("\nPress Enter key to stop the server...");
                Console.ReadLine();
    
                wssv.Stop();
            }
        }
    
        public class ScannerHandler : WebSocketBehavior
        {
            protected override void OnMessage(MessageEventArgs e)
            {
                if(e.Data == "scan")
                {
                    ScanResult result = ScanerHelper.Scan("D:\\test.jpg");
                    if (result.Success)
                    {
                        Console.WriteLine("scan success");
                        Send("scan success");
                    }
                    else
                    {
                        Send("scan eror");
                    }
                }
               
            }
        }


#### 前端代码

javascript代码


     	var ws;
        function initWS() {
            ws = new WebSocket("ws://127.0.0.1:10086/scan");
            ws.onopen = function () {
                console.log("Openened connection to websocket");
    
            };
            ws.onclose = function () {
                console.log("Close connection to websocket");
                // 断线重连
                initWS();
            }
    
            ws.onmessage = function (e) {
                alert(e.data)
            }
        }
        initWS();
        function scan() {
            ws && ws.send('scan');
        }


html代码


    <button onclick="scan()">扫描</button>


- initWS创建连接，支持断线重连
- 可以调用scan函数，发送scan指令


