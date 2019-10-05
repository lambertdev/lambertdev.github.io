---
title: Wordpress-使用七牛云存储测试域名过期问题解决
tags:
  - wordpress
  - 七牛
  - 图床
url: 750.html
id: 750
categories:
  - Wordpress建站
date: 2018-09-30 14:52:50
---

![Wordpress-使用七牛云存储测试域名过期问题解决](http://pic.l2h.site/l2hsitec9fcc3cec3fdfc03222bc595dd3f8794a4c2264f.jpg "Wordpress-使用七牛云存储测试域名过期问题解决")博客目前空间很小，一直使用的是[七牛云](https://portal.qiniu.com/signup?code=3leqs5tm1essy)做免费图床 今天登录控制台，发现七牛云公告，测试域名会有过期时间。好在七牛提供了绑定自己所需域名的方法，操作步骤如下：

 七牛融合 CDN 测试域名（以 clouddn.com/qiniucdn.com/qiniudn.com/qnssl.com/qbox.me 结尾），每个域名每日限总流量 10GB，每个测试域名自创建起 30 个自然日后系统会自动回收，仅供测试使用，详情查看 [七牛测试域名使用规范](https://developer.qiniu.com/fusion/kb/1319/test-domain-access-restriction-rules) 。点击域名可查看剩余回收时间。

1\. 七牛云后台添加域名：

*   绑定域名-->“加速域名”栏位填你自己独立的二级域名（例，http://pic.l2h.site），注意不要和你网站域名管理列表的解析记录有重复
*   其他栏位保留默认
*   此时七牛会返回一个cname记录pic.l2h.site.qiniudns.com请你添加

2\. 域名服务商处增加cname

*   增加pic.l2h.site CNAME解析到pic.l2h.site.qiniudns.com
*   等待一段时间后，域名解析记录生效，自动更新完成

3\. 博客后台图床插件（我使用的是[七牛云图床](http://www.75271.com/)插件）处，修改七牛绑定域名为http://pic.l2h.site 4. 根据自己需求，调整之前博客内容中已使用七牛测试域名为新的七牛绑定域名（可以使用php脚本操作） 以上完成后，也不用担心七牛测试域名过期问题了