---
title: fastText训练word2vec并用于训练任务
tags: ["fastText","jqpeng"]
categories: ["博客","jqpeng"]
date: 2019-11-22 15:45
---
文章作者:jqpeng
原文链接: [fastText训练word2vec并用于训练任务](https://www.cnblogs.com/xiaoqi/p/fastText.html)

最近测试OpenNRE，没有GPU服务器，bert的跑不动，于是考虑用word2vec，捡起fasttext

## 下载安装

先clone代码


    git clone https://github.com/facebookresearch/fastText.git


然后make编译：


    make


编译后，将生成的fastText移到bin


    cp fasttext /usr/local/bin/


## 训练word2vec

先讲语料分好词，比如保存到sent\_train.txt，文件内容是中文分词后的内容:


    楚穆王 十二年 : （ 丁未 ， 公元前 614 年 ） ， 在位 12 年 的 楚穆王 死 ， 死后 葬 在 楚 郢 之西 。


开始调用fasttext训练：


    fasttext skipgram -input sent_train.txt -output ./result


很快就跑完了，跑完后，可以看到生成两个文件：


    # ll result.*
    -rw-r--r-- 1 root root 876945058 Nov 21 09:29 result.bin
    -rw-r--r-- 1 root root  82656362 Nov 21 09:29 result.vec
    


来看下vec文件，可以看到是100维的向量


    # head result.vec 
    94161 100
    ， 0.030437 0.12177 -0.1367 0.11101 -0.49543 0.49908 0.033441 -0.025445 -0.036312 -0.081132 -0.082666 0.27204 -0.2367 -0.23424 -0.30124 -0.029666 -0.23803 -0.083255 -0.03177 -0.23129 0.33953 -0.095728 0.023824 -0.33981 -0.048715 -0.22876 0.24215 -0.094075 -0.077224 0.097473 -0.012714 -0.16661 -0.5156 -0.12635 -0.34265 -0.13444 -0.25535 -0.29832 0.14624 -0.24779 0.25403 0.17662 0.070345 -0.15927 0.3449 0.11372 0.22504 0.15652 0.19013 -0.029641 -0.1761 0.018512 -0.19782 -0.15607 0.39958 0.31343 0.30654 0.062457 -0.045659 -0.008893 0.11445 0.035771 0.048592 0.17336 0.15742 -0.085562 -0.12398 0.25767 -0.087141 -0.10011 -0.14832 0.11072 0.0037114 0.18156 -0.32666 -0.081212 0.1102 -0.035646 0.09467 0.014385 0.11191 -0.14713 -0.0052515 -0.006049 0.3735 -0.13804 0.12271 -0.050977 -0.019325 -0.034865 0.019665 -0.16755 0.034194 0.074825 0.16173 -0.20006 -0.03904 -0.18061 0.040119 -0.22622 
    </s> -0.23353 0.071758 -0.26913 -0.14217 -0.1736 0.22807 -0.11152 -0.047725 0.19557 0.13388 -0.21704 0.39025 -0.30286 0.16748 -0.18748 0.11423 -0.19393 -0.10635 -0.12826 -0.3244 0.27615 -0.25832 -0.17595 -0.12634 -0.094196 -0.19782 0.14435 -0.059313 -0.24001 -0.13996 -0.09501 -0.26155 -0.35677 0.059324 -0.23963 -0.20722 -0.37483 -0.11253 0.021369 -0.15571 0.059181 0.33843 -0.058266 -0.12393 0.17777 -0.032558 0.17864 0.28223 0.058037 0.13108 -0.31817 0.081199 -0.05605 -0.029366 0.30827 0.3208 0.070286 0.062643 0.0040956 -0.080481 0.0064075 -0.087952 0.19877 0.33604 0.28209 0.073563 -0.097628 0.035748 -0.20385 -0.28676 -0.12122 0.10025 -0.05521 0.22991 -0.326 0.062162 0.090364 -0.20831 0.3678 -0.00043566 0.059466 -0.068502 -0.072635 0.08424 0.56188 -0.2588 0.15091 -0.15923 -0.12595 0.086243 0.08293 -0.37854 0.055448 0.11274 0.19559 -0.17132 -0.0858 -0.072667 -0.10356 0.1394 
    。 -0.097674 0.13847 -0.25706 0.057651 -0.29097 0.23529 -0.022163 -0.046278 0.18153 0.15302 -0.25117 0.30537 -0.22519 0.072574 -0.20933 -0.012315 -0.098127 -0.13748 -0.10589 -0.24647 0.38788 -0.077517 -0.24665 -0.1512 -0.17301 -0.13699 0.15931 -0.050389 -0.20344 -0.10393 -0.086151 -0.12502 -0.51355 -0.078194 -0.32112 -0.25169 -0.55924 -0.083918 0.13193 -0.12648 0.030666 0.37635 -0.0068401 -0.082757 0.35918 0.099755 0.048127 0.14651 -0.078756 0.16794 -0.35228 0.096068 -0.083268 -0.16416 0.30675 0.2112 -0.0034745 0.06171 0.015094 0.035436 0.13429 -0.036958 0.052708 0.39273 0.15883 0.015595 -0.014254 0.025274 -0.061765 -0.20447 -0.11626 0.12291 -0.13875 0.28874 -0.46607 -0.0064296 0.09502 -0.19274 0.32262 -0.077533 0.058291 -0.11019 -0.23094 -0.023817 0.59467 -0.31411 0.13071 -0.064146 -0.10452 -0.014019 0.28547 -0.3112 -0.019938 0.073268 0.2858 -0.087934 -0.0038124 -0.032765 -0.086257 0.07277 
    的 -0.32055 0.26205 -0.12693 0.036473 -0.48332 0.37801 0.043741 0.063979 0.17719 -0.0034521 -0.28247 0.4286 -0.11431 0.02168 -0.2469 -0.34261 -0.29886 -0.11997 -0.068971 -0.084678 0.36461 -0.089133 -0.056445 -0.15533 0.00017123 0.1496 0.24858 0.12394 -0.23362 -0.26373 0.037876 -0.20656 -0.48941 0.093864 -0.21763 -0.35464 -0.31409 -0.10626 0.064067 -0.21431 0.028025 0.27952 -0.1368 -0.30315 0.39894 0.23021 0.19465 0.12281 0.22508 -0.14596 -0.31362 -0.035584 0.076387 -0.28307 0.32819 0.21772 0.2417 0.23587 -0.097756 -0.18368 -0.027078 -0.15416 0.095119 -0.16597 0.096744 0.20759 0.083306 0.24435 -0.055484 -0.17169 -0.031104 0.13582 0.15192 0.066508 -0.19847 -0.28637 0.027218 -0.030856 0.36561 -0.13589 0.26368 -0.13762 -0.21137 -0.24706 0.46078 -0.31472 0.080658 0.23818 -0.060492 0.18232 0.19158 -0.16032 0.14793 0.021469 0.22363 -0.20411 0.07628 -0.096523 -0.11407 -0.35992 
    


## 转换为pytorch可加载格式

为了方便训练使用，需要转换下：


    import pickle as pkl
    import numpy as np
    import os
    import json
    
    def create_wordVec(vec_file，word2id,vec):
        word_map = {}
        word_map['PAD'] = len(word_map)
        word_map['UNK'] = len(word_map)
        word_embed = []
        for line in open(vec_file):
            content = line.strip().split()
            if len(content) != 100 + 1:
                continue
            word_map[content[0]] = len(word_map)
            word_embed.append(np.asarray(content[1:], dtype=np.float32))
        word_embed = np.array(word_embed)
        np.save(vec,word_embed)
        with open(word2id, 'w+', encoding='utf-8') as fw:
          fw.write(json.dumps(word_map, ensure_ascii=False))
    
    create_wordVec('result.vec','word2id.json','word2vec.npy')
    


## 训练模型

参考opennre的cnn分类代码：


    import torch
    import numpy as np
    import json
    import opennre
    from opennre import encoder, model, framework
    
    print('load word2vec ...')
    ckpt = 'ckpt/semeval_cnn_softmax.pth.tar'
    wordi2d = json.load(open('pretrain/glove/word2id.json'))
    word2vec = np.load('pretrain/glove/word2vec.npy')
    rel2id = json.load(open('benchmark/ccks/ccks_rel2id.txt'))
    print('create model ...')
    sentence_encoder = opennre.encoder.CNNEncoder(token2id=wordi2d,
                                                 max_length=100,
                                                 word_size=100,
                                                 position_size=5,
                                                 hidden_size=230,
                                                 blank_padding=True,
                                                 kernel_size=3,
                                                 padding_size=1,
                                                 word2vec=word2vec,
                                                 dropout=0.5)
    model = opennre.model.SoftmaxNN(sentence_encoder, len(rel2id), rel2id)
    framework = opennre.framework.SentenceRE(
        train_path='benchmark/ccks/ccks_train.txt',
        val_path='benchmark/ccks/ccks_dev.txt',
        test_path='benchmark/ccks/ccks_dev.txt',
        model=model,
        ckpt=ckpt,
        batch_size=32,
        max_epoch=100,
        lr=0.1,
        weight_decay=1e-5,
        opt='sgd')
    # Train
    #framework.train_model(metric='micro_f1')
    # Test
    print('load model ...')
    
    framework.load_state_dict(torch.load(ckpt)['state_dict'])
    framework.train_model(metric='micro_f1')
    result = framework.eval_model(framework.test_loader)
    print('Accuracy on test set: {}'.format(result['acc']))
    print('Micro Precision: {}'.format(result['micro_p']))
    print('Micro Recall: {}'.format(result['micro_r']))
    print('Micro F1: {}'.format(result['micro_f1']))
    


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


