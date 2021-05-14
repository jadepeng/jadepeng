---
title: 获取free-ss账号并导入SSR
tags: ["SSR","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-03-10 18:39
---
文章作者:jqpeng
原文链接: [获取free-ss账号并导入SSR](https://www.cnblogs.com/xiaoqi/p/free-ss.html)

## pre

- 如果不知道SSR，请略过
- 下载SSR
- 电脑配置host： 104.18.36.36 free-ss.site


## 获取账号

- 打开free-ss.site，等待账号出现
- F12打开开发者工具，console写入脚本,会自动解析账号，然后提示保存文件



    
    function download(filename, text) {
      var element = document.createElement('a');
      element.setAttribute('href', 'data:text/plain;charset=utf-8,' + encodeURIComponent(text));
      element.setAttribute('download', filename);
     
      element.style.display = 'none';
      document.body.appendChild(element);
     
      element.click();
     
      document.body.removeChild(element);
    }
    
    var trList = $("#tbss tr")
    ssList = []
    for (let i = 1; i < trList.length; i++) {
          const tdList = trList.eq(i).find('td');
          ssList.push({
            server: tdList.eq(1).text(),
            server_port: tdList.eq(2).text(),
            password: tdList.eq(4).text(),
            method: tdList.eq(3).text() || 'aes-256-cfb',
            group:"crawl",
            enable: true,
            remarks: tdList.eq(6).text() + ' ' + (Math.ceil(Math.random() * 10000)),
            timeout: 5
          });
        }
    download("ssr-list.txt",JSON.stringify({"configs" : ssList}))


## 导入SSR

- 打开SSR软件，服务器-从文件导入服务器，选择刚生成那个文件
- 服务器选择crawl分组的服务器
- 启用服务器负载均衡


搞定！

