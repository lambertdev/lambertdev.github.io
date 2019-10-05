---
title: Mithell《机器学习》学习笔记 - Chapter 1&2
tags:
  - Machine Learning
  - 机器学习
url: 1236.html
id: 1236
categories:
  - 机器学习
date: 2018-11-11 08:55:40
---

机器学习
====

绪论
--

### 应用

*   *   语音识别
    *   自动车辆驾驶
    *   西洋双陆棋

*   ....

### 相关学科

*   *   人工智能
    *   贝叶斯方法
    *   计算复杂性理论
    *   信息论
    *   控制论

*   统计学

### 问题的标准描述

> **任务T**: 描述机器学习要做任务
> 
> **性能标准P**: 描述机器学习结果的性能
> 
> **训练经验E**: 用于学习的数据集
> 
> **目标函数\[latex\]V\[/latex\]**: 根据输入得到最优方案（例如，一段语音的最佳匹配、棋局的最佳落子）
> 
> **目标函数的最优近似\[latex\]{\\tilde V}\[/latex\]**: 根据输入得到近似目标函数的最优方案

    训练经验---->**学习器**---->目标函数\[latex\]V\[/latex\] （从当前任务允许的所有合法输入集合中，选出能得到最优性能标准P的输出的唯一输入）。

### 目标函数的描述

    通常来说，由于任务T的复杂性，无法完全得到目标函数，通常机器学习的目标是得到目标函数的某个近似(Approximation) \[latex\]{\\tilde V}\[/latex\] 。     \[latex\]{\\tilde V}\[/latex\] 函数需要可表示，这样才能根据任务的实际状况，评估出输出结果的性能。一般来讲，目标函数输入有多个变量维度。其最简单的一种表示方法为给每个维度的变量赋予一定的权值（weights）\[latex\]w\_0->w\_n\[/latex\]，乘以对应维度变量再求和的一个线性函数。例如：

> \[latex\]{\\tilde V} = w\_0+x\_1w\_1+x\_2w\_2+...+x\_nw_n\[/latex\]

### 目标函数近似函数的训练

    训练的目的，便是确定目标函数的近似函数。以上述函数为例，便是确定目标函数里的权值的确切数值。

#### 训练样例

    为了学习到\[latex\]{\\tilde V}\[/latex\]，需要一系列的训练样例，它们可以表示为：

> \[latex\]<<x\_1=3,x\_2=1,...,x_n=10>, 100>\[/latex\]

    其中100为\[latex\]<x\_1=3,x\_2=1,...,x_n=10>\[/latex\]的性能打分。

#### 权值调整

    训练过程就是对权值进行调整，逐渐逼近目标函数的过程。而调整权值的过程，就是寻找可以对训练集中的所有样例，函数得到的值和训练值之差的平方总和E最小的过程。即：

> \[latex\]E\\equiv\\sum_{\\mathclap{<b, V_{train}(b)> \\in Training Data}} (V_{train}(b)-\\tilde{V}(b))^2\[/latex\]

    权值更新则是根据当前训练值与当前近似函数的计算值的差，进行权值更新。公式如下：

> \[latex\]w_{new}⇠w\_i+\\eta(V\_{train}(b)-\\tilde{V}(b))x_i\[/latex\]

    其中\[latex\]\\eta\[/latex\]为小的常数代表调整幅度。

#### 总结

    用书中的一张图表示训练过程：

![机器学习](http://pic.l2h.site/l2hsiteMachine-Learning-Outline.png)

概念学习 （Concept Learning）
-----------------------

### 概念

> Concept Learning: Inferring a boolean-valued function from training examples of its input and output.

    “概念学习”指的是从有输入输出的训练数据中推导出返回布尔值的函数。 例如：识别物体是否是汽车、鸟类或植物。

### 概念学习任务

\[caption id="" align="aligncenter" width="629"\]![概念学习实例](http://pic.l2h.site/l2hsiteMachine-Learning-Outline-2.png) 机器学习\[/caption\]

    书中根据天气情况判断Aldo是否会做户外运动为实例。输入参数为一元组：<天空状况，空气温度，空气湿度，风力，水温，天气预报>。输出为布尔值：表示Aldo是否做户外运动。具体训练目标定义如下：

给出训练数据，即Aldo以往做户外运动的天气状况。从假设函数集合\[latex\]H\[/latex\]中推导出目标函数h的近似函数，满足\[latex\]h(x)=c(x)\[/latex\]，其中：

*   *   h()和c()函数均为返回布尔值的函数，表示Aldo是否会做户外运动。

*   c()函数为目标函数，事先并不知道，只有以往的训练数据。

#### Inductive Learning Hypothesis （归纳学习假设）

> 若某近似函数在足够大的训练数据集上与目标函数输出结果匹配，我们有理由相信剩余未被观察到的数据上该近似函数也可得到正确输出。

#### CONCEPT LEARNING AS SEARCH（概念学习即搜索）

    概念学习可以认为从一个大的假设空间逐步搜索到最符合训练数据集的假设。

> General-to-Specific Ordering of Hypotheses(假设的一般到特殊序)：以输入参数一元组中的“水温为例”，它可能有Warm、Cool、任意值(\[latex\]?\[/latex\])或者皆不可(\[latex\]\\Theta\[/latex\])。其中对假设的搜索顺序应该是从一般到特殊\[latex\]?\\to(Warm|Cool)\\to\\Theta\[/latex\].

#### FIND-S:寻找最大特殊假设

    按照一般到特殊顺序计算出匹配训练数据集的最一般假设。

#### 概念空间和候选消除算法

    候选消除算法，解决Find-S的部分问题：因为Find-S得到的可能只是诸多满足训练数据集合的一系列假设的集合。候选消除算法则从这些假设中消除不匹配的假设。