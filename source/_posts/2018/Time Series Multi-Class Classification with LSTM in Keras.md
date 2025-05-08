---
title: Time Series Classification with LSTM in Keras
date: 2018-03-07 10:15:41
tags: 机器学习
---

本文总结一下最近在项目中学到的一些东西。

其实在抽象问题上花了很多时间，抽象出来问题其实也很简单：

已知一个时序序列，序列中的每个元素本身就是一个label。
1. 预测序列下一个元素；
2. 取概率最高的元素，如果预测的label与真实的label一致，认为序列正常；否则序列出现异常。

这样问题的描述也就很清楚了。1是一个序列预测多分类问题，2是一个序列异常判断二分类问题。

# 1. 序列预测多分类问题

## 模型选择
因为LSTM在处理序列问题上有很好效果，所以使用两层LSTM，最后加一个全连接层作为输出层，激活函数选择softmax。

```
model = Sequential()
model.add(LSTM(64, input_shape=(timesteps, input_dim), return_sequences=True))
model.add(LSTM(64))
model.add(Dense(output_dim, activation='softmax'))
```

## 训练阶段
需要对原始数据进行编码处理，首先使用LabelEncoder将label转换为integer，然后使用np_utils.to_categorical()转换为one-hot编码。

```
# 进行one-hot编码
encoder = LabelEncoder()
encoder.fit(train_Y)
encoded_Y = encoder.transform(train_Y)
train_Y = np_utils.to_categorical(encoded_Y)
```

```
# 使用训练数据训练模型
model.compile(loss='categorical_crossentropy', optimizer='rmsprop', metrics=['accuracy'])
model.fit(X, Y, batch_size=5, epochs=100, verbose=1)
```

## 评估阶段
多分类问题使用macro-P/macro-R/macro-F1评估。

本来打算使用KFold进行交叉验证的，但是报错：`ValueError: Classification metrics can't handle a mix of multilabel-indicator and multiclass targets`。

确实，对于sample进行了one-hot编码后Classifier认为是multilabel问题，但真正的label只有一个。但是封装的API里面并不能one-hot编码decode回来，所以只好手动划分训练集和验证集。

```
# 预测label
predict_dummy_y = model.predict(test_x, verbose=0)
predict_y = encoder.inverse_transform(numpy.argmax(predict_dummy_y))
```

```
# 模型效果评估
precision = precision_score(test_Y, predict_Y, average='macro')
recall = recall_score(test_Y, predict_Y, average='macro')
f1 = f1_score(test_Y, predict_Y, average='macro')
```

# 2. 序列异常判断二分类问题（未完善）

因为缺乏标注数据，验证阶段没法进行。当然，如果认为原来的序列是完全正常的，这样可以用recall=TP/(TP+FN)计算recall。

在1中可以得到预测label的概率分布，直接进行评估即可。

## 评估阶段
如前所述，只能计算recall。

```
# 在预测时记录TP的情况
if predict_y == test_y:
    predict_Y.append(0)
```

```
# 计算recall
recall_abnormal = len(predict_Y)/len(test_Y)
```

---
## 参考文献
1. [Time Series Forecasting with the Long Short-Term Memory Network in Python](https://machinelearningmastery.com/time-series-forecasting-long-short-term-memory-network-python/)
2. [Multi-Class Classification Tutorial with the Keras Deep Learning Library](https://machinelearningmastery.com/multi-class-classification-tutorial-keras-deep-learning-library/)
3. [使用Scikit-Learn调用Keras的模型](https://cnbeining.github.io/deep-learning-with-python-cn/3-multi-layer-perceptrons/ch9-use-keras-models-with-scikit-learn-for-general-machine-learning.html)
