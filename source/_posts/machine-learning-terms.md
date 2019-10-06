---
title: 机器学习术语归纳
url: 2366.html
id: 2366
categories:
  - 机器学习
date: 2019-07-15 15:30:50
---

初学机器学习，往往容易淹没在浩瀚的属于中，本文归纳总结一下机器学习相关的术语，帮您更好理解神经网络

本文大部分翻译自[wildml.com](http://www.wildml.com/deep-learning-glossary/)

### A

#### Activation Function（激活函数）

使用非线性函数对训练模型中的输出（当然不限于最终输出）进行非线性化处理，这样神经网络可以学习到复杂的决策边界。常用的激活函数包括  [sigmoid](https://en.wikipedia.org/wiki/Sigmoid_function), [tanh](http://mathworld.wolfram.com/HyperbolicTangent.html), [ReLU (Rectified Linear Unit)](http://www.wildml.com/deep-learning-glossary/#relu)以及众多的变种.

#### Adadelta

一直基于梯度下降的学习算法，可以自适应调整参数的学习速率。作为  
 [Adagrad](http://www.wildml.com/deep-learning-glossary/#adagrad) 的变种，对超参数敏感，容易造成学习速率过快下降。可以作为标准  
 [SGD](http://www.wildml.com/deep-learning-glossary/#sgd)替代。相关文献：

*   [ADADELTA: An Adaptive Learning Rate Method](http://arxiv.org/abs/1212.5701)
*   [Stanford CS231n: Optimization Algorithms](http://cs231n.github.io/neural-networks-3/)  
    
*   [An overview of gradient descent optimization algorithms](http://sebastianruder.com/optimizing-gradient-descent/)

#### Adagrad

Adagrad是一种自适应调整学习速率的算法，它会追踪梯度平方的变化，并对学习速率做自适应调整。对稀疏数据处理非常有效（会对不常更新的参数加快学习速率）。

*   [Adaptive Subgradient Methods for Online Learning and Stochastic Optimization](http://www.magicbroom.info/Papers/DuchiHaSi10.pdf)  
    
*   [Stanford CS231n: Optimization Algorithms](http://cs231n.github.io/neural-networks-3/)  
    
*   [An overview of gradient descent optimization algorithms](http://sebastianruder.com/optimizing-gradient-descent/)

#### Adam

类似于 [rmsprop](http://www.wildml.com/deep-learning-glossary/#rmsprop) 的学习速率更新算法，更新主要采取即时的第一和第二时刻平均值。另外算法也包括了bias纠正单元，相关文献：

*   [Adam: A Method for Stochastic Optimization](http://arxiv.org/abs/1412.6980)
*   [An overview of gradient descent optimization algorithms](http://sebastianruder.com/optimizing-gradient-descent/)

#### Affine Layer

神经网络的一种全连接层。Affine的含义是：每个上层的神经元链接当前层的神经元，即这是标准的神经网络层。Affine层通常会与 [Convolutional Neural Networks](http://www.wildml.com/deep-learning-glossary/#cnn) 或者 [Recurrent Neural Networks](http://www.wildml.com/deep-learning-glossary/#rnn)  一起使用，用于最终产生一个决策。函数形式通常是\[latex\]y=f(Wx+b)\[/latex\]。W,X,b分别是权值，输入和偏移向量。f通常为非线性函数

#### Attention Mechanism

Attention Mechanisms（注意力机制）灵感的源于人类视觉注意力特点 ：可以关注图片上的特定某个区域。注意力机制可以与自然语言处理或者图片识别结构一起工作，帮助神经网络学习到进行决策时该“注意”到哪些部分。相关文献：

*   [Attention and Memory in Deep Learning and NLP](http://www.wildml.com/2016/01/attention-and-memory-in-deep-learning-and-nlp/)

#### Alexnet

Alexnet是大优势赢得2012年ILSVRC竞赛使用的CNN架构，它使大家重新对使用_**CNN网络**_识别图片的产生兴趣。它由5层卷积层组成，部分卷积层后跟随池化层，最后的全连接层是1000路的softmax分类。Alexnet的介绍见[ImageNet Classification with Deep Convolutional Neural Networks](http://papers.nips.cc/paper/4824-imagenet-classification-with-deep-convolutional-neural-networks).

#### Autoencoder

Autoencoder是一种神经网络模型，目标为通过网络中的一些“瓶颈”来预测网络的输入。通过引入瓶颈，强制网络学习到输入的低维度映射，从而有效地压缩输入维度。Autoencoders与PCA即一些其他降维技术有关，因为其本质上的非线性化特点，可以处理更复杂的映射。现有大量的autoencoder架构，包括[Denoising Autoencoders](http://www.jmlr.org/papers/volume11/vincent10a/vincent10a.pdf),、[Variational Autoencoders](http://arxiv.org/abs/1312.6114),或者[Sequence Autoencoders](http://arxiv.org/abs/1511.01432)

#### Average-Pooling

Average-Pooling是**_卷积神经网络_**识别图片采用的一种池化技术。工作原理为使用小于图片的窗口在图片特征上进行滑动，取得滑动位置上数值的平均值。从而降低数据特征的维度，同时有效保持数据的特征。与之类似的有最大值池化等方法。

### B

#### Backpropagation（逆传播，后向传播）

Backpropagation是有效计算神经网络梯度的方法。它通过微分运算，有效地将误差从输出位置传递到输入位置。它与上世纪70年代开始被使用。文献：

*   [Calculus on Computational Graphs: Backpropagation](http://colah.github.io/posts/2015-08-Backprop/)

#### Backpropagation Through Time (BPTT)

Backpropagation Through Time ([paper](http://axon.cs.byu.edu/~martinez/classes/678/Papers/Werbos_BPTT.pdf))是**_循环神经网络_**使用的逆传播算法。RNN的网络结构与传统的网络结构不同（每个阶段的神经单元共享参数），因此采用的逆传播也稍后差异。相关介绍见

*   [Backpropagation Through Time: What It Does and How to Do It](http://axon.cs.byu.edu/~martinez/classes/678/Papers/Werbos_BPTT.pdf)

#### Batch Normalization

Batch Normalization是对神经网络层输入数据进行小批量分组使用的技术。使用小批量数据分组而非完整数据包可以加速训练速度。其在**_卷积神经网络_**或者_**前向神经网络**_使用中被证明非常有效，不过其目前在**循环神经网络**的使用中，效果有限

*   [Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift](http://arxiv.org/abs/1502.03167)
*   [Batch Normalized Recurrent Neural Networks](http://arxiv.org/abs/1510.01378)

#### Bidirectional RNN

双向循环神经网络是包含两个不同走向循环神经网络的网络。正向RNN从前向后读取输入序列，逆向RNN反之。两个RNN互相交叠，输出由这两个RNN的隐藏层的状态决定。双向RNN主要被用于自然语言处理问题（例，处理一个单词需要考虑单词前后的单词）。相关文献

*   [Bidirectional Recurrent Neural Networks](http://www.di.ufpe.br/~fnj/RNA/bibliografia/BRNN.pdf)

### C

#### Caffe

[Caffe](http://caffe.berkeleyvision.org/) 是 Berkeley Vision和Learning Center开发的深度学习框架，在处理视觉处理问题和CNN模型方面非常有用。

#### Categorical Cross-Entropy Loss

分类交叉熵损失也被称作负对数似然，它是处理分类问题或者评估概率分布相似性的方法，特别是用于评估真值标签。其公式为\[latex\]L = -sum(y * log(y\_prediction))\[/latex\]，其中y是真标签的概率分布（独热向量），\[latex\]y\_prediction\[/latex\]是已预测标签的概率分布（一般使用softmax函数）

#### Channel

深度学习模型的输入数据可以有多个通道。例如，图片有RGB三个通道。因此图片可以被一个3维张量表示，分别是通道、高度和宽度。自然语言处理数据也有多个通道的概念。例如，数据有不同类别的_**嵌入**_表示。

#### Convolutional Neural Network (CNN, ConvNet)

卷积神经网络使用卷积层从输入数据中提取有效特征。通常卷积神经网络由卷积、池化和全连接层组成。因为其在视觉处理任务的出色表现，卷积神经网络近年来一直非常流行。相关文章：

*   [Stanford CS231n class – Convolutional Neural Networks for Visual Recognition](http://cs231n.github.io/)
*   [Understanding Convolutional Neural Networks for NLP](http://www.wildml.com/2015/11/understanding-convolutional-neural-networks-for-nlp/)

### D

#### Deep Belief Network (DBN)

深度信念网络，通过无监督的概率图模型来学习数据特征。DBN由多个隐层组成，前后隐层的神经元间相互连接。每层神经网络由受限玻尔兹曼机组成，分别进行训练层。

*   [A fast learning algorithm for deep belief nets](https://www.cs.toronto.edu/~hinton/absps/fastnc.pdf)

#### Deep Dream

Google发明的一项技术，对深度卷积神经网络学习到的数据进行提取，并用于生成新图片、修改图片甚至给图片加入梦幻般的效果。相关资料：

*   [Deep Dream on Github](https://github.com/google/deepdream)
*   [Inceptionism: Going Deeper into Neural Networks](http://googleresearch.blogspot.ch/2015/06/inceptionism-going-deeper-into-neural.html)

#### Dropout

随机失活是神经网络中用于避免过拟合的一种方法。最早被用于CNN网络，目前被广泛使用到其他神经网络中。相关资料：

*   [Dropout: A Simple Way to Prevent Neural Networks from Overfitting](https://www.cs.toronto.edu/~hinton/absps/JMLRdropout.pdf)
*   [Recurrent Neural Network Regularization](http://arxiv.org/abs/1409.2329)

### E

#### Embedding（嵌入）

嵌入指的是将单词或者句子映射成向量形式。比较流行的嵌入是单词嵌入（例如，[word2vec](http://www.wildml.com/deep-learning-glossary/#word2vec) 或 [GloVe](http://www.wildml.com/deep-learning-glossary/#glove)）。我们也可以嵌入句子、段落或者图片。比如说，通过映射图片和他们的文字描述到嵌入空间来减少他们之间的距离，来将图片和对应的标签进行关联。嵌入可以单独进行（如采用[word2vec](http://www.wildml.com/deep-learning-glossary/#word2vec)），也可以作为某个机器学习任务的一部分，例如情感分析。通常，神经网络的输入均是已经训练和优化过的数据。

#### Exploding Gradient Problem（梯度爆炸问题）

梯度爆炸问题与梯度消失问题正好相反。在深度神经网络中，逆传播过程可能会造成梯度爆炸从而产生数字溢出。一种解决梯度爆炸的方式是[梯度修剪](http://www.wildml.com/deep-learning-glossary/#gradient-clipping)

*   [On the difficulty of training recurrent neural networks](http://arxiv.org/abs/1211.5063)

### F

#### Fine-Tuning

优化调节指的是从另外的任务得到优化过的初始化学习参数。例如，使用[word2vec](http://www.wildml.com/deep-learning-glossary/#word2vec)对自然语言处理任务的单词做预处理

### G

#### Gradient Clipping

梯度修剪主要用于避免深度神经网络（特别是循环神经网络）的梯度爆炸问题。进行梯度修剪的方式有多种，一种常用的方式是对梯度进行L2正则化（new\_gradients = gradients * threshold / l2\_norm(gradients)），参考：

*   [On the difficulty of training recurrent neural networks](http://arxiv.org/abs/1211.5063)

#### GloVe

Glove是一种用于单词嵌入的无监督学习算法。Glove向量和wordvec用途相同，但是表示有差异，这是由于用于嵌入方式不同

*   [GloVe: Global Vectors for Word Representation](http://nlp.stanford.edu/pubs/glove.pdf)

#### GoogleLeNet

赢得2014年ILSVRC挑战的卷积神经网络框架。它使用记忆模块减少参数，同时提升对计算资源的有效利用率。参考：

*   [Going Deeper with Convolutions](http://arxiv.org/abs/1409.4842)

#### GRU

GRU( Gated Recurrent Unit ,门循环单元)是LSTM单元的简化形式，有更少的参数。类似于LSTM神经元，它使用门策略来避免梯度消失问题，使得RNN有效地学习长范围的关联。GRU内部有重置和更新门来决定旧的记忆是否需要保留还是要用当前时间的新值进行更新。参考：

*   [Learning Phrase Representations using RNN Encoder-Decoder for Statistical Machine Translation](http://arxiv.org/abs/1406.1078v3)
*   [Recurrent Neural Network Tutorial, Part 4 – Implementing a GRU/LSTM RNN with Python and Theano](http://www.wildml.com/2015/10/recurrent-neural-network-tutorial-part-4-implementing-a-grulstm-rnn-with-python-and-theano/)

### H

#### Highway Layer

Highway Layer ([论文参考](http://arxiv.org/abs/1505.00387))是使用门策略来控制神经网络层信息流的机制。叠加使用多个Highway层可以训练非常深层次的神经网络。Highway通过门函数选择输入的那个部分通过以及那个部分需要通过变化函数处理。Highway层的基本公式为\[latex\]T * h(x) + (1 - T) * x\[/latex\]，其中T是学习门函数，值位于0和1之间，h(x)是任意输入变化函数，x为输入数据。

### I

#### ICML

 [International Conference for Machine Learning](http://icml.cc/), 机器学习领域顶级会议

#### ILSVRC

[ImageNet Large Scale Visual Recognition Challenge](http://www.image-net.org/challenges/LSVRC/) 是图像识别分类领域最热门的竞赛。

#### Inception Module

记忆单元用于卷积神经网络，提升网络的计算性能。参考：

*   [Going Deeper with Convolutions](http://arxiv.org/abs/1409.4842)

### K

#### Keras[K](http://keras.io/)

[Keras](http://keras.io/)是包含对深度学习进行深度封装的Python库。可以在[TensorFlow](http://www.wildml.com/deep-learning-glossary/#tensorflow), [Theano](http://www.wildml.com/deep-learning-glossary/#theano), 或 [CNTK](https://github.com/Microsoft/CNTK)上层使用

### L

#### LSTM

长期短记忆网络主要用记忆门来避免RNN网络的梯度消失问题。利用LSTM单元计算RNN隐层，可以有效的传递梯度以及学习长范围关联。参考：

*   [Long Short-Term Memory](http://deeplearning.cs.cmu.edu/pdfs/Hochreiter97_lstm.pdf)
*   [Understanding LSTM Networks](http://colah.github.io/posts/2015-08-Understanding-LSTMs/)
*   [Recurrent Neural Network Tutorial, Part 4 – Implementing a GRU/LSTM RNN with Python and Theano](http://www.wildml.com/2015/10/recurrent-neural-network-tutorial-part-4-implementing-a-grulstm-rnn-with-python-and-theano/)

### M

#### Max-Pooling

最大池化是卷积神经网络的一种池化操作，池化时选择特征片段里的最大值，是卷积神经网络的常用池化操作。

### M

#### MNIST

 MNIST 数据集 是最常使用的图像识别数据集了。基本上也是许多机器学习课程的范例数据集，更多介绍直接参考[官网](http://yann.lecun.com/exdb/mnist/)即可。

#### Momentum

Momentum是梯度下降算法的扩展，加速和优化了参数更新过程。实际使用中，加入momentum到梯度下降中，可以使深度网络得到更好的收敛 。参考：

*   [Learning representations by back-propagating errors](http://www.nature.com/nature/journal/v323/n6088/abs/323533a0.html)

#### Multilayer Perceptron (MLP)

多层感知是一种多全连接层的前馈神经网络，使用**激活函数**处理数据做非线性化。MLP是多层神经网络或深度神经网络的最基本形式。

### N

#### Negative Log Likelihood (NLL)

见 [Categorical Cross Entropy Loss](http://www.wildml.com/deep-learning-glossary/#ce-loss).

#### Neural Machine Translation (NMT)

神经机器翻译指的是使用神经网络来翻译语言。参考：

*   [Sequence to sequence learning with neural networks](http://arxiv.org/abs/1409.3215)
*   [Learning Phrase Representations using RNN Encoder-Decoder for Statistical Machine Translation](http://arxiv.org/abs/1406.1078)

#### Neural Turing Machine (NTM)

神经图灵机可以从范例中推导简单的算法。例如，NTM可以从输入输出范例中学习分类算法。在程序运行时神经图灵机通常可以学习到一些处理状态的记忆方法

*   [Neural Turing Machines](http://arxiv.org/abs/1410.5401)

#### Nonlinearity（去线性化）

见**_激活函数_**.

#### Noise-contrastive estimation (NCE)

噪声对比评估是一种在大量词汇表输出常见用来训练分类器的损失抽样方法。通过计算所有可能分类的Softmax是代价昂贵的。而使用NCE，可以有效减少二分类问题的代价，而只需要通过从“真”分布和人工产生的噪声分布区来训练分类器。例如：

*   [Noise-contrastive estimation: A new estimation principle for unnormalized statistical models](http://www.jmlr.org/proceedings/papers/v9/gutmann10a/gutmann10a.pdf)
*   [Learning word embeddings efficiently with noise-contrastive estimation](http://papers.nips.cc/paper/5165-learning-word-embeddings-efficiently-with-noise-contrastive-estimation.pdf)

### P  

#### Pooling

见**_最大池化_**和**_平均池化_**.

#### Restricted Boltzmann Machine (RBN)

受限玻尔兹曼机是深度信念网络使用的一种概率图模型。参考：

*   [Chapter 6: Information Processing in Dynamical Systems: Foundations of Harmony Theory](http://www-psych.stanford.edu/~jlm/papers/PDP/Volume%201/Chap6_PDP86.pdf)
*   [An Introduction to Restricted Boltzmann Machines](http://image.diku.dk/igel/paper/AItRBM-proof.pdf)

### R

#### Recurrent Neural Network (RNN)

RNN代表循环神经网络，参考[本站文章](https://l2h.site/2019/07/13/%e5%be%aa%e7%8e%af%e7%a5%9e%e7%bb%8f%e7%bd%91%e7%bb%9crnn%e7%ae%80%e5%8d%95%e7%90%86%e8%a7%a3/)，或者：

*   [Understanding LSTM Networks](http://colah.github.io/posts/2015-08-Understanding-LSTMs/)
*   [Recurrent Neural Networks Tutorial, Part 1 – Introduction to RNNs](http://www.wildml.com/2015/09/recurrent-neural-networks-tutorial-part-1-introduction-to-rnns/)

#### Recursive Neural Network

递归神经网络是循环神经网络的一种树状形式。详情参见：

*   [Parsing Natural Scenes and Natural Language with Recursive Neural Networks](http://ai.stanford.edu/~ang/papers/icml11-ParsingWithRecursiveNeuralNetworks.pdf)

#### ReLU

线性整流函数（Rectified Linear Unit）是一种激活函数，深度学习中做去线性化处理。参考：

*   [Delving Deep into Rectifiers: Surpassing Human-Level Performance on ImageNet Classification](http://arxiv.org/abs/1502.01852)
*   [Rectifier Nonlinearities Improve Neural Network Acoustic Models](http://web.stanford.edu/~awni/papers/relu_hybrid_icml2013_final.pdf)
*   [Rectified Linear Units Improve Restricted Boltzmann Machines](http://www.cs.toronto.edu/~fritz/absps/reluICML.pdf)

#### ResNet

深度残差网络（Deep Residual Network），赢得了ILSVRC 2015挑战赛，参考。

*   [Deep Residual Learning for Image Recognition](http://arxiv.org/abs/1512.03385)

#### RMSProp

RMSProp是一种基于梯度的优化算法。具体算法介绍见 ：

*   [Neural Networks for Machine Learning Lecture 6a](http://www.cs.toronto.edu/~tijmen/csc321/slides/lecture_slides_lec6.pdf)
*   [Stanford CS231n: Optimization Algorithms](http://cs231n.github.io/neural-networks-3/)
*   [An overview of gradient descent optimization algorithms](http://sebastianruder.com/optimizing-gradient-descent/)

### S

#### Seq2Seq

序列到序列模型读取血量作为输入，产生另外一个序列作为输出。与RNN不同的地方是，在产生输出之前，输入序列被一次性完整的输入。一般使用两个RNN实现，经典应用为机器翻译、编解码等，参考：

*   [Sequence to Sequence Learning with Neural Networks](http://arxiv.org/abs/1409.3215)

#### SGD

随机梯度下降是一种有效的梯度优化算法([Wikipedia](https://en.wikipedia.org/wiki/Stochastic_gradient_descent))，其扩展算法包括 **Momentum**, **Adagrad**, **rmsprop**, **Adadelta** 以及 **Adam**.参考：

*   [Adaptive Subgradient Methods for Online Learning and Stochastic Optimization](http://www.magicbroom.info/Papers/DuchiHaSi10.pdf)
*   [Stanford CS231n: Optimization Algorithms](http://cs231n.github.io/neural-networks-3/)
*   [An overview of gradient descent optimization algorithms](http://sebastianruder.com/optimizing-gradient-descent/)

#### Softmax

[Softmax函数](https://baike.baidu.com/item/Softmax%E5%87%BD%E6%95%B0/22772270?fr=aladdin)，它能将一个含任意实数的K维向量 “压缩”到另一个K维实向量 中，使得每一个元素的范围都在 （0,1）之间，并且所有元素的和为1。主要作为处理分类问题的输出

### T

#### TensorFlow

[TensorFlow](https://www.tensorflow.org/) 是Google提供的开源深度学习框架，可以对数据流图做计算，也封装了现行主要的神经网络运算。支持C++/Python.

#### Theano

[Theano](http://deeplearning.net/software/theano/) 是一个封装深度神经网络算法的Python库

### V

#### Vanishing Gradient Problem

梯度消失问题在深度神经网络学习中越来越常见，特别是循环神经网络，使用较小的梯度（位于0和1直接）。因为梯度在逆传播过程中会相乘，所以会在层与层传递间逐渐“消失”，导致长范围的关联消失。解决方法主要有使用**ReLU**激活，或者使用改进网络**_LSTM_**等。参考：

*   [On the difficulty of training recurrent neural networks](http://www.jmlr.org/proceedings/papers/v28/pascanu13.pdf)

#### VGG

VGG是赢得2014 ImageNet定位和分类跟踪问题第一二名的卷积神经网络。它由16到19个权重层和1\*1或3\*3的小卷积过滤器组成。参考：

*   [Very Deep Convolutional Networks for Large-Scale Image Recognition](http://arxiv.org/abs/1409.1556)

### W

#### word2vec

word2vec用于单词嵌入的算法，参考：

*   [Efficient Estimation of Word Representations in Vector Space](http://arxiv.org/abs/1301.3781)
*   [Distributed Representations of Words and Phrases and their Compositionality](http://arxiv.org/abs/1310.4546)
*   [word2vec Parameter Learning Explained](http://arxiv.org/abs/1411.2738)