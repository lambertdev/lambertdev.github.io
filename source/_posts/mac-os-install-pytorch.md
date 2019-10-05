---
title: MAC OS本地安装PyTorch
tags:
  - python
  - pytorch
  - 机器学习
url: 1400.html
id: 1400
categories:
  - 机器学习
date: 2018-12-02 13:11:42
---

前言
--

PyTorchs是基于Python的机器学习操作库，他可以利用GPU的资源来进行复杂的[深度学习](https://l2h.site/category/machine-learning/)运算。

![pytorch](https://l2h.site/wp-content/uploads/2018/12/PyTorch.jpg)

如果需要充分利用Pytorch的CUDA支持，需要电脑上有NVIDIA GPU。不过本人电脑是Macbook Air，没有这样的条件。且入门学习实验，希望使用CPU支持即可。

官网上说明，Mac OS如果要使用CUDA支持，需要源代码编译PyTorch：

> Currently, CUDA support on macOS is only available by [building PyTorch from source](https://pytorch.org/get-started/locally/#mac-from-source)

安装前准备
-----

### macOS 版本

Mac OS 10.10 (Yosemite) or 更高

### Python

Mac OS is 默认安装的Python 2.7. 一般建议使用Python 3.6或更高版本，这样与PyTorch搭配更好（官方称Python 2.7 也可使用）。Python 3.6可以使用Anaconda 或者Homebrew包管理器安装，或者从Python官网安装 

Anaconda包管理器的安装
---------------

打开Mac OS的终端，执行如下命令即可：

> $curl -O https://repo.anaconda.com/archive/Anaconda3-5.2.0-MacOSX-x86_64.sh  
> $sudo sh Anaconda3-5.2.0-MacOSX-x86_64.sh

安装后，可以在终端输入如下命令验证：

> $which python  
> /Users/xh/anaconda3/bin/python
> 
> $ python --version  
> Python 3.6.5 :: Anaconda, Inc.

安装PyTorch
---------

使用Anaconda安装PyTorch也非常简单：

    conda install pytorch torchvision -c pytorch

验证PyTorch安装
-----------

可将如下代码保存为test.py文件，执行python test.py进行验证

    from __future__ import print_function
    import torch
    x = torch.rand(5, 3)
    print(x)
    

执行后输出类似于:

    tensor([[0.3380, 0.3845, 0.3217],
            [0.8337, 0.9050, 0.2650],
            [0.2979, 0.7141, 0.9069],
            [0.1449, 0.1132, 0.1375],
            [0.4675, 0.3947, 0.1426]]

至此，安装结束。