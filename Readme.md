# JadePeng的技术笔记本

## 安装 Hexo

    $ npm install -g hexo-cli

## 签出代码

    $ npm install

## 内容管理

### 手动创建

可直接在`source\_posts\` 文件夹里新建md文件，注意文件头格式：

```markdown
---
title:  从Spring框架看设计模式如何灵活使用
tags:
  - 设计模式
  - Spring
  - jqpeng
categories: ["技术分享"]
date: 2021-04-27 19:48:22
---
```

`tags`可以根据内容增加，**最后一个tag需要为自己的域账号**

`categories` :

- 第一个为`技术分享` or `博客` 
- 第二个为**自己的域账号**

### 创建新文章

``` bash
hexo new "新文章"
INFO  Validating config
INFO  Created: ~\source\_posts\新文章.md
```

会生成`blog\source\_posts\新文章.md`

打开编辑，修改title、tag和categories：



然后在下面增加内容，最后修改md文档文件名为合适的文件名（也可以在hexo new "文件名")

**如何插入图片、附件**

如果有图片、附件，可以放到`source/images` 文件夹里

然后通过类似于 `![](/images/image.jpg)` 的方法访问它们。

例如：

```markdown
[下载附件](/images/05%20TDD%E5%AE%9E%E8%B7%B5.pdf) 
```

[下载附件](/images/05%20TDD%E5%AE%9E%E8%B7%B5.pdf) 


更多信息: [Writing](https://hexo.io/docs/writing.html)


**如何插入pdf**

培训课程可以将ppt转为pdf，方便在线预览。

将pdf放到`source/images` 文件夹，然后在markdown中引用。

```markdown
{% pdf /images/your/file.pdf %}
```

比如：

```markdown
{% pdf /images/05%20TDD%E5%AE%9E%E8%B7%B5.pdf %}
```


### 预览

``` bash
$ hexo server
```
或者
``` bash
$ npm run server
```

More info: [Server](https://hexo.io/docs/server.html)

### 提交仓库

验证OK后，git提交到仓库，git ops会自动构建，并发布gitlab pages。