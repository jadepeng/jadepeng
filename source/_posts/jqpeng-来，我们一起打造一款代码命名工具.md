---
title: 来，我们一起打造一款代码命名工具
tags: ["代码命名","开源","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-08-31 20:34
---
文章作者:jqpeng
原文链接: [来，我们一起打造一款代码命名工具](https://www.cnblogs.com/xiaoqi/p/code-naming-tool.html)

你是否还在为代码命名而纠结不已？


> here are only two hard things in Computer Science: cache invalidation and naming things.-- Phil Karlton


![代码命名](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/31/1598875275591.png)

那么如何更好的命名呢？ 是否有好的工具可以支持我们命名呢？网上搜索一圈没有发现满意的，于是自己动手丰衣足食，[https://jadepeng.gitee.io/code-naming-tool/](https://jadepeng.gitee.io/code-naming-tool/)。

使用方法： 打开网页后，在中文输入框中输入 中文命名，然后回车即可。也可以直接在英文输入框输入英文，搜索候选。

## 现有的工具

unbug.github.io/codelf/ 提供了一个选择，作者先调用有道、百度等翻译，然后调用searchcode搜索代码，从搜索的代码中提取变量名。

![codeif](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/31/1598875386957.png)

界面做的很酷，但是推荐出来的变量名称质量参差不齐，失去了参考意义。

## 新的思路

我们常说以史为鉴，换一个思路，我们可以从优秀的开源库中去吸收他们命名的经验，看看他们是如何命名的，来供我们参考。

实现思路：  
 1. 从spring、apache等代码库，读取变量、方法、类名称  
 2. 根据关键词匹配出候选命名  
 3. 候选结果排序

![工具](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/31/1598875225454.png)

## 获取优秀命名

要获取命名，首先想到的是读取代码库，需要先下载代码，然后解析  ———— 工作量巨大，PASS。

那怎么做呢，换个角度，可以通过java的反射来实现。

首先添加一个辅助库：


    <dependency>
                <groupId>org.reflections</groupId>
                <artifactId>reflections</artifactId>
                <version>0.9.12</version>
            </dependency>


然后初始化Reflections,FilterBuilder可以用来过滤类库，我们设置"org","javax","com","io", 基本上囊库了主要的开源类库，比如spring，apache等.


     List<ClassLoader> classLoadersList = new LinkedList<ClassLoader>();
            classLoadersList.add(ClasspathHelper.contextClassLoader());
            classLoadersList.add(ClasspathHelper.staticClassLoader());
    
            Reflections reflections = new Reflections(new ConfigurationBuilder()
                    .setScanners(new SubTypesScanner(false), new ResourcesScanner())
                    .setUrls(ClasspathHelper.forClassLoader(classLoadersList.toArray(new ClassLoader[0])))
                    .filterInputsBy(new FilterBuilder().includePackage("org","javax","com","io")));


然后，可以通过`reflections.getSubTypesOf(Object.class);`来获取相关的class了,注意，我们初始化一个` Map<String, Integer> name2count = new HashMap<String, Integer>();`用来存储代码命名以及对应的出现次数。


    Set<Class<? extends Object>> allClasses =
                    reflections.getSubTypesOf(Object.class);
            Map<String, Integer> name2count = new HashMap<String, Integer>();
            for (Class<?> clazz : allClasses) {
                System.out.println(clazz.getName());
                try {
                    appendToNameMap(name2count, clazz.getSimpleName());
    
                    Field[] fields = clazz.getDeclaredFields();
                    for (Field field : fields) {
                        String name = field.getName();
                        appendToNameMap(name2count, name);
                    }
                    Method[] methods = clazz.getMethods();
                    for (Method method : methods) {
                        String name = method.getName();
                        appendToNameMap(name2count, name);
                        // parameters
                        Parameter[] parameters =  method.getParameters();
                        for (Parameter param : parameters) {
                            name = param.getName();
                            appendToNameMap(name2count, name);
                        }
                    }
                }catch(Throwable t)
                { }


其中`appendToNameMap`:


     private static void appendToNameMap(Map<String, Integer> name2count, String name) {
            // filter
            if(name.contains("-") || name.contains("_")|| name.contains("$")){
                return;
            }
    
            if (!name2count.containsKey(name)) {
                name2count.put(name, 1);
            } else {
                name2count.put(name, name2count.get(name) +1);
            }
        }


最后把结果存储到文件，作为我们的资源。


    FileUtils.writeAllText(JSON.toJSONString(name2count), new File("name2count.txt"));


可以到`https://gitee.com/jadepeng/code-naming-tool/blob/master/vars.js`查看结果。

## 命名推荐

命名推荐，还是遵循，先翻译，然后根据翻译结果搜索并召回。

其中翻译直接调用网易有道的，但是搜索如何搞定呢？

最简单的方法，肯定是分词，然后建立索引，lucene是标配。但是上lucene就要上服务器，PASS!

我们来找一个浏览器端的lucene，google 后选定`flexsearch`.

![flexsearch](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/31/1598876259412.png)

flexsearch github上有6.5k star，因此优先选择。

下面来看具体的实现。

### 建立索引

初始化FlexSearch，然后将之前获取的代码命名建立索引。


     var index = new FlexSearch({
                encode: "advanced",
                tokenize: "reverse",
                suggest: true,
                cache: true
            })
            var data = []
            var i = 0
            for (var name in names) {
                var indexName = name.replace(/([A-Z])/g, " $1")
                data[i] = {
                    "name": name,
                    "count": names[name]
                }
                index.add(i++, indexName)
            }
    


这里有个小技巧，`name.replace(/([A-Z])/g, " $1")` 可以将驼峰命名拆分成单词。  
 同时data数组会保存所有的命名和响应的出现次数。

### 搜索候选

先翻译，然后将翻译结果给FlexSearch搜索。


    function searchFromIndex(value) {
            var results = index.search(value, 25)
        
            results = results.map(function (i) {
                return data[i]
            })
        
            results = results.sort(function (a, b) {
                return b.count - a.count
            })
            return results
        }


先搜索，出来的结果是data中的index序号，转换为list对象，然后按照count倒排。

tips： 理论上，翻译的结果可以去除一些停用词，搜索效果应该更好，这里先放着。

### 显示结果

对结果进行格式化：


    function formatSuggestion(item){
        return `${item.name} <span class='tips'>代码库共出现${item.count}次 (相关搜索： <a target='_blank' href='https://unbug.github.io/codelf/#${item.name}'>codelf</a> &nbsp; <a target='_blank' href='https://searchcode.com/?q=${item.name}&lan=23'>searchcode</a>)</span>`;
    }


增加到codelf 和 searchcode的链接，显示结果如下：

![搜索结果](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/31/1598877282579.png)

## 开源地址

- github:  [https://github.com/jadepeng/code-naming-tool](https://github.com/jadepeng/code-naming-tool)
- gitee:  [https://gitee.com/jadepeng/code-naming-tool](https://gitee.com/jadepeng/code-naming-tool)


命名工具地址： [https://jadepeng.gitee.io/code-naming-tool/](https://jadepeng.gitee.io/code-naming-tool/)

欢迎大家体验使用，欢迎fork并贡献代码。

## 后续展望

当前仅仅用到了翻译+搜索，还有很多可以优化的地方：

- 搜索去停用词
- 从文本语义相似度层面去推荐
- 专业术语支持


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


