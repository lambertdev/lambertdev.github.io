---
title: 神经网络和深度学习(2) -- 后向传播算法原理
tags:
  - Machine Learning
  - 机器学习
  - 神经网络
url: 1884.html
id: 1884
categories:
  - 机器学习
date: 2019-02-05 19:56:23
---

前言
--

[神经网络和深度学习(1)](http://neuralnetworksanddeeplearning.com/chap1.html)中，我们从代码中看到机器学习使用了Back Propagation，但是并未介绍到其工作原理。本文则着重介绍Back Propagation算法(以下简称**BP算法**)的工作原理。本文大部分内容引用、整理或翻译自[Michael Nielsen](http://michaelnielsen.org/)的《[Neural Networks and Deep Learning](http://neuralnetworksanddeeplearning.com/index.html) 》，已征得原作者许可，转载此文请先与我联系。作者原声明如下：

> This work is licensed under a [Creative Commons Attribution-NonCommercial 3.0 Unported License](http://creativecommons.org/licenses/by-nc/3.0/deed.en_GB). This means you're free to copy, share, and build on this book, but not to sell it. If you're interested in  
> commercial use, please [contact me](mailto:mn@michaelnielsen.org).

BP算法最早于1970年代提出，但是直到1986年[David Rumelhart](http://en.wikipedia.org/wiki/David_Rumelhart)、[Geoffrey Hinton](http://www.cs.toronto.edu/~hinton/)和 [Ronald Williams](http://en.wikipedia.org/wiki/Ronald_J._Williams)的著名论文才开始获得重视。论文中描述了多个神经网络，而BP神经网络比更早的方法学习得快的多，这为神经网络解决之前无法解决的问题提供了可能。而到了今天，BP算法则成为了神经网络中的“劳模”。

跟其他章节相比，本文数学分析较多。若您对数学分析不是特别感冒，可以跳过本文，把数学原理当成一个黑盒子来看。这样，只要了解本文中的结论就可以，后续的章节学习也并不会受到影响。需要说明的是，BP算法的核心是误差函数(Cost函数）对w的偏导数\[latex\]∂C/∂w\[/latex\]。这个表达式说明了在w和b变化下，Cost的变化速度及其中的细节。因此，BP算法的原理也是非常值得学习的。

![](http://pic.l2h.site/Machine-Learning-Book.jpg)

热身：一个快速的基于矩阵的计算神经网络输出方法
-----------------------

我们使用\[latex\]w_{jk}^l\[/latex\] 来表示l-1层的第k个神经元到l层的第j个神经元的连接权重。例，下图表示了神经网络第二层的第四个神经元到第三层的第二个神经元的连接:

![](http://neuralnetworksanddeeplearning.com/images/tikz16.png)

该定义初看比较麻烦，需要一些时间来理解，掌握之后则发现它比较自然。有人可能比较奇怪j和k的含义：j当做输入层下标，而k为输出的下标不是更直觉更合理吗？之后再做解释。

我们对神经网络的bias（也就是负的阈值）和activation（即输出）也可以采用同样的标识。如下图，\[latex\]b_j^l\[/latex\] 表示第l层第j个感知器的bias，而\[latex\]a_j^l\[/latex\] 表示第l层第j个感知器的activation。

![](http://neuralnetworksanddeeplearning.com/images/tikz17.png)

由上述表示法，可以得到\[latex\]b_j^l\[/latex\] 第l层的第j个神经元的输出，可由第l-1层的输出表示如下：

\[latex\]\\begin{eqnarray} a^{l}_j = \\sigma\\left( \\sum_k w^{l}_{jk} a^{l-1}_k + b^l_j \\right), \\tag{1}\\end{eqnarray} \[/latex\]

可以把上述公式写成向量形式：

\[latex\]\\begin{eqnarray} a^{l} = \\sigma(w^l a^{l-1}+b^l). \\tag{2}\\end{eqnarray} \[/latex\]

从该公式不难看出，第l层的神经元输出，由前一层的所有神经元输出作为矩阵乘以对应的权重矩阵加上bias之后，代入σ函数即可。该公式非常简洁，在实际使用中也非常有用（有许多库提供了快速计算向量的积、求和等方法）。

在计算\[latex\]a^l\[/latex\]过程中，有先计算中间变量\[latex\]z^l\\equiv\\sigma(w^l a^{l-1}+b^l)\[/latex\]。该变量在后续章节中非常有用，我们给其命名为_带权输入（weight input）_。需要注意\[latex\]z^l\[/latex\]也是向量，由如下向量元素组成：\[latex\]z^l_j = \\sum_k w^l_{jk} a^{l-1}_k+b^l_j\[/latex\]，其中\[latex\]z^l_j\[/latex\]是第l层第j个元素的带权输入。

误差函数（Cost Function）
-------------------

本章我们仍然使用[前一章](https://www.l2h.site/2019/02/02/machine-learning-neural-network-1/)的误差函数：

\[latex\]\\begin{eqnarray} C(w,b) \\equiv \\frac{1}{2n} \\sum_x \\| y(x) - a\\|^2 \\nonumber\\end{eqnarray}\[/latex\]

另\[latex\]C_x = \\frac{1}{2} \\|y-a^L \\|^2\[/latex\]，上述误差函数可重写为：\[latex\]C = \\frac{1}{n} \\sum_x C_x\[/latex\]。同时，误差函数也可以写成对输出\[latex\]a^L\[/latex\]的函数（其也显然是），如下图：

![](http://neuralnetworksanddeeplearning.com/images/tikz18.png)

哈达马积（The Hadamard product）
--------------------------

与一般矩阵求积不同，哈达马积（表示为\[latex\]{s}\\odot{t }\[/latex\]）为求向量见对应元素位置的乘积，例：

\[latex\]\\begin{eqnarray} \\left\[\\begin{array}{c} 1 \\\ 2 \\end{array}\\right\] \\odot \\left\[\\begin{array}{c} 3 \\\ 4\\end{array} \\right\] = \\left\[ \\begin{array}{c} 1 * 3 \\\ 2 * 4 \\end{array} \\right\] = \\left\[ \\begin{array}{c} 3 \\\ 8 \\end{array} \\right\]. \\end{eqnarray} \[/latex\]

哈达马积也被称为_Schur product_。

BP算法的四个基本公式
-----------

BP算法的核心主要是理解网络中的权重和bias是怎么对误差函数进行影响的。说到底，是为了计算网络中的偏导数\[latex\]\\partial C / \\partial w^l_{jk}和 \\partial C / \\partial b^l_j\[/latex\]。为了计算这些偏导数，我们首先介绍一个中间数，\[latex\]delta^l_j\[/latex\]，它表示第l层第j个神经元的_error（误差）_。

BP算法首先计算\[latex\]delta^l_j\[/latex\]，然后将它与\[latex\]\\partial C / \\partial w^l_{jk} 和 \\partial C / \\partial b^l_j\[/latex\]相关联。

为了理解_error_是如何定义的，我们先想象神经网络中有一个小怪兽，位于l层的第j个神经元。当神经元的输入层有输入时，小怪兽对神经元的操作进行破坏--对神经元的带权输入产生\[latex\]\\Delta z^l_j\[/latex\]的变化。这样神经元的输出由原来的\[latex\]\\sigma(z^l_j)\[/latex\]变为了\[latex\]\\sigma(z^l_j+\\Delta z^l_j)\[/latex\]。这个变化向网络中的后边层传播，最后导致误差函数的结果产生\[latex\]\\frac{\\partial C}{\\partial z^l_j} \\Delta z^l_j\[/latex\]大小的变化。

倘若小怪兽是做好事的小怪兽，希望找到\[latex\]\\Delta z^l_j \[/latex\]以帮助我们优化误差函数的结果：假设\[latex\]\\frac{\\partial C}{\\partial z^l_j} \[/latex\] 很大，小怪兽会仔细挑选\[latex\]\\Delta z^l_j\[/latex\]，让它和\[latex\]\\frac{\\partial C}{\\partial z^l_j} \[/latex\]的正负值相反，以此很大程度地降低误差。相反，如果\[latex\]\\frac{\\partial C}{\\partial z^l_j}\[/latex\]接近为0，小怪兽是没办法通过影响带权输入\[latex\]z^l_j\[/latex\]的值来优化误差函数。也就是说，该神经元已经接近最优了。因此，给我们的直觉，\[latex\]\\frac{\\partial C}{\\partial z^l_j}\[/latex\]就是表示神经元误差的一个度量方式。

因此，我们定义误差\[latex\]\\begin{eqnarray} \\delta^l_j \\equiv \\frac{\\partial C}{\\partial z^l_j}. \\tag{29}\\end{eqnarray} \[/latex\]。

根据本文惯例，我们使用\[latex\]\\delta^l\[/latex\]表示第l层的误差向量。BP算法将会用一些公式来计算每一层的\[latex\]\\delta^l\[/latex\]，并将这些误差与\[latex\]\\partial C / \\partial w^l_{jk} 和 \\partial C / \\partial b^l_j\[/latex\]相关联。

言归正传，我们来看看BP公式的算法：

### BP1

\[latex\]\\begin{eqnarray} \\delta^L_j = \\frac{\\partial C}{\\partial a^L_j} \\sigma'(z^L_j). \\tag{BP1}\\end{eqnarray} \[/latex\]

公式BP1为微积分里一个基本的导数公式，比较容易理解。写成哈达马积形式如下：

\[latex\]\\begin{eqnarray} \\delta^L = \\nabla_a C \\odot \\sigma'(z^L). \\end{eqnarray}\[/latex\]

其中\[latex\]\\nabla_a C\[/latex\] 为误差函数对所有\[latex\]a\[/latex\] 的偏导数组成的向量。其中，\[latex\]\\nabla_a C=(a^L-y)\[/latex\].

### BP2

\[latex\]\\begin{eqnarray} \\delta^l = ((w^{l+1})^T \\delta^{l+1}) \\odot \\sigma'(z^l), \\tag{BP2}\\end{eqnarray} \[/latex\]

证明如下：

\[latex\]\\begin{eqnarray} \\delta^l_j & = & \\frac{\\partial C}{\\partial z^l_j} \\\ & = & \\sum_k \\frac{\\partial C}{\\partial z^{l+1}_k} \\frac{\\partial z^{l+1}_k}{\\partial z^l_j} \\\ & = & \\sum_k \\frac{\\partial z^{l+1}_k}{\\partial z^l_j} \\delta^{l+1}_k, \\end{eqnarray}\[/latex\]

其中：

\[latex\]\\begin{eqnarray} z^{l+1}_k = \\sum_j w^{l+1}_{kj} a^l_j +b^{l+1}_k = \\sum_j w^{l+1}_{kj} \\sigma(z^l_j) +b^{l+1}_k. \\end{eqnarray} \[/latex\]

求导得到：

\[latex\]\\begin{eqnarray} \\frac{\\partial z^{l+1}_k}{\\partial z^l_j} = w^{l+1}_{kj} \\sigma'(z^l_j). \\end{eqnarray}\[/latex\]

合并以上公式得到BP2：

\[latex\]\\begin{eqnarray} \\delta^l_j = \\sum_k w^{l+1}_{kj} \\delta^{l+1}_k \\sigma'(z^l_j). \\end{eqnarray} \[/latex\]

### BP3和BP4

\[latex\]\\begin{eqnarray} \\frac{\\partial C}{\\partial b^l_j} = \\delta^l_j. \\tag{BP3}\\\ \\frac{\\partial C}{\\partial w^l_{jk}} = a^{l-1}_k \\delta^l_j. \\tag{BP4}\\end{eqnarray} \[/latex\]

证明BP4如下：

\[latex\]\\begin{eqnarray}\\frac{\\partial C}{\\partial w^l_{jk}} = \\frac{\\partial C}{\\partial z^l_j} \\frac{\\partial z^l_j}{\\partial w^l_{jk}}. \\end{eqnarray}\[/latex\]

其中：

\[latex\]\\frac{\\partial C}{\\partial z^l_j}=\\delta^l_j\\\ z^l_j = \\sum_k w^l_{jk}a^{l-1}_k\\\\\frac{\\partial z^l_j}{\\partial w^l_{jk}}=a^{l-1}_k\[/latex\]

而BP3的证明类同

有了BP1到BP4四个公式，我们就能根据输入数据一步步来更新各层的w和b（注意：这里是输出层先计算，然后输入层再计算。这也是BP算法中back的意义）。算法对应代码可在[上一章代码实例](https://www.l2h.site/2019/02/02/machine-learning-neural-network-1/#i-7)处查阅。

总结
--

这里主要介绍BP算法的数学原理。下一章节会介绍算法中各个初始化参数以及误差函数（cost function）的选择对结果的影响。