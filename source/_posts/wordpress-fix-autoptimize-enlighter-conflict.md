---
title: 'Wordpress:解决Autoptimize+Enlighter插件冲突'
tags:
  - Autoptimze
  - Enlighter
  - Gutenberg
  - wordpress
url: 1440.html
id: 1440
categories:
  - Wordpress建站
date: 2018-12-08 12:10:22
---

最近博客增加了不少图片，头部也增加了不少自定义代码，导致页面打开速度变慢。便在后台给博客增加了Autoptimize插件压缩一下CSS和JS脚本，以减小页面大小。安装后，整体运行速度的确有提高。但是打开类似文章《[LINUX电源管理介绍-1](https://l2h.site/2018/10/24/linux-low-power-1/)》后，发现里边的Enlighter代码高亮全不见了。如下图：

![Wordpress:解决Autoptimize+Enlighter插件冲突](https://l2h.site/wp-content/uploads/2018/12/autoptimize-enlighter-conflict-1.png)

[Bing搜索](http://www.bing.com)，Autoptimize支持特定脚本或者CSS不压缩。可通过以下方式解决：

1. 打开后台，Autoptimize设定

2. 选择右上方高级设定

![Wordpress:解决Autoptimize+Enlighter插件冲突](https://l2h.site/wp-content/uploads/2018/12/autoptimize-enlighter-conflict-2.png)

3. 找到“JavaScript 选项-->从 Autoptimize 排除脚本”（注：后台英文的就查找exclude关键字），添加如下脚本到排除脚本：

> mootools-core-yc.js, EnlighterJS.min.js

![Wordpress:解决Autoptimize+Enlighter插件冲突](https://l2h.site/wp-content/uploads/2018/12/autoptimize-enlighter-conflict-3.png)

4. 保存，并删除页面缓存后，问题解决：

![Wordpress:解决Autoptimize+Enlighter插件冲突](https://l2h.site/wp-content/uploads/2018/12/autoptimize-enlighter-conflict-4.png)

Wordpress:解决Autoptimize+Enlighter插件冲突