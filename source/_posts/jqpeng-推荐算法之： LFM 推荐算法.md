---
title: 推荐算法之： LFM 推荐算法
tags: ["推荐","jqpeng"]
categories: ["博客","jqpeng"]
date: 2020-10-12 21:10
---
文章作者:jqpeng
原文链接: [推荐算法之： LFM 推荐算法](https://www.cnblogs.com/xiaoqi/p/13805530.html)

## LFM介绍

LFM(Funk SVD) 是利用 矩阵分解的推荐算法：


    R  = P * Q


其中：

- P矩阵是User-LF矩阵，即用户和隐含特征矩阵
- Q矩阵是LF-Item矩阵，即隐含特征和物品的矩阵
- R：R矩阵是User-Item矩阵，由P\*Q得来


见下图：

![LFM](https://gitee.com/jadepeng/pic/raw/master/pic/2020/10/12/1602505053626.png)

R评分举证由于物品和用户数量巨大，且稀疏，因此利用矩阵乘法，转换为 P（n\_user \* dim) 和 Q （dim\*n\_count) 两个矩阵，dim 是隐含特征数量。

## Tensorflow实现

下载`ml-100k` 数据集


    !wget http://files.grouplens.org/datasets/movielens/ml-100k.zip
    !unzip ml-100k.zip



    Resolving files.grouplens.org (files.grouplens.org)... 128.101.65.152
    Connecting to files.grouplens.org (files.grouplens.org)|128.101.65.152|:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 4924029 (4.7M) [application/zip]
    Saving to: ‘ml-100k.zip’
    
    ml-100k.zip         100%[===================>]   4.70M  16.2MB/s    in 0.3s    
    
    2020-10-12 12:25:07 (16.2 MB/s) - ‘ml-100k.zip’ saved [4924029/4924029]
    
    /bin/bash: uzip: command not found


评分数据在u.data里，分别是 user\_id, movie\_id, rating, timestamp


    !head ml-100k/u.data



    186	302	3	891717742
    22	377	1	878887116
    244	51	2	880606923
    166	346	1	886397596
    298	474	4	884182806
    115	265	2	881171488
    253	465	5	891628467
    305	451	3	886324817
    6	86	3	883603013


读取数据


    import os
    def read_data(path: str, separator: str):
        data = []
        with open(path, 'r') as f:
            for line in f.readlines():
                values = line.strip().split(separator)
                user_id, movie_id, rating, timestamp = int(values[0]), int(values[1]), int(values[2]), int(values[3])
                data.append((user_id, movie_id, rating, timestamp))
        return data
    
    data = read_data('ml-100k/u.data', '\t')
    
    print(data[0])



    (0, 0, 0.6)
    


拆分训练集和测试集，test\_ratio比例为0.3：


    
    data = [(d[0], d[1], d[2]/5.0) for d in data]
    
    # 拆分
    test_ratio = 0.3
    n_test = int(len(data) * test_ratio)
    test_data, train_data = data[:n_test], data[n_test:]
    


id规整化，从0开始增长


    #id 规整
    def normalize_id(data):
        new_data = []
        n_user, n_item = 0, 0
        user_id_old2new, item_id_old2new = {}, {}
        for user_id_old, item_id_old, label in data:
            if user_id_old not in user_id_old2new:
                user_id_old2new[user_id_old] = n_user
                n_user += 1
            if item_id_old not in item_id_old2new:
                item_id_old2new[item_id_old] = n_item
                n_item += 1
            new_data.append((user_id_old2new[user_id_old], item_id_old2new[item_id_old], label))
        return new_data, n_user, n_item, user_id_old2new, item_id_old2new
    
    data, n_user, n_item, user_id_old2new, item_id_old2new = normalize_id(data)


查看数据


    print(train_data[0:10])
    print(test_data[0])
    print('n_user',n_user)
    print('n_item',n_item)



    (196, 242, 0.6)
    n_user 943
    n_item 1682


准备数据集


    import tensorflow as tf
    
    def xy(data):
            user_ids = tf.constant([d[0] for d in data], dtype=tf.int32)
            item_ids = tf.constant([d[1] for d in data], dtype=tf.int32)
            labels = tf.constant([d[2] for d in data], dtype=tf.keras.backend.floatx())
            return {'user_id': user_ids, 'item_id': item_ids}, labels
    
    batch = 128
    train_ds = tf.data.Dataset.from_tensor_slices(xy(train_data)).shuffle(len(train_data)).batch(batch)
    test_ds = tf.data.Dataset.from_tensor_slices(xy(test_data)).batch(batch)


TF模型


    def LFM_model(n_user: int, n_item: int, dim=100, l2=1e-6) -> tf.keras.Model:
        l2 = tf.keras.regularizers.l2(l2)
        user_id = tf.keras.Input(shape=(), name='user_id', dtype=tf.int32)
    
        user_embedding = tf.keras.layers.Embedding(n_user, dim, embeddings_regularizer=l2)(user_id)
        # (None,dim)
        item_id = tf.keras.Input(shape=(), name='item_id', dtype=tf.int32)
        item_embedding = tf.keras.layers.Embedding(n_item, dim, embeddings_regularizer=l2)(item_id)
        # (None,dim)
        r = user_embedding * item_embedding
        y = tf.reduce_sum(r, axis=1)
        y = tf.where(y < 0., 0., y)
        y = tf.where(y > 1., 1., y)
        y = tf.expand_dims(y, axis=1)
        return tf.keras.Model(inputs=[user_id, item_id], outputs=y)
    
    model = LFM_model(n_user + 1, n_item + 1, 64)


编译模型


    model.compile(optimizer="adam", loss=tf.losses.MeanSquaredError(), metrics=['AUC', 'Precision', 'Recall'])
    model.summary()
    



    __________________________________________________________________________________________________
    Layer (type)                    Output Shape         Param #     Connected to                     
    ==================================================================================================
    user_id (InputLayer)            [(None,)]            0                                            
    __________________________________________________________________________________________________
    item_id (InputLayer)            [(None,)]            0                                            
    __________________________________________________________________________________________________
    embedding_12 (Embedding)        (None, 64)           60416       user_id[0][0]                    
    __________________________________________________________________________________________________
    embedding_13 (Embedding)        (None, 64)           107712      item_id[0][0]                    
    __________________________________________________________________________________________________
    tf_op_layer_Mul_6 (TensorFlowOp [(None, 64)]         0           embedding_12[0][0]               
                                                                     embedding_13[0][0]               
    __________________________________________________________________________________________________
    tf_op_layer_Sum_6 (TensorFlowOp [(None,)]            0           tf_op_layer_Mul_6[0][0]          
    __________________________________________________________________________________________________
    tf_op_layer_Less_6 (TensorFlowO [(None,)]            0           tf_op_layer_Sum_6[0][0]          
    __________________________________________________________________________________________________
    tf_op_layer_SelectV2_12 (Tensor [(None,)]            0           tf_op_layer_Less_6[0][0]         
                                                                     tf_op_layer_Sum_6[0][0]          
    __________________________________________________________________________________________________
    tf_op_layer_Greater_6 (TensorFl [(None,)]            0           tf_op_layer_SelectV2_12[0][0]    
    __________________________________________________________________________________________________
    tf_op_layer_SelectV2_13 (Tensor [(None,)]            0           tf_op_layer_Greater_6[0][0]      
                                                                     tf_op_layer_SelectV2_12[0][0]    
    __________________________________________________________________________________________________
    tf_op_layer_ExpandDims_6 (Tenso [(None, 1)]          0           tf_op_layer_SelectV2_13[0][0]    
    ==================================================================================================
    Total params: 168,128
    Trainable params: 168,128
    Non-trainable params: 0
    __________________________________________________________________________________________________


下面开始训练10次epochs


    model.fit(train_ds,validation_data=test_ds,epochs=10)



    Epoch 1/10
    
    /usr/local/lib/python3.6/dist-packages/tensorflow/python/framework/indexed_slices.py:432: UserWarning: Converting sparse IndexedSlices to a dense Tensor of unknown shape. This may consume a large amount of memory.
      "Converting sparse IndexedSlices to a dense Tensor of unknown shape. "
    547/547 [==============================] - 2s 4ms/step - loss: 0.4388 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.0594 - val_loss: 0.1439 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.4180
    Epoch 2/10
    547/547 [==============================] - 2s 4ms/step - loss: 0.0585 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.8171 - val_loss: 0.0486 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8655
    Epoch 3/10
    547/547 [==============================] - 2s 3ms/step - loss: 0.0393 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.9053 - val_loss: 0.0433 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8982
    Epoch 4/10
    547/547 [==============================] - 2s 3ms/step - loss: 0.0346 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.9107 - val_loss: 0.0415 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8947
    Epoch 5/10
    547/547 [==============================] - 2s 4ms/step - loss: 0.0301 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.9071 - val_loss: 0.0410 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8869
    Epoch 6/10
    547/547 [==============================] - 2s 4ms/step - loss: 0.0257 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.8958 - val_loss: 0.0410 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8849
    Epoch 7/10
    547/547 [==============================] - 2s 4ms/step - loss: 0.0218 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.8844 - val_loss: 0.0414 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8753
    Epoch 8/10
    547/547 [==============================] - 2s 4ms/step - loss: 0.0183 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.8719 - val_loss: 0.0425 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8659
    Epoch 9/10
    547/547 [==============================] - 2s 4ms/step - loss: 0.0153 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.8624 - val_loss: 0.0435 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8620
    Epoch 10/10
    547/547 [==============================] - 2s 4ms/step - loss: 0.0132 - auc: 0.0000e+00 - precision: 1.0000 - recall: 0.8535 - val_loss: 0.0449 - val_auc: 0.0000e+00 - val_precision: 1.0000 - val_recall: 0.8531


* * *


> 作者：Jadepeng  
>  出处：jqpeng的技术记事本--[http://www.cnblogs.com/xiaoqi](http://www.cnblogs.com/xiaoqi)  
>  您的支持是对博主最大的鼓励，感谢您的认真阅读。  
>  本文版权归作者所有，欢迎转载，但未经作者同意必须保留此段声明，且在文章页面明显位置给出原文连接，否则保留追究法律责任的权利。


