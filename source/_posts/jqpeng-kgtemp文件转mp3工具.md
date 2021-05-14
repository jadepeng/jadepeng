---
title: kgtemp文件转mp3工具
tags: ["kgtemp","c#","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-12-22 13:24
---
文章作者:jqpeng
原文链接: [kgtemp文件转mp3工具](https://www.cnblogs.com/xiaoqi/p/8085563.html)

kgtemp文件是酷我音乐软件的缓存文件，本文从技术层面探讨如何解密该文件为mp3文件，并通过读取ID3信息来重命名。

备注：针对老版本的酷我音乐生效，新版本不支持！！！

## kgtemp解密

kgtemp文件前1024个字节是固定的包头信息，解密方案详细可以参见([http://www.cnblogs.com/KMBlog/p/6877752.html](http://www.cnblogs.com/KMBlog/p/6877752.html))：


    class Program{	static void Main(string[] args)	{
    		byte[] key={0xAC,0xEC,0xDF,0x57};		using (var input = new FileStream(@"E:\KuGou\Temp\236909b6016c6e98365e5225f488dd7a.kgtemp", FileMode.Open, FileAccess.Read))		{			var output = File.OpenWrite(@"d:\test.mp3");//输出文件			input.Seek(1024, SeekOrigin.Begin);//跳过1024字节的包头			byte[] buffer = new byte[key.Length];			int length;			while((length=input.Read(buffer,0,buffer.Length))>0)			{				for(int i=0;i<length;i++)				{					var k = key[i];					var kh = k >> 4;					var kl = k & 0xf;					var b = buffer[i];					var low = b & 0xf ^ kl;//解密后的低4位					var high = (b >> 4) ^ kh ^ low & 0xf;//解密后的高4位					buffer[i] = (byte)(high << 4 | low);				}				output.Write(buffer, 0, length);			}			output.Close();		}		Console.WriteLine("按任意键退出...");		Console.ReadKey();	}}


这样解密出来就是mp3文件了

## 读取ID3信息

解密出来的文件还需要手动命名，不是很方便，可以读取ID3V1信息重命名文件。  
 ID3V1比较简单，它是存放在MP3文件的末尾，用16进制的编辑器打开一个MP3文件，查看其末尾的128个顺序存放字节，数据结构定义如下：  
 char Header[3](https://raw.githubusercontent.com/jadepeng/blogpic/master/pic/2018/1513919861693.jpg "1513919861693");    /*标签头必须是"TAG"否则认为没有标签*/  
 char Title[30];    /*标题*/  
 char Artist[30];   /*作者*/  
 char Album[30];    /*专集*/  
 char Year[4](https://raw.githubusercontent.com/jadepeng/blogpic/master/pic/2018/1513920067729.jpg "1513920067729");    /*出品年代*/  
 char Comment[30];   /*备注*/  
 char Genre;    /*类型，流派*/

解析代码比较简单，注意中文歌曲用GBK编码就可以了：


      private static Mp3Info FormatMp3Info(byte[] Info, System.Text.Encoding Encoding)	{		Mp3Info myMp3Info = new Mp3Info();		string str = null;		int i;		int position = 0主要代码jia，; //循环的起始值		int currentIndex = 0; //Info的当前索引值
    		//获取TAG标识		for (i = currentIndex; i < currentIndex + 3; i++)		{			str = str + (char)Info[i];			position++;		}		currentIndex = position;		myMp3Info.identify = str;
    		//获取歌名		str = null;		byte[] bytTitle = new byte[30]; //将歌名部分读到一个单独的数组中		int j = 0;		for (i = currentIndex; i < currentIndex + 30; i++)		{			bytTitle[j] = Info[i];			position++;			j++;		}		currentIndex = position;		myMp3Info.Title = ByteToString(bytTitle, Encoding);
    		//获取歌手名		str = null;		j = 0;		byte[] bytArtist = new byte[30]; //将歌手名部分读到一个单独的数组中		for (i = currentIndex; i < currentIndex + 30; i++)		{			bytArtist[j] = Info[i];			position++;			j++;		}		currentIndex = position;		myMp3Info.Artist = ByteToString(bytArtist, Encoding);
    		//获取唱片名		str = null;		j = 0;		byte[] bytAlbum = new byte[30]; //将唱片名部分读到一个单独的数组中		for (i = currentIndex; i < currentIndex + 30; i++)		{			bytAlbum[j] = Info[i];			position++;			j++;		}		currentIndex = position;		myMp3Info.Album = ByteToString(bytAlbum, Encoding);
    		//获取年		str = null;		j = 0;		byte[] bytYear = new byte[4]; //将年部分读到一个单独的数组中		for (i = currentIndex; i < currentIndex + 4; i++)		{			bytYear[j] = Info[i];			position++;			j++;		}		currentIndex = position;		myMp3Info.Year = ByteToString(bytYear, Encoding);
    		//获取注释		str = null;		j = 0;		byte[] bytComment = new byte[28]; //将注释部分读到一个单独的数组中		for (i = currentIndex; i < currentIndex + 25; i++)		{			bytComment[j] = Info[i];			position++;			j++;		}		currentIndex = position;		myMp3Info.Comment = ByteToString(bytComment, Encoding);
    		//以下获取保留位		myMp3Info.reserved1 = (char)Info[++position];		myMp3Info.reserved2 = (char)Info[++position];		myMp3Info.reserved3 = (char)Info[++position];
    		//		return myMp3Info;	}


## 转换小工具

写了一个小工具，来进行转换

![装换工具](https://raw.githubusercontent.com/jadepeng/blogpic/master/pic/2018/1513919861693.jpg "1513919861693")

下载地址：[https://pan.baidu.com/s/1o7FIsPk](https://pan.baidu.com/s/1o7FIsPk)

PS:上面只读取了IDV1，部分歌曲可能不存在  
 可以下载@缤纷 提供的程序，增加了ID3V2的支持：  
[https://files.cnblogs.com/files/gxlxzys/kgtemp文件转mp3工具.zip](https://files.cnblogs.com/files/gxlxzys/kgtemp%E6%96%87%E4%BB%B6%E8%BD%ACmp3%E5%B7%A5%E5%85%B7.zip)

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


