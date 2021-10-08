---
title: PlaidML作为Keras的Backend
tags: 机器学习
categories: 机器学习
abbrlink: 84c9fbae
date: 2021-07-25 16:00:43
---

开发本选择MacOs的时候虽然很方便，得益于MacOS的便捷性，良好的生态，还有很好的交互体验，以及颜值也很高，但是还是会有些许尴尬，比如如果你是搞机器学习的同学。

首先新版本的数据科学家Mac本子，是带独立GPU的，听起来好像有GPU是一个很激动的事，但是，不要太激动。

Mac的GPU并不是常见的英伟达的显卡，而是苹果的AMD独立显卡，也不支持CUDA的安装包，所以很多时候常见这些DL框架都无法很好的使用Mac的显卡。

在使用TF这种底层框架的时候局限尤为明显，Keras的好处就是解耦了底层的框架，可以选择不同的backend，plaidml就是其中的一个backend。

一个简单的例子如下所示：


```
import numpy as np
from os import environ
environ["KERAS_BACKEND"] = "plaidml.keras.backend"
import keras
from keras.layers import Dense
from matplotlib import pyplot as plt

# Params
num_samples = 100000; vect_len = 20; max_int = 10; min_int = 1;

# Generate dataset
X = np.random.randint(min_int, max_int, (num_samples, vect_len))
Y = np.sum(X, axis=1)

# Get 80% of data for training
split_idx = int(0.8 * len(Y))
train_X = X[:split_idx, :]; test_X = X[split_idx:, :]
train_Y = Y[:split_idx]; test_Y = Y[split_idx:]

# Make model
model = keras.models.Sequential()
model.add(keras.layers.Dense(32, activation='relu', input_shape=(vect_len,)))
model.add(keras.layers.Dense(1))
model.compile('adam', 'mse')

history = model.fit(train_X, train_Y, validation_data=(test_X, test_Y), \
                    epochs=10, batch_size=100)

# summarize history
plt.plot(history.history['loss'])
plt.plot(history.history['val_loss'])
plt.title('model loss')
plt.ylabel('loss')
plt.xlabel('epoch')
plt.legend(['train', 'test'], loc='upper left')
plt.show()
```

参考链接：
[参考文章](https://link.medium.com/yCdHxDdQaib)
[GitHub地址](https://github.com/plaidml/plaidml)
