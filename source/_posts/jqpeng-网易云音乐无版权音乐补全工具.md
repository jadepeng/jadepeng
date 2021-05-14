---
title: 网易云音乐无版权音乐补全工具
tags: ["XMusicDownloader","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-08-25 19:05
---
文章作者:jqpeng
原文链接: [网易云音乐无版权音乐补全工具](https://www.cnblogs.com/xiaoqi/p/music163tool.html)

## 缘起

网易云音乐的不少歌曲因为版权下架了，或者变成收费的，导致无法收听，因此需要一个小工具，希望可以从其他来源补全歌曲。

![](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/25/1598352099450.png)

如图所示，不能听的显示为灰色。

之前写的小工具XMusicDownloader([https://github.com/jadepeng/XMusicDownloader](https://github.com/jadepeng/XMusicDownloader)) 可以从多个来源搜索歌曲并下载，因此以这个为基础，可以很快实现需求。

查看本文之前，建议查看[开源工具软件XMusicDownloader——音乐下载神器](https://www.cnblogs.com/xiaoqi/p/xmusicdownloader.html#4277262913).

## 歌曲版权声明

- 该工具本质是**歌曲聚合搜索**，API来自公开网络，`非破解版`，只能下载**各家公开的音乐**，**付费歌曲不能下载**，例如QQ、网易等的收费歌曲不能从QQ\网易源下载，工具的原理是基于聚合实现补全
- **工具和代码仅供技术研究使用**，禁止将本工具用于商业用途，**如产生法律纠纷与本人无关，如有侵权，请联系删除**。


## 工具原理

1. 获取用户歌单，找出无版权和收费歌曲
2. 从QQ、咪咕、百度等源搜索这些歌曲，匹配成功的可以下载
3. 下载后可以手动上传到云盘


### 获取用户歌单

借助[NeteaseCloudMusicApi](https://github.com/wwh1004/NeteaseCloudMusicApi)，可以方便调用云音乐的api。

分析获取到的json，可以发现，包含`noCopyrightRcmd`的是没有版权的，包含`fee`的是收费的，我们可以将这些歌曲提取出来，变为song对象。


    private static List<Song> FetchNoCopyrightSongs(JObject json)
            {
                List<Song> noCopyrightsSongs = new List<Song>();
                foreach (JObject songObj in json["songs"])
                {
                    int id = 0;
                   
                    if (songObj["noCopyrightRcmd"].HasValues || songObj["fee"].Value<int>() == 1)
                    {
                        noCopyrightsSongs.Add(NeteaseProvider.extractSong(ref id, songObj));
                    }
                }
    
                return noCopyrightsSongs;
            }		public static Song extractSong(ref int index, JToken songItem)
            {
                var song = new Song
                {
                    id = (string)songItem["id"],
                    name = (string)songItem["name"],
    
                    album = (string)songItem["al"]["name"],
                    //rate = 128,
                    index = index++,
                    //size = (double)songItem["FileSize"],
                    source = "网易",
                    duration = (double)songItem["dt"] / 1000
                };
    
                song.singer = "";
                foreach (var ar in songItem["ar"])
                {
                    song.singer += ar["name"] + " ";
                }
    
                if (songItem.Contains("privilege") && songItem["privilege"].HasValues)
                {
                    song.rate = ((int)songItem["privilege"]["fl"]) / 1000;
                    var fl = (int)songItem["privilege"]["fl"];
                    if (songItem["h"] != null && fl >= 320000)
                    {
                        song.size = (double)songItem["h"]["size"];
                    }
                    else if (songItem["m"] != null && fl >= 192000)
                    {
                        song.size = (double)songItem["m"]["size"];
                    }
                    else if (songItem["l"] != null)
                    {
                        song.size = (double)songItem["l"]["size"];
                    }
                }
                else
                {
                    song.rate = 128;
                    song.size = 0;
                }
    
                return song;
            }


### 从其他来源获取歌曲

在之前的博文[开源工具软件XMusicDownloader——音乐下载神器](https://www.cnblogs.com/xiaoqi/p/xmusicdownloader.html#4277262913)里，我们有一个聚合的搜索歌曲的方法：


     public List<MergedSong> SearchSongs(string keyword, int page, int pageSize)
            {
                var songs = new List<Song>();
                Providers.AsParallel().ForAll(provider =>
                {
                    var currentSongs = provider.SearchSongs(keyword, page, pageSize);
                    songs.AddRange(currentSongs);
                });
    
                // merge
    
                return songs.GroupBy(s => s.getMergedKey()).Select(g => new MergedSong(g.ToList())).OrderByDescending(s => s.score).ToList();
            }


类似的，匹配也是先搜索，但是要排除网易源，然后根据搜索结果去匹配。搜索的时候，可以将 “歌曲名称 + 歌手名称” 组合用来搜索。


      public MergedSong SearchSong(string singer, string songName, string exceptProvider)
            {	  // search
                var songs = new List<Song>();
                Providers.AsParallel().ForAll(provider =>
                {
                    try
                    {
                        if (provider.Name != exceptProvider)
                        {
                            var currentSongs = provider.SearchSongs(singer + " " + songName, 1, 10);
                            songs.AddRange(currentSongs);
                        }
                    }
                    catch (Exception e)
                    {
    
                    }
                });
    
                // merge
    
                List<MergedSong> mergedSongs = songs.GroupBy(s => s.getMergedKey()).Select(g => new MergedSong(g.ToList())).OrderByDescending(s => s.score).ToList();		 // match
                foreach (MergedSong song in mergedSongs)
                {
                    if (song.singer == singer && song.name == songName)
                    {
                        return song;
                    }
                }
    
                return null;
            }


## 软件界面

软件界面，增加用户、密码输入

![界面](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/25/1598352954630.png)

搜索结果，设置为默认选中：


     List<ListViewItem> listViewItems = new List<ListViewItem>();
                mergedSongs.ForEach(item =>
                {
                    ListViewItem lvi = new ListViewItem();
                    lvi.Text = item.name;
                    lvi.SubItems.Add(item.singer);
                    lvi.SubItems.Add(item.rate + "kb");
                    lvi.SubItems.Add((item.size / (1024 * 1024)).ToString("F2") + "MB");  //将文件大小装换成MB的单位
                    TimeSpan ts = new TimeSpan(0, 0, (int)item.duration); //把秒数换算成分钟数
                    lvi.SubItems.Add(ts.Minutes + ":" + ts.Seconds.ToString("00"));
                    lvi.SubItems.Add(item.source);
                    lvi.Tag = item;
                    lvi.Checked = true; // 默认选中
                    listViewItems.Add(lvi);
                });


搜索出来后，下载可以完全复用之前逻辑。

## 下载歌曲使用

下载后的歌曲，可以通过网易云音乐客户端，上传到云盘，然后批量选中，添加到我喜欢的音乐

![上传](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/25/1598353051852.png)

批量选中后收藏到歌单：

![批量收藏](https://gitee.com/jadepeng/pic/raw/master/pic/2020/8/25/1598353082913.png)

## 工具下载地址

- 可以从github下载  
[https://github.com/jadepeng/music163tool](https://github.com/jadepeng/music163tool)
- 国内gitee地址：  
[https://gitee.com/jadepeng/music163tool](https://gitee.com/jadepeng/music163tool)


欢迎关注作者微信公众号, 一起交流软件开发。

![欢迎关注作者微信公众号](https://img2020.cnblogs.com/blog/38465/202101/38465-20210118185019928-1475690632.png)

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


