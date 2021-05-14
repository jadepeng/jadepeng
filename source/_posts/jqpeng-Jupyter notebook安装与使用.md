---
title: Jupyter notebook安装与使用
tags: ["python","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-02-13 14:35
---
文章作者:jqpeng
原文链接: [Jupyter notebook安装与使用](https://www.cnblogs.com/xiaoqi/p/6393677.html)

- Jupyter Notebook（此前被称为 IPython notebook）是一个交互式笔记本，支持运行 40 多种编程语言。
- 安装

    - 安装python 3
    - pip安装
        pip3 install --upgrade pip
        pip3 install jupyter
- 运行 

    - jupyter notebook 启动，会默认打开http://localhost:8888/，访问用户目录
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144027332-1540737258.png)
        
        如何自定义端口？
        *jupyter notebook --port 9999*
    - Nootbook dashboard
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144027957-1866895760.png)
- 笔记操作

    - 打开已有笔记
        例如，打开网友分享的CS231课程笔记，https://github.com/zlotus/cs231n
        下载zip包，解压到用户目录，刷新dashboard就能看到了
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144028613-1675066940.png)
        .ipynb就是保存的笔记，单击打开，enjoy it！
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144029379-1230425518.png)
    - 运行实例管理
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144030004-1079356774.png)
        在running里可以查看正在运行的实例，可以shutdown
    - 新建笔记
        点击new菜单里的Python 3就可以创建一个笔记了
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144030425-1666909010.png)
    - 插入代码
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144032285-207623686.png)
    - 运行代码
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144032785-1190858195.png)
        
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144033207-856058105.png)
    - 增加文本段
        点击新增（+），选择类型为markdown
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144033863-833209297.png)
    - 保存
        CTRL+S 或点击save按钮保存
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144034285-1049238861.png)
    - 命名
        点击标题，会弹出rename框
        ![](https://images2015.cnblogs.com/38465/201702/38465-20170213144034769-1640688543.png)


