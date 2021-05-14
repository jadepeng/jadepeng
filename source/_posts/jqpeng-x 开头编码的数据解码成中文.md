---
title: x 开头编码的数据解码成中文
tags: ["python","jqpeng"]
categories: ["博客","jqpeng"]
date: 2016-01-05 11:38
---
文章作者:jqpeng
原文链接: [x 开头编码的数据解码成中文](https://www.cnblogs.com/xiaoqi/p/5101795.html)

## Python 解码

在python里，直接decode('utf-8')即可



    >>> "\xE5\x85\x84\xE5\xBC\x9F\xE9\x9A\xBE\xE5\xBD\x93 \xE6\x9D\x9C\xE6\xAD\x8C".decode('utf-8')
    u'\u5144\u5f1f\u96be\u5f53 \u675c\u6b4c'
    >>> print "\xE5\x85\x84\xE5\xBC\x9F\xE9\x9A\xBE\xE5\xBD\x93 \xE6\x9D\x9C\xE6\xAD\x8C".decode('utf-8')
    兄弟难当 杜歌
    >>>





## Java 解码



     public static void main(String[] args) throws UnsupportedEncodingException {
            String utf8String = "\\xE5\\x85\\x84\\xE5\\xBC\\x9F\\xE9\\x9A\\xBE\\xE5\\xBD\\x93 \\xE6\\x9D\\x9C\\xE6\\xAD\\x8C";
            System.out.println(decodeUTF8Str(utf8String));
        }
    
        public static String decodeUTF8Str(String xStr) throws UnsupportedEncodingException {
            return URLDecoder.decode(xStr.replaceAll("\\\\x", "%"), "utf-8");
        }





以上代码输出:



    兄弟难当 杜歌
    
    Process finished with exit code 0





## 深入理解：　　

要了解原理，推荐阅读http://www.ruanyifeng.com/2007/10/ascii\_unicode\_and\_utf-8.html

UTF-8是unicode编码的一种落地方案：

Unicode符号范围 | UTF-8编码方式  
(十六进制) | （二进制）  
--------------------+---------------------------------------------  
0000 0000-0000 007F | 0xxxxxxx  
0000 0080-0000 07FF | 110xxxxx 10xxxxxx  
0000 0800-0000 FFFF | 1110xxxx 10xxxxxx 10xxxxxx  
0001 0000-0010 FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx

\x对应的是UTF-8编码的数据，通过转化规则可以转换为Unicode编码，就能得到对应的汉字，转换规则很简单，先将\x去掉，转换为数字，然后进行对应的位移操作即可，需要注意的是先要判断utf-8的位数.

可以通过下面的scala代码深入了解：



     val pattern = """(\d+\.\d+\.\d+\.\d+) \- (\S+) (\S+) \[([^\]]+)\] \"(\w+) (\S+) \S+\" (\S+) (\S+) \"([^\"]+)\" \"([^\"]+)\" \"([^\"]+)\" \"([^\"]+)""".r
      val decodeDataPattern = """(\\x([0-9A-Z]){2})+""".r
      def decodeUtf8(utf8Str:String):String={
        var data =   decodeDataPattern.replaceAllIn(utf8Str, m=>{
            var item = decodeXdata(m.toString())
            item
         }) 
         return data
       }
         
       def decodeXdata(utf8Str:String):String={
         var arr = utf8Str.split("\\\\x")
         var result = new StringBuilder()
         var isMatchEnd = true
         var matchIndex = 0
         var currentWordLength = 0
         var current = 0
         var e0=0xe0;
         
         for(item <-arr){
            var str = item.trim
            if(str.length()>0){
               var currentCode =  Integer.parseInt(str, 16);
               if(isMatchEnd){
                 isMatchEnd = false
                 var and = currentCode & e0;
                 if(and == 0xe0){
                    matchIndex = 1;
                    currentWordLength = 3;
                    current =  (currentCode & 0x1f) <<12  // 3位编码的
                 }else if(and==96){
                    matchIndex = 1;
                    currentWordLength = 2;
                    current =  (currentCode & 0x1f) <<6 // 2位编码的
                 }else{
                   current = currentCode  // 1位编码的
                 }
              }else{
                matchIndex = matchIndex+1;
                if(matchIndex == 2)
                {
                  current+=(currentCode & 0x3f) <<6
                }else{
                   current+=(currentCode & 0x3f) 
                }
              }
               if(matchIndex==currentWordLength){
                   var hex = Integer.toHexString(current)
                   hex = if(hex.length()<4) "\\u00"+hex else "\\u"+hex  //补0
                   result.append(new String(StringEscapeUtils.unescapeJava(hex).getBytes,"utf-8")) 
                   current = 0
                   matchIndex=0
                   isMatchEnd = true
               }
            }
         }
         
         return result.toString()
       }





Javascript解码：

# [Javascript \x 反斜杠x 16进制 编解码](https://www.cnblogs.com/xiaoqi/p/js-x-encode-decode.html)

