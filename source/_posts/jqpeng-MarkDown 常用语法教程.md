---
title: MarkDown 常用语法教程
tags: ["Markdown","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-03-18 22:33
---
文章作者:jqpeng
原文链接: [MarkDown 常用语法教程](https://www.cnblogs.com/xiaoqi/p/6576517.html)

# MarkDown 语法说明




目录

- MarkDown 语法说明
    - 标题
    - 列表
        - 无序列表
        - 有序列表
    - 引用
    - 图片与链接
    - 粗体与斜体
    - 表格
    - 插入代码
    - 分割线





## 标题


    标题1
    ======
    
    标题2
    -----
    
    ## 大标题
    ### 小标题 
    #### 小标题 


## 列表

### 无序列表


    + 列表文本前使用 [减号+空格]
    * 列表文本前使用 [加号+空格]
    - 列表文本前使用 [星号+空格]


- 列表文本前使用 [减号+空格]


- 列表文本前使用 [加号+空格]


- 列表文本前使用 [星号+空格]


### 有序列表


    1. 列表前使用 [数字+空格]
    2. 我们会自动帮你添加数字
    7. 不用担心数字不对，显示的时候我们会自动把这行的 7 纠正为 3


1. 列表前使用 [数字+空格]
2. 我们会自动帮你添加数字
3. 不用担心数字不对，显示的时候我们会自动把这行的 7 纠正为 3


## 引用


    > 引用文本前使用 [大于号+空格]
    > 折行可以不加，新起一行都要加上哦



> 引用文本前使用 [大于号+空格]


## 图片与链接

插入链接与插入图片的语法很像，区别在一个 !号


    图片为：![]()
    链接为：[]()


插入链接：

[Baidu](http://baidu.com)

图片：  
![图片提示](http://ww2.sinaimg.cn/large/6aee7dbbgw1efffa67voyj20ix0ctq3n.jpg)

## 粗体与斜体


    **两个为粗体**


**两个为粗体**


    *单个为斜体*


*单个为斜体*

## 表格

使用html


    <table>
        <tr>
            <td>Foo</td>
        </tr>
    </table>


markdown语法


    name | age
    ---- | ---
    LearnShare    | 12
    Mike |  32



| name | age |
| --- | --- |
| LearnShare | 12 |
| Mike | 32 |


对齐：


    | Tables        |      Are      |  Cool |
    |:--------------|:-------------:|------:|
    | col 3 is      | right-aligned | $1600 |
    | col 2 is      |   centered    |   $12 |
    | zebra stripes |   are neat    |    $1 |



| Tables | Are | Cool |
| --- | --- | --- |
| col 3 is | right-aligned | $1600 |
| col 2 is | centered | $12 |
| zebra stripes | are neat | $1 |


## 插入代码

用两个 ``` 包裹一段代码，并指定一种语言


    ```javascript
    $(document).ready(function () {
        alert('hello world');
    });
    ```



    $(document).ready(function () {
        alert('hello world');
    });


支持常用的语言，下面试下go语言


    package main
    
    import (
        "fmt"
        "math"
        "runtime"
    )
    
    func sqrt(x float64) string {
        if x < 0 {
            return sqrt(-x) + "i"
        }
        return fmt.Sprint(math.Sqrt(x))
    } 


## 分割线

分割线的语法只需要三个 \* 号，


    ***


* * *

