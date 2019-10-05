---
title: Wordpress如何关联熊掌号
tags:
  - wordpress
  - 熊掌号
  - 百度
url: 1268.html
id: 1268
categories:
  - Wordpress建站
date: 2018-11-11 19:07:17
---

Wordpress关联熊掌号
==============

前言
--

![熊掌号](http://pic.l2h.site/l2hsiteXiongZhangHao-0.jpg)

做站还是希望能得到大家的关注，百度作为国内最大的搜索引擎，给站长提供了熊掌号平台。绑定网站后，百度搜索会倾向于优先展现熊掌号的内容，可以为网站引入一定搜索流量。

通过摸索，得到建立熊掌号的流程如下：

*   *   建立熊掌号

*   *   Wordpress页面改造

*   提交页面

建立熊掌号
-----

### 申请熊掌号

到[熊掌号官网](https://xiongzhang.baidu.com/ "https://xiongzhang.baidu.com/")按照百度的要求进行提交审核即可，一般5个工作日左右，需要耐心等待。

### 熊掌号装潢

装饰页面可以提升熊掌号指数。 可以到“熊掌号后台-->主页管理-->主页装修”进行主页的装饰。如下图，运营位设置为网址链接：

![主页装饰](http://pic.l2h.site/l2hsiteXiongzhanghao-decorate-1.png)“熊掌号后台-->主页管理-->自定义栏目”可以添加专栏文章，专栏名可以自定义：

![主页装饰](http://pic.l2h.site/l2hsiteXiongzhanghao-decorate-2.png)“熊掌号后台-->主页管理-->自定义菜单”可以根据实际需求增加首页菜单，菜单可以定义到二级：

![主页装饰](http://pic.l2h.site/l2hsiteXiongzhanghao-decorate-3.png)

### 提升熊掌号指数

#### 绑定网站

“熊掌号后台-->内容源设置-->绑定设置”来绑定您的网站

#### 提升指数

这里到“熊掌号后台-->熊掌号成长-->熊掌号指数”里可以看到您的熊掌号指数成长。点击“去优化指数”，可以根据后台提示来提升指数。一些提升指数的方法：

*   *   绑定微信公众号、微博、今日头条等

*   *   装饰主页

*   *   用百度APP发表动态（类似发微信朋友圈）

*   *   开放平台接口调用（用API等方式提交资源）

*   **最直接方法**：提交资源并成功收录

![优化指数](http://pic.l2h.site/l2hsiteXiongZhangHao-1.png)![优化指数](http://pic.l2h.site/l2hsiteXiongZhangHao-2.png)![优化指数](http://pic.l2h.site/l2hsiteXiongZhangHao-3.png)

指数在100分以下时，许多后台操作是无法进行的。而超过100分后，后台操作权限相应就会放开。

Wordpress熊掌号H5页面改造
------------------

若要Wordpress页面能够被正常收录，需要改造您的Wordpress文章页面。Wordpress后台-->外观-->编辑-->修改主题的header.php。在<header>标签前增加如下熊掌号ID声明：

> <script src="https://xiongzhang.baidu.com/sdk/c.js?appid=xxxxxx"></script>

同时增加如下页面改造代码：

<?php if( is\_single() || is\_page() ): ?>
<script type="application/ld+json">
    {
        "@context": "https://ziyuan.baidu.com/contexts/cambrian.jsonld",
        "@id": "<?php the_permalink(); ?>",
        "appid": "您的熊掌号APP ID",
        "title": "<?php the_title(); ?>",
        "images": \[
            "<?php echo zillah\_catch\_that_image(); ?>"
            \],
        "description": "<?php if ($post->post_excerpt) 
        {$printDescription = $post->post_excerpt;} 
              else{
      $printDescription = preg\_replace('/\\s+/','',mb\_strimwidth(strip\_tags($post->post\_content),0,145,''));
                }
            echo $printDescription;?>",
        "pubDate": "<?php echo get\_the\_time('Y-m-d\\TG:i:s'); ?>"
    }
</script>	
<?php endif; ?>

提交页面
----

### 历史数据提交

到[Curl官网](https://curl.haxx.se/windows/ "https://curl.haxx.se/windows/")下载Curl For Windows并解压（注意根据您的Windows版本下载）。

准备好urls.txt，里边为您希望提交的历史数据URL，每行一个网址。到解压的Curl的bin目录执行如下命令：

> curl -H 'Content-Type:text/plain' --data-binary @urls.txt "[http://data.zz.baidu.com/urls?appid=xxxxxx&token=xxxxxxxx](http://data.zz.baidu.com/urls?appid%3Dxxxxxx%26token%3Dxxxxxxxx "http://data.zz.baidu.com/urls?appid%3Dxxxxxx%26token%3Dxxxxxxxx")"

其中appid和token需要更换成您自己熊掌号的id和token。以上命令也可以在"熊掌号后台-->资源提交-->API提交-->历史内容接口-->推送示例"中找到。 历史数据提交上限为50000，对个人站长绰绰有余了。

### 新增数据提交

新增数据也可用Curl进行提交，提交方法与历史数据提交类似，差别只是在于调用时的参数增加 _**type=realtime。**_

> curl -H 'Content-Type:text/plain' --data-binary @urls.txt "[http://data.zz.baidu.com/urls?appid=zxxxxxxx&token=xxxxxxxxxxx&type=realtime](http://data.zz.baidu.com/urls?appid%3Dzxxxxxxx%26token%3Dxxxxxxxxxxx%26type%3Drealtime "http://data.zz.baidu.com/urls?appid%3Dzxxxxxxx%26token%3Dxxxxxxxxxxx%26type%3Drealtime")"

每天提交上限为10条

### 增加Wordpress插件做数据提交

总是使用Curl提交难免复杂，要是Wordpress后台添加文章的时候自动提交则比较简单。

*   *   Wordpress后台-->插件-->搜索"BaiduXZH Submit"，添加并安装

*   *   转到插件设置页面

*   添加熊掌号的APP ID及Token（可以在百度熊掌号后台查看）并保存

![熊掌号插件配置](http://pic.l2h.site/l2hsiteXiongzhanghao-Plugin-0.png)

### 网站自动源同步

熊掌号支持网站自动抓取，参照"绑定网站"一节绑定网站并添加同步源。添加过后，一般一天之内即可抓取完网站内容。之后百度也会定期自动抓取。

总结
--

以上为Wordpress绑定熊掌号的一些基础设置，更多内容待继续摸索和发掘。