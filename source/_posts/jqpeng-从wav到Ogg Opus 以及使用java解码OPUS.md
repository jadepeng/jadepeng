---
title: 从wav到Ogg Opus 以及使用java解码OPUS
tags: ["OPUS","jqpeng"]
categories: ["博客","jqpeng"]
date: 2021-04-13 10:09
---
文章作者:jqpeng
原文链接: [从wav到Ogg Opus 以及使用java解码OPUS](https://www.cnblogs.com/xiaoqi/p/ogg-opus.html)

## PCM

自然界中的声音非常复杂，波形极其复杂，通常我们采用的是脉冲代码调制编码，即PCM编码。PCM通过抽样、量化、编码三个步骤将连续变化的模拟信号转换为数字编码。

**采样率**

采样频率，也称为采样速度或者采样率，定义了每秒从连续信号中提取并组成离散信号的采样个数，它用赫兹（Hz）来表示。采样频率的倒数是采样周期或者叫作采样时间，它是采样之间的时间间隔。通俗的讲采样频率是指计算机每秒钟采集多少个信号样本。

工业界常用的16K，就是1s有16000个采样点。

## WAV

PCM是原始语音，依据采样率的定义，我们知道要播放PCM，需要知道采样率，因此需要一个文件格式可以封装PCM，`wav`就是微软公司专门为Windows开发的一种标准数字音频文件，该文件能记录各种单声道或立体声的声音信息。

![WAV格式](https://gitee.com/jadepeng/pic/raw/master/pic/2021/4/13/1618277649258.png)

wav文件前44个字节，定义了采样率，channel等参数，播放器通过这个数据就可以播放PCM数据了。

## MP3

`wav` 很好的解决了PCM播放的问题，但是PCM实在是太大了，因此出现了`mp3`等音频格式，通过一定的压缩算法压缩语音，以便于互联网传输分享。

## Ogg 与 Opus

随着音视频应用的越来越广泛，工业界有了越来越多的编解码器，比如`Speek`,`Opus`

Opus编解码器是专门设计用于互联网的交互式语音和音频传输。它是由IETF的编解码器工作组设计的，合并了Skype的SILK和Xiph. Org的CELT技术。

![OPUS](https://gitee.com/jadepeng/pic/raw/master/pic/2021/4/13/1618278091526.png)

OPUS编解码

[https://github.com/lostromb/concentus](https://github.com/lostromb/concentus) 是一个纯java库，可以编解码OPUS。

OPUS一般是分帧编码，比如一个320采样点（640字节）的数据，编码后为70多个字节，和PCM一样，编码后的OPUS不能直接播放：

- 无法从文件本身获取音频的元数据(采样率,声道数,码率等)
- 缺少帧分隔标识,无法从连续的文件流中分隔帧(尤其是vbr情况)


伴随着HTML5的发展，出现了OGG媒体文件格式，Ogg是一个自由且开放标准的多媒体文件格式，由Xiph.Org基金会所维护。Ogg格式并不受到软件专利的限制，并设计用于有效率地流媒体和处理高质量的数字多媒体。“Ogg”意指一种文件格式，可以纳入各式各样自由和开放源代码的编解码器，包含音效、视频、文字（像字幕）与元数据的处理。

OGG音频


| 压缩类型 | 格式 | 说明 |
| --- | --- | --- |
| 有损 | Speek | 以低比特率处理语音数据（〜2.1-32 kbit / s /通道） |
|  | Vorbis | 处理中高级可变比特率（每通道≈16-500kbit / s）的一般音频数据 |
|  | Opus： | 以低和高可变比特率处理语音，音乐和通用音频（每通道≈6-510kbit / s） |
| 无损 | FLAC | 处理文件和高保真音频数据 |
| 未压缩 | OggPCM | 处理未压缩的PCM音频,与WAV类似 |


参考: [https://juejin.cn/post/6844904016254599175](https://juejin.cn/post/6844904016254599175)

借博主的图:

![OGG封装](https://gitee.com/jadepeng/pic/raw/master/pic/2021/4/13/1618278801473.png)

## java 解码OPUS文件

通过ffmpeg可以轻松的将wav转换为opus文件，本质是一个ogg封装的opus，我们可以通过`vorbis-java` 来读取opus文件。

通过OpusInfoTool，可以打印OPUS文件信息：


    Processing file "C:\Users\jqpeng\Downloads\opus\wav16k.opus"
    
    Opus Headers:
      Version: 1
      Vendor: Lavf58.27.103
      Channels: 1
      Rate: 16000Hz
      Pre-Skip: 104
      Playback Gain: 0dB
    
    User Comments:
      encoder=Lavc58.53.100 libopus
    
    Logical stream 81c1bbc0 (-2118009920) completed
    
    Opus Audio:
      Total Data Packets: 579
      Total Data Length: 41406
      Audio Length Seconds: 11.564333333333334
      Audio Length: 00:00:11.56
      Packet duration:     20.0ms (max),     20.0ms (avg),     20.0ms (min)
      Page duration:     1000.0ms (max),    965.0ms (avg),    580.0ms (min)
      Total data length: 41406 (overhead: 2.34%)
      Playback length: 00:00:11.56
      Average bitrate: 28.70 kb/s, w/o overhead: 27.97 kb/s
    


再借助`concentus `，我们来解码OPUS文件为PCM文件。


    
    public void testDecode() throws IOException, OpusException {
            FileInputStream fs = new FileInputStream("\\wav16k.opus");
            OggFile ogg = new OggFile(fs);
            OpusFile of = new OpusFile(ogg);
            OpusAudioData ad = null;
    
            System.out.println(of.getInfo().getSampleRate());
            System.out.println(of.getInfo().getNumChannels());
    
            OpusDecoder decoder = new OpusDecoder(of.getInfo().getSampleRate(),
                                                  of.getInfo().getNumChannels());
            System.out.println(of.getTags());
            FileOutputStream fileOut = new FileOutputStream("wav16k.pcm");
            // 
            byte[] data_packet = new byte[of.getInfo().getSampleRate()];
            int samples = 0;
            while ((ad = of.getNextAudioPacket()) != null) {
                // NOTE: samplesDecoded 是decode出来的short个数，byte需要*2
                int samplesDecoded =
                        decoder.decode(ad.getData(), 0, ad.getData().length
                                , data_packet, 0, of.getInfo().getSampleRate() / 2,
                                       false);
    
                fileOut.write(data_packet, 0, samplesDecoded * 2);
                samples += samplesDecoded;
            }
    
            System.out.println("samples: " + samples);
            System.out.println("durationSeconds: " + (samples / 16000f));
            fileOut.close();
        }


* * *

感谢您的认真阅读。

如果你觉得有帮助，欢迎点赞支持！

不定期分享软件开发经验，欢迎关注作者, 一起交流软件开发：

