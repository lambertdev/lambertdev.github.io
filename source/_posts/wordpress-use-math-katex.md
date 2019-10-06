---
title: Wordpress显示数学公式：WP-KaTeX插件 + VS Code Markdown Math插件
tags:
  - wordpress
url: 1175.html
id: 1175
categories:
  - 建站
date: 2018-11-04 09:35:43
mathjax: true
---

前言
--

    最近希望在了解一些机器学习的相关知识，并在博客里做一些记录。先从著名的Tom Mitchell大神的《机器学习》开始。奈何发现数学公式过多，用[WORDPRESS使用MARKDOWN编辑器](http://www.l2h.site/2018/10/28/markdown-editor/ "http://www.l2h.site/2018/10/28/markdown-editor/")里介绍的方法做笔记，想添加公式发现力不从心。经过搜索，摸索到后文的方法。

本地编辑：VS Code + “Markdown+Math”插件
--------------------------------

### 工具准备

1.  1.  打开Visual Studio Code, Ctrl+Shift+X打开插件搜索框

1.  安装并重启VS Code

![Markdown+Math](http://pic.www.l2h.site/l2hsiteMarkdown-Math-1.png "Markdown+Math")

### 工具使用

    Markdown+Math让使用者可以输入LaTex格式的数学公式，使用数学公式渲染库KaTeX输出公式的样式。 使用Markdown+Math非常简单，只需要在美元符号中间输入需要的Tex数学公式即可。

*   *   若是一行文本内的插入公式，公式前后各一个美元符号，例如："$a^2+b^2=c^2$".

*   若是单行仅公式，则使用$$公式$$。例：

$${\\tilde V} = w_0+x_1w_1+x_2w_2+...+x_nw_n$$

    更多LaTeX数学公式可以参考[LaTex简介文档](http://pic.www.l2h.site/l2hsitelatex-short-cn.pdf "http://pic.www.l2h.site/l2hsitelatex-short-cn.pdf")的"数学公式"一节。

发布到Wordpress
------------

    由于没有对应的CSS档，如果将上文里的公式直接拷贝到Wordpress编辑器，漂亮的数学公式立马失效。 ![WP-KaTex](http://pic.www.l2h.site/l2hsiteMarkdown-Math-2.png "WP-KaTex")

          这时候需要Wordpress插件出场。

*   *   在Wordpress后台搜索"WP-KaTeX"插件并安装

*   *   启用插件

*   建议在插件设置页面勾选上“Use jsDelivr to load files”，使用CDN加速公式显示公式的脚本加载

![WP-KaTex](http://pic.www.l2h.site/l2hsiteMarkdown-Math-3.png "WP-KaTex")

*   数学公式的左右增加“(latex)”和“(/latex)”，()换成[]例如：

> $${\\tilde V} = w_0+x_1w_1+x_2w_2+...+x_nw_n$$

*   KaTex支持大部分$${\\LaTeX}$$数学公式，具体可以参考$$\\href{https://katex.org/}{\\LaTeX}$$ 和 [https://katex.org/docs/supported.html](https://katex.org/docs/supported.html)

     这边可以用Notepad++的查找/替换功能+正则表达式做语法替换，替换规则参考：

*   *   查找“\\$\\$(.+)\\$\\$”

*   *   替换为(latex)**\\1**(\\latex) -> 注意:本博客latex相关语法插件已经生效，圆括号()要换成方括号[]

*   参考下图

![正则替换](http://pic.www.l2h.site/l2hsiteMarkdown-Math-4.png)

    查找样式表示查找由两边各两个美元符号包含的任意样式，圆括号表示要标记的字段，接下来会引用。因为美元符号在正则表达式有特殊含义，查找其必须使用斜杠做转义。 替换样式中最关键的就是红色字体\\1了，表示刚刚查找样式的第一个圆括号标记。 更多正则表达式语法可以参考[这里](http://www.runoob.com/regexp/regexp-tutorial.html "http://www.runoob.com/regexp/regexp-tutorial.html")。

总结
--

    Wordpress原生对Markdown支持较差，目前所使用方法为暂解。希望之后Wordpress可以增加Markdown支持吧。

        另外，机器学习入门解惑推荐博友OldPAN的<[一篇文章解决机器学习入门疑惑](https://oldpan.me/archives/machine-deeplearning-introduction)>