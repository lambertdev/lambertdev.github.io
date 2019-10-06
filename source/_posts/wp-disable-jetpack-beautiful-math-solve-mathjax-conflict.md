---
title: WP禁用Jetpack自带Beautiful Math解决MathJax冲突
tags:
  - Jetpack
  - wordpress
  - 数学
url: 2038.html
id: 2038
categories:
  - 建站
date: 2019-03-01 22:23:40
---

问题
--

前段时间给博客增加了Jetpack插件，今天突然发现[文章](https://www.l2h.site/2019/02/05/machine-learning-neural-network-2/)里数学公式大部分变成了下边样式的图片：

![](https://s0.wp.com/latex.php?latex=%5CLaTeX%26s%3DX&bg=F9F9F9&fg=333333&s=0)

排查
--

一开始怀疑是博客的Cache插件问题，不过经过逐个关闭插件排查，发现是Jetpack的Beautiful Math模块在作怪。看看[Jetpack官网](https://jetpack.com/support/beautiful-math-with-latex/)的说明：

> If your LaTex  code is broken, instead of the equation you’ll see an ugly yellow and red error message. Sorry, **we can’t provide support for **LaTex **syntax**, but there are plenty of useful guides elsewhere online. Or a quick post in our [forums](https://en.forums.wordpress.com/) might find you a solution. One thing to keep in mind is that WordPress puts all of your  LaTex code inside a  LaTex `math` environment. If you try to use LaTex that doesn’t work inside the `math` environment (such as `\begin{align} ... \end{align}`), you will get an error

也就是说，我的MathJax-Latex插件并没有生效，反而被Jetpack的Beautiful Math代替了，而其对一些LaTex公式并不支持。

解决
--

解决方案也很简单，若不希望更换公式的写法为Beautiful Math，只需要关掉它就好：

*   打开“你的网址/+/wp-admin/admin.php?page=jetpack_modules”
*   找到“Beautiful Math”或者中文“ 美妙的数学 ”
*   禁用之

再打开您的页面，问题是否解决了呢？XD