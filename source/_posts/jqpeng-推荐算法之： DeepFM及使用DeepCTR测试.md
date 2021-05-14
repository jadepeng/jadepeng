---
title: 推荐算法之： DeepFM及使用DeepCTR测试
tags: ["DeepFM","推荐","算法","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-10-16 14:21
---
文章作者:jqpeng
原文链接: [推荐算法之： DeepFM及使用DeepCTR测试](https://www.cnblogs.com/xiaoqi/p/deepfm.html)

## 算法介绍

左边deep network，右边FM，所以叫deepFM

![DeepFm](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/15/1602764397990.png)

包含两个部分：

- Part1： FM(Factorization machines)，因子分解机部分


![FM](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/15/1602764448337.png)

在传统的一阶线性回归之上，加了一个二次项，可以表达两两特征的相互关系。

![特征相互关系](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/15/1602764632765.png)

这里的公式可以简化，减少计算量，下图来至于网络。

![FM](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/15/1602765047458.png)

- Part2： Deep部分


deep部分是多层dnn网络。

## 算法实现

实现部分，[用Keras实现一个DeepFM](https://blog.csdn.net/songbinxu/article/details/80151814) 和[·清尘·《FM、FMM、DeepFM整理（pytorch）》](https://blog.csdn.net/u012969412/article/details/88684723)

讲的比较清楚，这里引用keras实现来说明。

整体的网络结构：

![网络结构](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/15/1602764969628.png)

### 特征编码

特征可以分为3类：

- 连续型field，比如数字类型特征
- 单值离散型特征，比如gender，可选为male、female
- 多值离散型，比如tag，可以有多个


连续型field，可以拼接到一起，dense数据。

单值，多值field进行Onehot后，可见单值离散field对应的独热向量只有一位取1，而多值离散field对应的独热向量有多于一位取1，表示该field可以同时取多个特征值。


| label | shop\_score | gender=m | gender=f | interest=f | interest=c |
| --- | --- | --- | --- | --- | --- |
| 0 | 0.2 | 1 | 0 | 1 | 1 |
| 1 | 0.8 | 0 | 1 | 0 | 1 |


### FM 部分

![FM](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/15/1602765456508.png)

看公式：  
![FM](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/15/1602764448337.png)

先算 FM一次项：

- 连续型field 可以用Dense(1)层实现
- 单值离散型field 用Embedding(n,1), n是分类中值的个数
- 多值离散型field可以同时取多个特征值，为了batch training，必须对样本进行补零padding。同样可以用Embedding实现，因为有多个Embedding，可以取下平均值。


![1次项](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/15/1602765746903.png)

然后计算FM二次项，这里理解比较费劲一点。

[·清尘·《FM、FMM、DeepFM整理（pytorch）》](https://blog.csdn.net/u012969412/article/details/88684723) 深入浅出的讲明白了这个过程，大家可以参见。

我们来看具体实现方面，这里的[DeepFM模型CTR预估理论与实战](http://fancyerii.github.io/2019/12/19/deepfm/#fm-layer) 讲解更容易理解。

![FM公式](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/16/1602819038585.png)

假设只有前面的C1和C2两个Category的特征，词典大小还是3和2。假设输入还是C1=2，C2=2(下标从1开始)，则Embedding之后为V2=[e21,e22,e23,e24]和V5=[e51,e52,e53,e54]。

因为xi和xj同时不为零才需要计算，所以上面的公式里需要计算的只有i=2和j=5的情况。因此：

![FM](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/16/1602819112460.png)

扩展到多个，比如C1,C2,C3，需要算内积

![](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/16/1602819135487.png)

怎么用用矩阵乘法一次计算出来呢？我们可以看看这个

![](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/16/1602819176970.png)

对应的代码就是


           square_of_sum = tf.square(reduce_sum(		concated_embeds_value, axis=1, keep_dims=True))	sum_of_square = reduce_sum(		concated_embeds_value * concated_embeds_value, axis=1, keep_dims=True)	cross_term = square_of_sum - sum_of_square	cross_term = 0.5 * reduce_sum(cross_term, axis=2, keep_dims=False)


其中concated\_embeds\_value是拼接起来的embeds\_value。

### Deep部分

DNN比较简单，FM的输入和DNN的输入都是同一个group\_embedding\_dict。

## 使用movielens 来测试

下载`ml-100k` 数据集


    wget http://files.grouplens.org/datasets/movielens/ml-100k.zip
    unzip ml-100k.zip


安装相关软件包，sklearn，deepctr

导入包：


    import pandas
    import pandas as pd
    import sklearn
    from sklearn.metrics import log_loss, roc_auc_score
    from sklearn.model_selection import train_test_split
    from sklearn.preprocessing import LabelEncoder
    from tensorflow.python.keras.preprocessing.sequence import pad_sequences
    import tensorflow as tf
    from tqdm import tqdm
    
    from deepctr.models import DeepFM
    from deepctr.feature_column import SparseFeat, VarLenSparseFeat, get_feature_names
    import numpy as np


读取评分数据：


    u_data = pd.read_csv("ml-100k/u.data", sep='\t', header=None)
    u_data.columns = ['user_id', 'movie_id', 'rating', 'timestamp']


有评分的设置为1，随机采用未评分的


    def neg_sample(u_data, neg_rate=1):
        # 全局随机采样
        item_ids = u_data['movie_id'].unique()
        print('start neg sample')
        neg_data = []
        # 负采样
        for user_id, hist in tqdm(u_data.groupby('user_id')):
            # 当前用户movie
            rated_movie_list = hist['movie_id'].tolist()
            candidate_set = list(set(item_ids) - set(rated_movie_list))
            neg_list_id = np.random.choice(candidate_set, size=len(rated_movie_list) * neg_rate, replace=True)
            for id in neg_list_id:
                neg_data.append([user_id, id, -1, 0])
        u_data_neg = pd.DataFrame(neg_data)
        u_data_neg.columns = ['user_id', 'movie_id', 'rating', 'timestamp']
        u_data = pandas.concat([u_data, u_data_neg])
        print('end neg sample')
        return u_data


读取item数据


    u_item = pd.read_csv("ml-100k/u.item", sep='|', header=None, error_bad_lines=False)
        genres_columns = ['Action', 'Adventure',
                          'Animation',
                          'Children', 'Comedy', 'Crime', 'Documentary', 'Drama', 'Fantasy',
                          'Film_Noir', 'Horror', 'Musical', 'Mystery', 'Romance', 'Sci-Fi',
                          'Thriller', 'War', 'Western']
    
        u_item.columns = ['movie_id', 'title', 'release_date', 'video_date', 'url', 'unknown'] + genres_columns
    


处理genres并删除单独的genres列


         genres_list = []
        for index, row in u_item.iterrows():
            genres = []
            for item in genres_columns:
                if row[item]:
                    genres.append(item)
            genres_list.append('|'.join(genres))
    
        u_item['genres'] = genres_list
        for item in genres_columns:
            del u_item[item]


读取用户信息：


      # user id | age | gender | occupation(职业) | zip code（邮编，地区)
        u_user = pd.read_csv("ml-100k/u.user", sep='|', header=None)
        u_user.columns = ['user_id', 'age', 'gender', 'occupation', 'zip']


join到一起：


     data = pandas.merge(u_data, u_item, on="movie_id", how='left')
     data = pandas.merge(data, u_user, on="user_id", how='left')
     data.to_csv('ml-100k/data.csv', index=False)


处理特征：


    sparse_features = ["movie_id", "user_id",
                       "gender", "age", "occupation", "zip", ]
    
    data[sparse_features] = data[sparse_features].astype(str)
    target = ['rating']
    
    # 评分
    data['rating'] = [1 if int(x) >= 0 else 0 for x in data['rating']]


先特征编码：


    for feat in sparse_features:	lbe = LabelEncoder()	data[feat] = lbe.fit_transform(data[feat])


处理genres特征，一个movie有多个genres，先拆分，然后编码为数字，注意是从1开始；由于每个movie的genres长度不一样，可以计算最大长度，位数不足的后面补零（pad\_sequences，在post补0）


    	 def split(x):		key_ans = x.split('|')		for key in key_ans:			if key not in key2index:				# Notice : input value 0 is a special "padding",so we do not use 0 to encode valid feature for sequence input				key2index[key] = len(key2index) + 1		return list(map(lambda x: key2index[x], key_ans))
    
    	key2index = {}	genres_list = list(map(split, data['genres'].values))	genres_length = np.array(list(map(len, genres_list)))	max_len = max(genres_length)	# Notice : padding=`post`	genres_list = pad_sequences(genres_list, maxlen=max_len, padding='post', )


构建deepctr的特征列，主要分为两类特征，一是定长的SparseFeat，稀疏的类别特征，二是可变长度的VarLenSparseFeat，像genres这样的包含多个的。


    	   fixlen_feature_columns = [SparseFeat(feat, data[feat].nunique(), embedding_dim=4)							  for feat in sparse_features]
    	use_weighted_sequence = False	if use_weighted_sequence:		varlen_feature_columns = [VarLenSparseFeat(SparseFeat('genres', vocabulary_size=len(			key2index) + 1, embedding_dim=4), maxlen=max_len, combiner='mean',												   weight_name='genres_weight')]  # Notice : value 0 is for padding for sequence input feature	else:		varlen_feature_columns = [VarLenSparseFeat(SparseFeat('genres', vocabulary_size=len(			key2index) + 1, embedding_dim=4), maxlen=max_len, combiner='mean',												   weight_name=None)]  # Notice : value 0 is for padding for sequence input feature
    	linear_feature_columns = fixlen_feature_columns + varlen_feature_columns	dnn_feature_columns = fixlen_feature_columns + varlen_feature_columns
    	feature_names = get_feature_names(linear_feature_columns + dnn_feature_columns)


封装训练数据，先shuffle（乱排）数据，然后生成dict input数据。


    data = sklearn.utils.shuffle(data)
    train_model_input = {name: data[name] for name in sparse_features}  #
    train_model_input["genres"] = genres_list


构建DeepFM模型，由于目标值是0,1，因此采用binary，损失函数用binary\_crossentropy


    model = DeepFM(linear_feature_columns, dnn_feature_columns, task='binary')
    
    model.compile(optimizer=tf.keras.optimizers.Adam(), loss='binary_crossentropy',
                  metrics=['AUC', 'Precision', 'Recall'])
    model.summary()


训练模型：


    model.fit(train_model_input, data[target].values,					batch_size=256, epochs=20, verbose=2,					validation_split=0.2			)


开始训练：


    Epoch 1/20
    625/625 - 3s - loss: 0.5081 - auc: 0.8279 - precision: 0.7419 - recall: 0.7695 - val_loss: 0.4745 - val_auc: 0.8513 - val_precision: 0.7563 - val_recall: 0.7936
    Epoch 2/20
    625/625 - 2s - loss: 0.4695 - auc: 0.8538 - precision: 0.7494 - recall: 0.8105 - val_loss: 0.4708 - val_auc: 0.8539 - val_precision: 0.7498 - val_recall: 0.8127
    Epoch 3/20
    625/625 - 2s - loss: 0.4652 - auc: 0.8564 - precision: 0.7513 - recall: 0.8139 - val_loss: 0.4704 - val_auc: 0.8545 - val_precision: 0.7561 - val_recall: 0.8017
    Epoch 4/20
    625/625 - 2s - loss: 0.4624 - auc: 0.8579 - precision: 0.7516 - recall: 0.8146 - val_loss: 0.4724 - val_auc: 0.8542 - val_precision: 0.7296 - val_recall: 0.8526
    Epoch 5/20
    625/625 - 2s - loss: 0.4607 - auc: 0.8590 - precision: 0.7521 - recall: 0.8173 - val_loss: 0.4699 - val_auc: 0.8550 - val_precision: 0.7511 - val_recall: 0.8141
    Epoch 6/20
    625/625 - 2s - loss: 0.4588 - auc: 0.8602 - precision: 0.7545 - recall: 0.8165 - val_loss: 0.4717 - val_auc: 0.8542 - val_precision: 0.7421 - val_recall: 0.8265
    Epoch 7/20
    625/625 - 2s - loss: 0.4574 - auc: 0.8610 - precision: 0.7535 - recall: 0.8192 - val_loss: 0.4722 - val_auc: 0.8547 - val_precision: 0.7549 - val_recall: 0.8023
    Epoch 8/20
    625/625 - 2s - loss: 0.4561 - auc: 0.8619 - precision: 0.7543 - recall: 0.8201 - val_loss: 0.4717 - val_auc: 0.8548 - val_precision: 0.7480 - val_recall: 0.8185
    Epoch 9/20
    625/625 - 2s - loss: 0.4531 - auc: 0.8643 - precision: 0.7573 - recall: 0.8210 - val_loss: 0.4696 - val_auc: 0.8583 - val_precision: 0.7598 - val_recall: 0.8103
    Epoch 10/20
    625/625 - 2s - loss: 0.4355 - auc: 0.8768 - precision: 0.7787 - recall: 0.8166 - val_loss: 0.4435 - val_auc: 0.8769 - val_precision: 0.7756 - val_recall: 0.8293
    Epoch 11/20
    625/625 - 2s - loss: 0.4093 - auc: 0.8923 - precision: 0.7915 - recall: 0.8373 - val_loss: 0.4301 - val_auc: 0.8840 - val_precision: 0.7806 - val_recall: 0.8390
    Epoch 12/20
    625/625 - 2s - loss: 0.3970 - auc: 0.8988 - precision: 0.7953 - recall: 0.8497 - val_loss: 0.4286 - val_auc: 0.8867 - val_precision: 0.7903 - val_recall: 0.8299
    Epoch 13/20
    625/625 - 2s - loss: 0.3896 - auc: 0.9029 - precision: 0.8001 - recall: 0.8542 - val_loss: 0.4253 - val_auc: 0.8888 - val_precision: 0.7913 - val_recall: 0.8322
    Epoch 14/20
    625/625 - 2s - loss: 0.3825 - auc: 0.9067 - precision: 0.8038 - recall: 0.8584 - val_loss: 0.4205 - val_auc: 0.8917 - val_precision: 0.7885 - val_recall: 0.8506
    Epoch 15/20
    625/625 - 2s - loss: 0.3755 - auc: 0.9102 - precision: 0.8074 - recall: 0.8624 - val_loss: 0.4204 - val_auc: 0.8940 - val_precision: 0.7868 - val_recall: 0.8607
    Epoch 16/20
    625/625 - 2s - loss: 0.3687 - auc: 0.9136 - precision: 0.8117 - recall: 0.8653 - val_loss: 0.4176 - val_auc: 0.8956 - val_precision: 0.8097 - val_recall: 0.8236
    Epoch 17/20
    625/625 - 2s - loss: 0.3617 - auc: 0.9170 - precision: 0.8155 - recall: 0.8682 - val_loss: 0.4166 - val_auc: 0.8966 - val_precision: 0.8056 - val_recall: 0.8354
    Epoch 18/20
    625/625 - 2s - loss: 0.3553 - auc: 0.9201 - precision: 0.8188 - recall: 0.8716 - val_loss: 0.4168 - val_auc: 0.8977 - val_precision: 0.7996 - val_recall: 0.8492
    Epoch 19/20
    625/625 - 2s - loss: 0.3497 - auc: 0.9227 - precision: 0.8214 - recall: 0.8741 - val_loss: 0.4187 - val_auc: 0.8973 - val_precision: 0.8079 - val_recall: 0.8358
    Epoch 20/20
    625/625 - 2s - loss: 0.3451 - auc: 0.9248 - precision: 0.8244 - recall: 0.8753 - val_loss: 0.4210 - val_auc: 0.8982 - val_precision: 0.7945 - val_recall: 0.8617


最后我们测试下数据：


     pred_ans = model.predict(train_model_input, batch_size=256)
     count = 0
        for (i, j) in zip(pred_ans, data['rating'].values):
            print(i, j)
            count += 1
            if count > 10:
                break


输出如下：


    [0.20468083] 0
    [0.1988303] 0
    [7.7236204e-05] 0
    [0.9439401] 1
    [0.76648283] 0
    [0.80082995] 1
    [0.7689271] 0
    [0.8515004] 1
    [0.93311656] 1
    [0.40019292] 0
    [0.60735244] 0


## 参考

- [deepFM in pytorch](https://blog.csdn.net/w55100/article/details/90295932)
- [皮果提《Factorization Machines 学习笔记（二）模型方程》](https://blog.csdn.net/itplus/article/details/40534923)
- [·清尘·《FM、FMM、DeepFM整理（pytorch）》](https://blog.csdn.net/u012969412/article/details/88684723)
- [用Keras实现一个DeepFM](https://blog.csdn.net/songbinxu/article/details/80151814)


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


