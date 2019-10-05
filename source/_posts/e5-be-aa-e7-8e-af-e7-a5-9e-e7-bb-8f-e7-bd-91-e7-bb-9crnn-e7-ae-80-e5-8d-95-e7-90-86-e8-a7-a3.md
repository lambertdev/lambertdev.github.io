---
title: 循环神经网络(RNN)简单理解
tags:
  - RNN
  - 神经网络
url: 2436.html
id: 2436
categories:
  - 机器学习
date: 2019-07-13 16:55:32
---

循环神经网络(Recurrent Nerual Networks,简称RNN)近年来被业界视作处理序列数据以及做自然语言处理的灵丹妙药。其变种LSTM仍是当今最先进的数据处理模型之一。

理解RNN的工作原理，可帮助机器学习人员建立起有效的模型，更好地对数据进行有效的处理。

### 概念

什么是RNN？首先让我们比较下传统前向神经网络与RNN的网络架构。

![](https://blog.floydhub.com/content/images/2019/04/Slide3-1.jpg)

左：传统神经网络架构 右：RNN架构

从图中可以看出，两者的差异主要在于网络是如何接收输入数据的。

* **传统前向神经网络**：接受固定数量（所有数据量）的输入，进行定量的输出
*  **RNN**：并非一次性使用所有的数据作为输入。反之，RNN有多个步骤，每个步骤将一部分数据序列化输入，经过一系列计算产生输出，直到所有的序列结束。

这样讲可能仍然难以理解，下把这张序列化的动图可以辅助大家更好地理解。

![](https://blog.floydhub.com/content/images/2019/04/rnn-2.gif)

上图可以看到，每一步的计算会将上一步的计算输出对应的隐藏状态（Hidden State）作为一部分输入。由此看出，RNN对处理序列化相关的数据有天生的优势。

另外我们可以看到，RNN每一步的神经元计算，是采用相同的网络结构，这是RNN的另外一个重要特点。

### RNN网络的输入输出

您可能会有疑问了：RNN输出来自于网络的哪一步？答案是，这取决于您要解决的问题是什么。例如，如果您用RNN做分类任务，那么您所需要的是从所有输入得到的最终输出；或者您要做单词预测任务，那么您会需要RNN网络序列的每一步都作出输出。

![](https://blog.floydhub.com/content/images/2019/04/karpathy.jpeg)

RNN输出数据多种形式

上图可以看出，RNN是非常灵活的，可以根据您的需要制定RNN的网络模型，喂给网络不同类型的输入，得到不同的输出。

![](https://blog.floydhub.com/content/images/2019/04/Slide6.jpg)

RNN多对一输出

上图示例中，所有时刻的输入，经过RNN网络，得到最后的输出结果。

![](https://blog.floydhub.com/content/images/2019/04/Slide7.jpg)

RNN多对多输出

而本示例中（上图），RNN序列的每一步输出都是我们需要的。除此之外，在例如翻译任务中，我们可能在会先接受多个输入序列，产生一个输出。再根据这个输出，最后产生多个输出序列。如下图英语翻译为法语的示例：

![](https://blog.floydhub.com/content/images/2019/04/Slide8.jpg)

RNN 翻译示例

### RNN单元内部工作原理

到这里，您可能对RNN网络的框架有个基本了解。不过具体每一个RNN单元是如何工作的呢？

首先我们看传给RNN序列下一步的隐藏状态是如何产生的。有如下公式：

[latex] hidden_t = F(hidden_{t-1}, input_t) [/latex]

即当前步的隐藏状态，由上一步的隐藏状态加上这部分的输入经过函数F处理后产生。而第一步的隐藏状态，一般会在整个RNN初始化时人为设置为0。在最简单的RNN中，函数F一般为每个输入乘以对应的权重再用激活函数做非线性处理。激活函数一般有RELU、Sigmoid或tanh。下边公式为采用tanh作为激活函数：

[latex] hidden_t = tanh(W_{hiddent}*hidden_{t-1},W_{input} * input_t) [/latex]

而我们若需要在RNN单元每一步产生一个输出，那么这个输出一般由该步的隐藏单元做一个线性处理产生，例如：

[latex] output_t = W_{output}* hidden_t [/latex]

可以看出，上一个RNN单元的隐藏状态会被传递给下一个RNN单元，如此重复，直到运行到我们设定的停止条件。

当然，这是一个最简单的RNN网络形式。RNN网络还有许多相对复杂的变种（当然是为了针对性解决其他网络形式的一些问题而提出），例如LSTM、GRU等。

### RNN网络的训练

RNN需要经过训练才能学习“”到更精准的拟合，从而得到我们想要的数据。

![](https://blog.floydhub.com/content/images/2019/04/rnn-bptt-with-gradients.png)

RNN网络的权重更新

我们知道，神经网络通过学习和更新网络中的权重来接近最优解（见[本站文章](https://l2h.site/2019/02/02/machine-learning-neural-network-1/)），RNN也不例外。通过一步步后传播算法来减小损失函数（Cost Function）最终得到最小值，RNN的后传播会需要前面RNN单元的数据，本文不作推导。

### 总结

本文对循环神经网络(RNN)的基本框架和原理做了介绍。为方便理解，略去的数学推导。接下来，计划对RNN网络的变形LSTM、GRU等做介绍，并增加Tensorflow代码实现。