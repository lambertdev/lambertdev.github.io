---
title: Wordpress新评论微信通知+邮件回复
tags:
  - wordpress
  - 评论通知
url: 1992.html
id: 1992
categories:
  - 建站
date: 2019-02-27 21:09:39
---

前言
--

建立博客，与路过的大佬们互动必不可少。除了优质内容，互动也是提高博客回头率的小手段。本文主要解决两个问题：

*   如何第一时间知道大佬们在博客留言呢？
*   对留言大佬回复，如何通知到对方？

新评论微信通知
-------

第一个问题主要是时效性问题。其实Wordpress本身就可以通知博主邮箱做留言审核，当然也有一些不错的插件（例如，Jetpack）。不过这还不够快（因为邮箱毕竟不是时时都看的）。因此，经过搜索，L&H Site使用的是“[Server酱](http://sc.ftqq.com/3.version)”。对，就是本站那个友情链接。

### 原理

盗用Server酱官网的一张图说明微信推送原理。

![](http://anime-img.stor.sinaapp.com/5bf55cc81c840.gif)

### 注册Server酱并关注

如上图，注册Server酱，您首先需要有一个[Github](https://github.com/)账号（不要告诉我您不知道什么是Github哦）。

使用Github登录Server酱，打开[页面](http://sc.ftqq.com/?c=code)，获取到SCKey。

![](http://pic.l2h.site/ServerChan2-512x1024.jpg)

打开[微信绑定页](http://sc.ftqq.com/?c=wechat&a=bind)，使用微信扫描个人专属二维码并关注，然后电脑点击【检查结果并确认绑定】便关联成功

![](http://pic.l2h.site/ServerChan1-512x1024.png)

关联成功后，可以在消息发送页尝试发送一条消息看微信是否收得到，正常收到消息的微信如下图：

![](http://pic.l2h.site/ServerChan3-512x1024.jpg)

### 编辑主题function.php

编辑主题function.php，加入如下代码：
```PHP
//评论微信推送  
function sc_send($comment_id)  
{  
$text = '博客上有一条新的评论';  
$comment = get_comment($comment_id);  
$desp = $comment->comment_content;  
$key = 'SCUxxxxxx';  //你的SCKey，从之前Server酱网站得到
$postdata = http_build_query(  
array(  
'text' => $text,  
'desp' => $desp  
)  
);  
   
$opts = array('http' =>  
array(  
'method' => 'POST',  
'header' => 'Content-type: application/x-www-form-urlencoded',  
'content' => $postdata  
)  
);  
$context = stream_context_create($opts);  
return $result = file_get_contents('http://sc.ftqq.com/'.$key.'.send', false, $context);  
}  
add_action('comment_post', 'sc_send', 19, 2);  
```
完成后保存。之后留言推送，效果如前一幅图。

评论回复邮件通知
--------

### 安装SMTP邮件插件

推荐安装Easy WP SMTP，官方插件页面搜索即可安装，当然也有很多同类型替代，大同小异。

### 插件配置

后台-->设置-->Easy WP SMTP-->SMTP Settings

按照您的电子邮件提供商的设定进行配置，注意您的网站所在主机需要开放所需连接SMTP端口的权限。推荐使用outlook.com邮件，163容易被当作垃圾邮件。

设置完保存即可。您也可以到 后台-->设置-->Easy WP SMTP--> Test Email测试配置是否正确

![](http://pic.l2h.site/Mail-Notify-2.png)

### 修改主题funtion.php

若要评论触发邮件通知，则需要修改function.php增加如下代码：
```PHP
function comment_mail_notify($comment_id) {
  $comment = get_comment($comment_id);
  $parent_id = $comment->comment_parent ? $comment->comment_parent : '';
  $spam_confirmed = $comment->comment_approved;
  if (($parent_id != '') && ($spam_confirmed != 'spam')) {
    $wp_email = 'no-reply@' . preg_replace('#^www.#', '', strtolower($_SERVER['SERVER_NAME'])); //e-mail 发出点, no-reply 可改为可用的 e-mail.
    $to = trim(get_comment($parent_id)->comment_author_email);
    $subject = '您在 [' . get_option("blogname") . '] 的留言有了回复';
    $message = '
    <div style="background-color:#eef2fa; border:1px solid #d8e3e8; color:#111; padding:0 15px; -moz-border-radius:5px; -webkit-border-radius:5px; -khtml-border-radius:5px;">
      <p>' . trim(get_comment($parent_id)->comment_author) . ', 您好!</p>
      <p>您曾在《' . get_the_title($comment->comment_post_ID) . '》的留言:<br />'
       . trim(get_comment($parent_id)->comment_content) . '</p>
      <p>' . trim($comment->comment_author) . ' 给您的回复:<br />'
       . trim($comment->comment_content) . '<br /></p>
      <p>您可以点击<a href="' . htmlspecialchars(get_comment_link($parent_id, array('type' => 'comment'))) . '">查看回应完整内容</a></p>
      <p>欢迎再度光临<a href="' . get_option('home') . '">' . get_option('blogname') . '</a></p>
      <p>(此邮件由系统自动发送，请勿回复.)</p>
    </div>';
      $from = "From: \\"" . get_option('blogname') . "\\" <$wp_email>";
      $headers = "$from\\nContent-Type: text/html; charset=" . get_option('blog_charset') . "\\n";
      wp_mail( $to, $subject, $message, $headers );
  }
}
add_action('comment_post', 'comment_mail_notify');
```
### 效果

配置成功后，试试效果吧。如下图：

![](http://pic.l2h.site/Mail-Notify.png)

结语
--

以上就是Wordpress新评论微信通知+邮件回复的设置方法了，是不是很简单呢？快试试吧。