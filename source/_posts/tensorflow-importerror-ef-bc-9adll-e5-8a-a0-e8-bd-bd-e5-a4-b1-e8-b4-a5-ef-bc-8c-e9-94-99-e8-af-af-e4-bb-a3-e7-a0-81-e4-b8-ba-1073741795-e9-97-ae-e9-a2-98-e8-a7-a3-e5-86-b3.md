---
title: TensorFlow ImportError：DLL加载失败，错误代码为-1073741795 问题解决
tags:
  - tensorflow
url: 2291.html
id: 2291
categories:
  - 机器学习
date: 2019-06-28 12:59:01
---

问题
--

赛扬J3160 CPU，使用pip install tensorflow安装好tensorflow，运行如下代码，

import tensorflow as tf
import math
import pandas as pd
import numpy as np
from tensorflow.python.data import Dataset
from matplotlib import pyplot as plt
import seaborn as sns

得到错误如下：

ImportError: Traceback (most recent call last):
  File "C:\\Users\\Administrator.USER-20190627CO\\AppData\\Local\\Programs\\Python\\Python37\\lib\\site-packages\\tensorflow\\python\\pywrap_tensorflow.py", line 58, in <module>
    from tensorflow.python.pywrap\_tensorflow\_internal import *
  File "C:\\Users\\Administrator.USER-20190627CO\\AppData\\Local\\Programs\\Python\\Python37\\lib\\site-packages\\tensorflow\\python\\pywrap\_tensorflow\_internal.py", line 28, in <module>
    \_pywrap\_tensorflow\_internal = swig\_import_helper()
  File "C:\\Users\\Administrator.USER-20190627CO\\AppData\\Local\\Programs\\Python\\Python37\\lib\\site-packages\\tensorflow\\python\\pywrap\_tensorflow\_internal.py", line 24, in swig\_import\_helper
    \_mod = imp.load\_module('\_pywrap\_tensorflow_internal', fp, pathname, description)
  File "C:\\Users\\Administrator.USER-20190627CO\\AppData\\Local\\Programs\\Python\\Python37\\lib\\imp.py", line 242, in load_module
    return load_dynamic(name, filename, file)
  File "C:\\Users\\Administrator.USER-20190627CO\\AppData\\Local\\Programs\\Python\\Python37\\lib\\imp.py", line 342, in load_dynamic
    return _load(spec)
ImportError: DLL load failed with error code -1073741795

找到[github上](https://github.com/tensorflow/tensorflow/issues/17386)有人碰到类似错误，原因是CPU缺少 AVX 指令集支持（看来是赛扬处理器稍低端了）。

解决
--

解法也很简单，依次执行如下步骤:

*   pip uninstall tensorflow
*   到[github](https://github.com/fo40225/tensorflow-windows-wheel)下不支持AVX指令集的tensorflow轮子。一般选最新版的tensorflow；注意您的python版本，若是3.7，就选1.x.0/py37/CPU/sse2下的wheel
*   下好到本地后执行“pip install 刚刚下好wheel的本地路径”