---
title: HTML5录音控件
tags: ["JavaScript","jqpeng"]
categories: ["博客","jqpeng"]
date: 2017-06-12 17:07
---
文章作者:jqpeng
原文链接: [HTML5录音控件](https://www.cnblogs.com/xiaoqi/p/6993912.html)

最近的项目又需要用到录音，年前有过调研，再次翻出来使用，这里做一个记录。

HTML5提供了录音支持，因此可以方便使用HTML5来录音，来实现录音、语音识别等功能，语音开发必备。但是ES标准提供的API并不人性化，不方便使用，并且不提供保存为wav的功能，开发起来费劲啊！！

github寻找轮子，发现[Recorder.js](https://github.com/mattdiamond/Recorderjs)，基本上可以满足需求了，良好的封装，支持导出wav，但是存在：

- wav采样率不可调整
- recorder创建麻烦，需要自己初始化getUserMedia
- 无实时数据回调，不方便绘制波形
- 。。。


## 改造轮子

### 创建recorder工具方法

提供创建recorder工具函数，封装audio接口：


    static createRecorder(callback,config){
            window.AudioContext = window.AudioContext || window.webkitAudioContext;
            window.URL = window.URL || window.webkitURL;
            navigator.getUserMedia = navigator.getUserMedia || navigator.webkitGetUserMedia || navigator.mozGetUserMedia || navigator.msGetUserMedia;
            
            if (navigator.getUserMedia) {
                navigator.getUserMedia(
                    { audio: true } //只启用音频
                    , function (stream) {
                        var audio_context = new AudioContext;
                        var input = audio_context.createMediaStreamSource(stream);
                        var rec = new Recorder(input, config);
                        callback(rec);
                    }
                    , function (error) {
                        switch (error.code || error.name) {
                            case 'PERMISSION_DENIED':
                            case 'PermissionDeniedError':
                                throwError('用户拒绝提供信息。');
                                break;
                            case 'NOT_SUPPORTED_ERROR':
                            case 'NotSupportedError':
                                throwError('浏览器不支持硬件设备。');
                                break;
                            case 'MANDATORY_UNSATISFIED_ERROR':
                            case 'MandatoryUnsatisfiedError':
                                throwError('无法发现指定的硬件设备。');
                                break;
                            default:
                                throwError('无法打开麦克风。异常信息:' + (error.code || error.name));
                                break;
                        }
                    });
            } else {
                throwError('当前浏览器不支持录音功能。'); return;
            }
        }


### 采样率

H5录制的默认是44k的，文件大，不方便传输，因此需要进行重新采样，一般采用插值取点方法：

以下代码主要来自stackoverflow：


                 /**
                 * 转换采样率
                 * @param data
                 * @param newSampleRate 目标采样率
                 * @param oldSampleRate 原始数据采样率
                 * @returns {any[]|Array}
                 */
                function interpolateArray(data, newSampleRate, oldSampleRate) {
                    var fitCount = Math.round(data.length * (newSampleRate / oldSampleRate));
                    var newData = new Array();
                    var springFactor = new Number((data.length - 1) / (fitCount - 1));
                    newData[0] = data[0]; // for new allocation
                    for (var i = 1; i < fitCount - 1; i++) {
                        var tmp = i * springFactor;
                        var before = new Number(Math.floor(tmp)).toFixed();
                        var after = new Number(Math.ceil(tmp)).toFixed();
                        var atPoint = tmp - before;
                        newData[i] = this.linearInterpolate(data[before], data[after], atPoint);
                    }
                    newData[fitCount - 1] = data[data.length - 1]; // for new allocation
                    return newData;
                }
    
                function linearInterpolate(before, after, atPoint) {
                    return before + (after - before) * atPoint;
                }


修改导出wav函数exportWAV，增加采样率选项：


                /**
                 * 导出wav
                 * @param type
                 * @param desiredSamplingRate 期望的采样率
                 */
                function exportWAV(type,desiredSamplingRate) {
                    // 默认为16k
                    desiredSamplingRate = desiredSamplingRate || 16000;
                    var buffers = [];
                    for (var channel = 0; channel < numChannels; channel++) {
                        var buffer = mergeBuffers(recBuffers[channel], recLength);
                        // 需要转换采样率
                        if (desiredSamplingRate!=sampleRate) {
                            // 插值去点
                            buffer = interpolateArray(buffer, desiredSamplingRate, sampleRate);
                        }
                        buffers.push(buffer);
                    }
                    var interleaved = numChannels === 2 ? interleave(buffers[0], buffers[1]) : buffers[0];
                    var dataview = encodeWAV(interleaved,desiredSamplingRate);
                    var audioBlob = new Blob([dataview], { type: type });
                    self.postMessage({ command: 'exportWAV', data: audioBlob });
                }


### 实时录音数据回调

为了方便绘制音量、波形图，需要获取到实时数据：

config新增一个回调函数onaudioprocess：


      config = {
            bufferLen: 4096,
            numChannels: 1, // 默认单声道
            mimeType: 'audio/wav',
            onaudioprocess:null
        };


修改录音数据处理函数：


            this.node.onaudioprocess = (e) => {
                if (!this.recording) return;
                var buffer = [];
    
                for (var channel = 0; channel < this.config.numChannels; channel++) {
                    buffer.push(e.inputBuffer.getChannelData(channel));
                }
    
                // 发送给worker
                this.worker.postMessage({
                    command: 'record',
                    buffer: buffer
                });
    
                // 数据回调
                if(this.config.onaudioprocess){
                    this.config.onaudioprocess(buffer[0]);
                }
            };


这样，在创建recorder时，配置onaudioprocess就可以获取到实时数据了

### 实时数据编码

编码计算耗时，需要放到worker执行：

接口函数新增encode，发送消息给worker，让worker执行：


        encode(cb,buffer,sampleRate) {
            cb = cb || this.config.callback;
            if (!cb) throw new Error('Callback not set');
            this.callbacks.encode.push(cb);
            this.worker.postMessage({ command: 'encode',buffer:buffer,sampleRate:sampleRate});
        }


worker里新增encode函数，处理encode请求，完成后执行回调


    
     self.onmessage = function (e) {
                    switch (e.data.command) {
    
                        case 'encode':
                            encode(e.data.buffer,e.data.sampleRate);
                            break;
    
                    }
                };		
        encode(cb,buffer,sampleRate) {
            cb = cb || this.config.callback;
            if (!cb) throw new Error('Callback not set');
            this.callbacks.encode.push(cb);
            this.worker.postMessage({ command: 'encode',buffer:buffer,sampleRate:sampleRate});
        }


### wav上传

增加一个上传函数：


         exportWAVAndUpload(url, callback) {
            var _url = url;
            exportWAV(function(blob){
                var fd = new FormData();
                fd.append("audioData", blob);
                var xhr = new XMLHttpRequest();
                if (callback) {
                    xhr.upload.addEventListener("progress", function (e) {
                        callback('uploading', e);
                    }, false);
                    xhr.addEventListener("load", function (e) {
                        callback('ok', e);
                    }, false);
                    xhr.addEventListener("error", function (e) {
                        callback('error', e);
                    }, false);
                    xhr.addEventListener("abort", function (e) {
                        callback('cancel', e);
                    }, false);
                }
                xhr.open("POST", url);
                xhr.send(fd);
            })     
        }


### 完整代码

=[点击下载](http://files.cnblogs.com/files/xiaoqi/recorder.js)

## 发现新轮子

今天再次看这个项目，发现这个项目已经不维护了，


> Note: This repository is not being actively maintained due to lack of time and interest. If you maintain or know of a good fork, please let me know so I can direct future visitors to it. In the meantime, if this library isn't working, you can find a list of popular forks here: [http://forked.yannick.io/mattdiamond/recorderjs](http://forked.yannick.io/mattdiamond/recorderjs).


作者推荐[https://github.com/chris-rudmin/Recorderjs](https://github.com/chris-rudmin/Recorderjs)，提供更多的功能：

- **bitRate** (*optional*) Specifies the target bitrate in bits/sec. The encoder selects an application-specific default when this is not specified.
- **bufferLength** - (*optional*) The length of the buffer that the internal JavaScriptNode uses to capture the audio. Can be tweaked if experiencing performance issues. Defaults to `4096`.
- **encoderApplication** - (*optional*) Specifies the encoder application. Supported values are `2048` - Voice, `2049` - Full Band Audio, `2051` - Restricted Low Delay. Defaults to `2049`.
- **encoderComplexity** - (*optional*) Value between 0 and 10 which determines latency and processing for resampling. `0` is fastest with lowest complexity. `10` is slowest with highest complexity. The encoder selects a default when this is not specified.
- **encoderFrameSize** (*optional*) Specifies the frame size in ms used for encoding. Defaults to `20`.
- **encoderPath** - (*optional*) Path to encoderWorker.min.js worker script. Defaults to `encoderWorker.min.js`
- **encoderSampleRate** - (*optional*) Specifies the sample rate to encode at. Defaults to `48000`. Supported values are `8000`, `12000`, `16000`, `24000` or `48000`.
- **leaveStreamOpen** - (*optional*) Keep the stream around when trying to `stop` recording, so you can re-`start` without re-`initStream`. Defaults to `false`.
- **maxBuffersPerPage** - (*optional*) Specifies the maximum number of buffers to use before generating an Ogg page. This can be used to lower the streaming latency. The lower the value the more overhead the ogg stream will incur. Defaults to `40`.
- **monitorGain** - (*optional*) Sets the gain of the monitoring output. Gain is an a-weighted value between `0` and `1`. Defaults to `0`
- **numberOfChannels** - (*optional*) The number of channels to record. `1` = mono, `2` = stereo. Defaults to `1`. Maximum `2` channels are supported.
- **originalSampleRateOverride** - (*optional*) Override the ogg opus 'input sample rate' field. Google Speech API requires this field to be `16000`.
- **resampleQuality** - (*optional*) Value between 0 and 10 which determines latency and processing for resampling. `0` is fastest with lowest quality. `10` is slowest with highest quality. Defaults to `3`.
- **streamPages** - (*optional*) `dataAvailable` event will fire after each encoded page. Defaults to `false`.


推荐使用

* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


