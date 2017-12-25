title: Tensorflow学习笔记2-深入MNIST
---
> TF的后端是C++，Python与后端的连接称为session。一般，首先创建一个图，然后再session中启动它。

上次的模型准确率是91%左右。现在，采用**多层卷积网络来提高准确率**。

## 权重初始化

为了提高模型的准确性，需要创建大量的权重(W)和偏置量(b)。这个模型中的权重在初始化时需要加入少量的**噪声**来打破*对称性*和避免*0梯度*。本次实验使用的是**ReLU**神经元，因此比较好的做法是用一个较小的正数来初始化偏置项，以避免神经元节点输出恒为0的问题(dead neurons)。为了不在建立模型的时候反复进行初始化，定义两个函数用于初始化。


```
# 初始化操作
def weight_variable(shape):
    initial = tf.truncated_normal(shape, stddev=0.1)
    return tf.Variable(initial)

def bias_variable(shape):
    initial = tf.constant(0.1, shape=shape)
    return tf.Variable(initial)
```

## 卷积和池化

本次实验使用**vanilla**版本，卷积使用1步长(stride size)，0边距(padding size)的模板，保证输出和输入是同一个大小。池化使用简单传统的2x2大学的模板max pooling。


```
def conv2d(x, W):
    return tf.nn.conv2d(x, W, strides=[1, 1, 1, 1], padding='SAME')

def max_pool_2x2(x):
    return tf.nn.max_pool(x, ksize=[1, 2, 2, 1],
                          strides=[1, 2, 2, 1], padding='SAME')
```

## 第一层卷积

组成：1卷积+1max pooling。

卷积在每个5x5的patch中算出32的特征。卷积的权重张量形状是[5, 5, 1, 32],前两个维度是patch的大小，接着是输入的通道数目，最后是输出的通道数目。对于每一个输出通道都有对应的偏置量。


```
w_conv1 = weight_variable([5, 5, 1, 32])
b_conv1 = bias_variable([32])
```

为了用这一次，我们把x变成一个4D向量，其2，3维度对应图片的宽，高，最后一维代表图片的颜色通道数（灰度图，通道数为1，rgv为3）。


```
x_image = tf.reshape(x, [-1, 28, 28, 1])
```

把x_image和权值向量进行卷积，加上偏置项，然后使用ReLU函数激活，最后进行max pooling。


```
h_conv1 = tf.nn.relu(conv2d(x_image, W_conv1) + b_conv1)
h_pool1 = max_pool_2x2(h_conv1)
```

## 第二层卷积

把类似的层堆叠起来，第二层中，每个5x5的patch会得到64个特征。


```
W_conv2 = weight_variable([5, 5, 32, 64])
b_conv2 = bias_variable([64])

h_conv2 = tf.nn.relu(conv2d(h_pool1, W_conv2) + b_conv2)
h_pool2 = max_pool_2x2(h_conv2)
```

## 密集连接层

现在，图片尺寸缩小到了7x7，我们加入一个有1024个神经元的全连接层，用于处理整个图片。把池化层输出的张量reshape成一些向量，乘以权重矩阵，加上偏置，然后对其使用ReLU.


```
W_fc1 = weight_variable([7 * 7 * 64, 1024])
b_fc1 = bias_variable([1024])

h_pool2_flat = tf.reshape(h_pool2, [-1, 7*7*64])
h_fc1 = tf.nn.relu(tf.matmul(h_pool2_flat, W_fc1) + b_fc1)
```

## Dropout

为了减少过拟合，需要在输出层加入dropout。用一个placeholder来代表一个神经元的输出在dropout中保持不变的概率。


```
keep_prob = tf.placeholder("float")
h_fc1_drop = tf.nn.dropout(h_fc1, keep_prob)
```

## 输出层

最后，添加一个SoftMax层来进行softmax回归。

```
W_fc2 = weight_variable([1024, 10])
b_fc2 = bias_variable([10])

y_conv = tf.nn.softmax(tf.matmul(h_fc1_drop, W_fc2) + b_fc2)
```

## 训练和评估模型

使用更复杂的ADAM优化器来做梯度最速下降。


```
cross_entropy = -tf.reduce_sum(y_*tf.log(y_conv))
train_step = tf.train.AdamOptimizer(1e-4).minimize(cross_entropy)
correct_prediction = tf.equal(tf.argmax(y_conv,1), tf.argmax(y_,1))
accuracy = tf.reduce_mean(tf.cast(correct_prediction, "float"))
sess.run(tf.initialize_all_variables())
for i in range(20000):
    batch = mnist.train.next_batch(50)
    if i%100 == 0:
        train_accuracy = accuracy.eval(feed_dict={
        x:batch[0], y_: batch[1], keep_prob: 1.0})
        print("step %d, training accuracy %g"%(i, train_accuracy))
    train_step.run(feed_dict={x: batch[0], y_: batch[1], keep_prob: 0.5})

print("test accuracy %g"%accuracy.eval(feed_dict={
    x: mnist.test.images, y_: mnist.test.labels, keep_prob: 1.0}))
```

运行结果：


```
Extracting MNIST_data\train-images-idx3-ubyte.gz
Extracting MNIST_data\train-labels-idx1-ubyte.gz
Extracting MNIST_data\t10k-images-idx3-ubyte.gz
Extracting MNIST_data\t10k-labels-idx1-ubyte.gz
2017-12-25 20:43:07.185964: I C:\tf_jenkins\home\workspace\rel-win\M\windows-gpu\PY\35\tensorflow\core\platform\cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX AVX2
2017-12-25 20:43:07.446422: I C:\tf_jenkins\home\workspace\rel-win\M\windows-gpu\PY\35\tensorflow\core\common_runtime\gpu\gpu_device.cc:1030] Found device 0 with properties:
name: GeForce GTX 960 major: 5 minor: 2 memoryClockRate(GHz): 1.2025
pciBusID: 0000:01:00.0
totalMemory: 4.00GiB freeMemory: 3.33GiB
2017-12-25 20:43:07.446715: I C:\tf_jenkins\home\workspace\rel-win\M\windows-gpu\PY\35\tensorflow\core\common_runtime\gpu\gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: GeForce GTX 960, pci bus id: 0000:01:00.0, compute capability: 5.2)
WARNING:tensorflow:From C:\Users\Leo\AppData\Local\Programs\Python\Python35\lib\site-packages\tensorflow\python\util\tf_should_use.py:107: initialize_all_variables (from tensorflow.python.ops.variables) is deprecated and will be removed after 2017-03-02.
Instructions for updating:
Use `tf.global_variables_initializer` instead.
step 0, training accuracy 0.12
step 100, training accuracy 0.88
step 200, training accuracy 0.9
step 300, training accuracy 0.8
step 400, training accuracy 0.92
step 500, training accuracy 0.94
step 600, training accuracy 1
step 700, training accuracy 0.94
step 800, training accuracy 0.9
step 900, training accuracy 1
step 1000, training accuracy 0.96
step 1100, training accuracy 1
step 1200, training accuracy 0.98
step 1300, training accuracy 0.98
step 1400, training accuracy 1
step 1500, training accuracy 0.98
step 1600, training accuracy 1
step 1700, training accuracy 1
step 1800, training accuracy 1
step 1900, training accuracy 1
step 2000, training accuracy 0.96
step 2100, training accuracy 0.96
step 2200, training accuracy 1
step 2300, training accuracy 1
step 2400, training accuracy 1
step 2500, training accuracy 1
step 2600, training accuracy 1
step 2700, training accuracy 0.96
step 2800, training accuracy 0.96
step 2900, training accuracy 1
step 3000, training accuracy 1
step 3100, training accuracy 0.98
step 3200, training accuracy 1
step 3300, training accuracy 0.98
step 3400, training accuracy 0.96
step 3500, training accuracy 1
step 3600, training accuracy 1
step 3700, training accuracy 1
step 3800, training accuracy 0.98
step 3900, training accuracy 1
step 4000, training accuracy 1
step 4100, training accuracy 0.98
step 4200, training accuracy 1
step 4300, training accuracy 0.94
step 4400, training accuracy 1
step 4500, training accuracy 1
step 4600, training accuracy 1
step 4700, training accuracy 1
step 4800, training accuracy 0.98
step 4900, training accuracy 0.98
step 5000, training accuracy 0.98
step 5100, training accuracy 0.94
step 5200, training accuracy 1
step 5300, training accuracy 1
step 5400, training accuracy 1
step 5500, training accuracy 1
step 5600, training accuracy 0.98
```

