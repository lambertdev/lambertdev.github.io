---
title: Wordpress--免修改scandir添加模板页面
tags:
  - wordpress
url: 681.html
id: 681
categories:
  - Wordpress建站
date: 2018-09-28 00:24:21
---

今天打算给博客增加个友链内页，发现新增页面时没有选择模板的栏位。 ![Wordpress--无法修改scandir如何按照模板添加模板页面](http://pic.l2h.site/l2hsitelink-page-6.PNG "Wordpress--无法修改scandir如何按照模板添加模板页面") 搜索了下，说是需要修改php.ini的scandir属性，将其从disable项目里去掉。但博客因为是虚拟主机，后台提供的修改栏位很少，无法更改。 追了下增加页面的源码，发现可以到数据库的post_meta表中增加项目来完成。  好在虚拟主机提供商有数据库在线编辑的功能，具体操作步骤如下：

*   1.  按照网上的介绍从page.php拷贝修改新增模板页，假设名为fr-links.php
    2.  博客后台新建页面
    3.  登录到数据库管理页(比较常见的是phpmyadmin)
    4.  打开wordpress的post表，找到新建页面的id并记下，注意：wordpress会为revision增加表项，所以请选择post_status为publish的表项![Wordpress--无法修改scandir如何按照模板添加模板页面](http://pic.l2h.site/l2hsitelink-page-2.PNG "Wordpress--无法修改scandir如何按照模板添加模板页面") ![Wordpress--无法修改scandir如何按照模板添加模板页面](http://pic.l2h.site/l2hsitelink-page-3.PNG "Wordpress--无法修改scandir如何按照模板添加模板页面")
    5.  打开post_meta表，新增一项（注意meta_id不要重复，且是往后顺延添加），post_id为新增页面的ID（即Post表中的ID）, meta_key填入_wp_page_template，meta_value填入你第一步新增的模板页名字“fr-links.php”![Wordpress--无法修改scandir如何按照模板添加模板页面](http://pic.l2h.site/l2hsitelink-page-1.PNG "Wordpress--无法修改scandir如何按照模板添加模板页面")
    6.  修改成功，效果如下。注意如果以后要在后台修改新增的页面，还要到数据库中对publish那个表项做个修改。![Wordpress--无法修改scandir如何按照模板添加模板页面](http://pic.l2h.site/l2hsitelink-page-4.PNG "Wordpress--无法修改scandir如何按照模板添加模板页面")