title: Tensorflow学习笔记[1]-MINST入门
---

# Tensorflow学习笔记[1]-MINST入门   

![](https://ws1.sinaimg.cn/large/d0b7a0eely1fmoopgnhr6j20ra0arq39.jpg)   

总的来说，训练集和测试集都由图片和标签（数字）组成   
每个标签有10维向量，只有标签对应的维度是1   
图片由28 * 28 = 784个像素组成，由于图片是黑白的   
所以像素值的取值只有0，1   
当然，需要把二维的28 * 28展开成一维的784，要保证每张图片的展开方式相同



导入训练集和测试集 -> 用于评估模型性能   
   
![](https://ws1.sinaimg.cn/large/d0b7a0eely1fmoor251fwj215m0iuwgf.jpg)   
   
![](https://ws1.sinaimg.cn/large/d0b7a0eely1fmoor29y8mj214k0e1glv.jpg)   
   
训练集60000行，测试集10000行   

因此图片集是[60000, 784]的张量，第一个维度索引图片，第二个维度索引每张图片的像素点   

one-hot vector: 除了某一位的数字是1其他都是0   

```
import input_data
import tensorflow as tf
mnist = input_data.read_data_sets("MNIST_data/", one_hot=True)
```



图片集 None表示任意数量的图片

```
x = tf.placeholder("float", [None, 784])
```

W是权重， b是bias，因为要学习W和b的值 所以可以任意设置   
W的维度是[784, 10]，因为要用784维的图片乘以他得到10为的evidence   
矩阵乘法 第一个矩阵的列数等于第二个矩阵的行数才能相乘   
And [60000, 784] * [784, 10] = [60000, 10]   


```
W = tf.Variable(tf.zeros([784, 10]))#数字
b = tf.Variable(tf.zeros([10]))
```


实现模型 SoftMax
SoftMax用于给不同的模型分配概率
比如，图片是9的概率为90%, 8的概率为70%
为了得到某张图片对应的数字的evidence，我们对图片的784个像素加权求和   
如图   
   
![](https://ws1.sinaimg.cn/large/d0b7a0eely1fmoosw8noqj20vt0fzdly.jpg)   
   

0中间的像素值都为0所以权值为红色，负数，而对应的轮廓为正数

同时，也需要一个额外的偏置量(bias)，用来过滤干扰量。   

那么evidence可以表示为：   
   
![](https://ws1.sinaimg.cn/large/d0b7a0eely1fmootcku5sj2093030t8i.jpg)   

比如说是数字1的evidence, 把图片上每个像素值[0, 783]与像素值在1上的权重相称
的和，加一个数字1的偏置量   

SoftMax是一个激励函数，把我们定义的线性函数的输出转换成我们想要得到格式，也就是10个数字的概率分布   
   


建立模型

```
y = tf.nn.softmax(tf.matmul(x, W) + b)
```


训练模型   
指标：成本(cost)或者损失(loss)   
交叉熵(cross-entropy) 函数   
y_用于表示正确值，   

```
y_ = tf.placeholder("float", [None, 10])
```


交叉熵函数   
交叉熵是用来衡量我们的预测用于描述真相的低效性， 即错误率


```
cross_entropy = -tf.reduce_sum(y_*tf.log(y))
```


使用梯度下降算法以0.01的学习速率最小化交叉熵

```
train_step = tf.train.GradientDescentOptimizer(0.01).minimize(cross_entropy)
```



```
init = tf.global_variables_initializer()

sess = tf.Session()
sess.run(init)
```


模型循环训练1000次

```
for i in range(1000):
    batch_xs, batch_ys = mnist.train.next_batch(100)
    sess.run(train_step, feed_dict={x: batch_xs, y_:batch_ys})
```


评估模型   
tf.argmax   
返回bool   

```
correct_prediction = tf.equal(tf.argmax(y, 1), tf.argmax(y_, 1))
```




```
accuracy = tf.reduce_mean(tf.cast(correct_prediction, "float"))

print(sess.run(accuracy, feed_dict={x: mnist.test.images, y_: mnist.test.labels}))
```

运行结果：   
   

```
Successfully downloaded train-images-idx3-ubyte.gz 9912422 bytes.
Extracting MNIST_data/train-images-idx3-ubyte.gz
Successfully downloaded train-labels-idx1-ubyte.gz 28881 bytes.
Extracting MNIST_data/train-labels-idx1-ubyte.gz
Successfully downloaded t10k-images-idx3-ubyte.gz 1648877 bytes.
Extracting MNIST_data/t10k-images-idx3-ubyte.gz
Successfully downloaded t10k-labels-idx1-ubyte.gz 4542 bytes.
Extracting MNIST_data/t10k-labels-idx1-ubyte.gz
2017-12-20 20:11:55.706629: I C:\tf_jenkins\home\workspace\rel-win\M\windows-gpu\PY\35\tensorflow\core\platform\cpu_feature_guard.cc:137] Your CPU supports instructions that this TensorFlow binary was not compiled to use: AVX AVX2
2017-12-20 20:11:56.136245: I C:\tf_jenkins\home\workspace\rel-win\M\windows-gpu\PY\35\tensorflow\core\common_runtime\gpu\gpu_device.cc:1030] Found device 0 with properties: 
name: GeForce GTX 960 major: 5 minor: 2 memoryClockRate(GHz): 1.2025
pciBusID: 0000:01:00.0
totalMemory: 4.00GiB freeMemory: 3.33GiB
2017-12-20 20:11:56.136493: I C:\tf_jenkins\home\workspace\rel-win\M\windows-gpu\PY\35\tensorflow\core\common_runtime\gpu\gpu_device.cc:1120] Creating TensorFlow device (/device:GPU:0) -> (device: 0, name: GeForce GTX 960, pci bus id: 0000:01:00.0, compute capability: 5.2)
0.9187

***Repl Closed***

```
