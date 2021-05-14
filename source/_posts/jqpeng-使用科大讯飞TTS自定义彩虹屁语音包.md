---
title: 使用科大讯飞TTS自定义彩虹屁语音包
tags: ["rainbow-fart-tts","idea-rainbow-fart","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-06-29 13:16
---
文章作者:jqpeng
原文链接: [使用科大讯飞TTS自定义彩虹屁语音包](https://www.cnblogs.com/xiaoqi/p/rainbow-fart-tts.html)

# rainbow-fart-tts

彩虹屁插件红了后，你是否想生成定义语音包呢？本文给出一个解决方案，使用科大讯飞的TTS生成彩虹屁语音包，你可以完全自定义文本，自定义发音人哦！

开源地址： [https://github.com/jadepeng/rainbow-fart-tts](https://github.com/jadepeng/rainbow-fart-tts)

## 修改文本

打开`manifest.json`, 修改text，可以按需增加keyword和对应text


     {
          "keywords": [
            "if",
            "else"
          ],
          "text": [
            "你就是因为想太多如果，所以才交不到女朋友吧？",
            " 别试了，我的可爱不需要用 if 来判断！",
            " 人生没有那么多如果，有没有比编程更容易呢？"
          ]
     }


## 获取开发者账号

到`https://www.xfyun.cn/` 注册账号，创建应用，然后开通语音合成,可以开通免费包,好使的话可以购买套餐哦。

![讯飞tts免费包](https://gitee.com/jadepeng/pic/raw/master/pic/2020/6/29/1593406803680.png)

然后到控制面板，查看appid等信息：

![appid](https://gitee.com/jadepeng/pic/raw/master/pic/2020/6/29/1593406879615.png)

然后打开VoicePackageMakerApp，将对应的信息填入:


    public class VoicePackageMakerApp {
        private static final String hostUrl = "https://tts-api.xfyun.cn/v2/tts";
    
        // 到控制台-语音合成页面获取
        private static final String APPID = "";
    
        // 到控制台-语音合成页面获取
        private static final String API_SECRET = "";
    
        //到控制台-语音合成页面获取
        private static final String API_KEY = "";


## 选取发音人

讯飞开放平台的在线语音合成有很多发音人，可以到[https://www.xfyun.cn/services/online\_tts](https://www.xfyun.cn/services/online_tts) 查看：

![发音人](https://gitee.com/jadepeng/pic/raw/master/pic/2020/6/29/1593407032570.png)

选择自己心仪的，然后到控制面板开通权限：

![开通发音人](https://gitee.com/jadepeng/pic/raw/master/pic/2020/6/29/1593407082655.png)

比如我选择的讯飞玲姐姐（志林姐姐），发音人是`x_xiaoling`，修改代码：


    public class VoicePackageMakerApp {
    
    
        // 默认发音人
        private static final String DEFAULT_VCN = "x_xiaoling";
    


## 生成和使用语音包

上面步骤做完后，直接运行`VoicePackageMakerApp`即可，然后在`voicePackages`目录下会生成`x_xiaoling`文件夹，里面是合成的语音包，可以给各个版本的彩虹屁插件使用。

![使用语音包](https://gitee.com/jadepeng/pic/raw/master/pic/2020/6/29/1593407303218.png)

本文开源地址： [https://github.com/jadepeng/rainbow-fart-tts](https://github.com/jadepeng/rainbow-fart-tts)

IDE版本的语音包请参见：[https://github.com/jadepeng/idea-rainbow-fart](https://github.com/jadepeng/idea-rainbow-fart)

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


