---
title: 网站被黑了
url: 2115.html
id: 2115
categories:
  - Wordpress建站
date: 2019-03-21 21:39:03
tags:
---

事件
--

上班无暇照料网站，今天XH发微信说网站登录不上了。打开一看，居然被重定位到了一些不可描述页面：

![](https://l2h.site/wp-content/uploads/2019/03/webwxgetmsgimg-512x1024.png)

问题查找经过
------

首先分别做了以下检查：

* 登录阿里云后台查看域名解析，不存在被改写情况
* 登录到服务器查找主题页面被修改情况，也未见修改
* 修改数据库，禁用所有插件，并删除掉所有页面缓存，仍热有此问题

后来根据重定向到的页面 setforconfigplease.com，到万能的Google找到了问题原因

问题原因
----

简言之，网站的如下两个设定直接被改写成了黑客想要重定向的页面。

![](https://l2h.site/wp-content/uploads/2019/03/Image-2.png)

个人后台密码一直设置的非常严，且数据库也保护较好，为何会有设定被改的现象？Google说，最新的Easy WP SMTP插件有权限检查不严漏洞，导致普通用户可以构造攻击数据来改写WP_OPTION。
```PHP
301 $is_import_settings = filter_input( INPUT_POST, 'swpsmtp_import_settings', FILTER_SANITIZE_NUMBER_INT );
302 if ( $is_import_settings ) {
303 $err_msg = __( 'Error occurred during settings import', 'easy-wp-smtp' );
304 if ( empty( $_FILES[ 'swpsmtp_import_settings_file' ] ) ) {
305 echo $err_msg;
306 wp_die();
307 }
308 $in_raw = file_get_contents( $_FILES[ 'swpsmtp_import_settings_file' ][ 'tmp_name' ] );
309 try {
310    $in = unserialize( $in_raw );
311   if ( empty( $in[ 'data' ] ) ) {
312       echo $err_msg;
313       wp_die();
314  }
315   if ( empty( $in[ 'checksum' ] ) ) {
316     echo $err_msg;
317      wp_die();
318   }
319  if ( md5( $in[ 'data' ] ) !== $in[ 'checksum' ] ) {
320       echo $err_msg;
321     wp_die();
322  }
323  $data = unserialize( $in[ 'data' ] );
324  foreach ( $data as $key => $value ) {
325  update_option( $key, $value );
326  }
327  set_transient( 'easy_wp_smtp_settings_import_success', true, 60 * 60 );
328  $url = admin_url() . 'options-general.php?page=swpsmtp_settings';
329   wp_safe_redirect( $url );
330   exit;
```
黑客正是如此更改了我的网站URL，导致访问被重定向到了他们指定的网址。

不仅如此，网站设置中的“允许用户注册”选项也被如此改写，而且被改成了用户注册默认即是管理员。这我才想起前两天收到邮件推送一个怪怪的新用户注册的邮件，由于当时正在工作给忽略了。T_T

解决办法
----

1.  使用数据库命令先删除掉Easy WP SMTP插件（避免再次被人利用）
2.  使用数据库命令更改siteurl和home回自己的网址https://l2h.site,此时就可以正常登录网站了
    *   注意你的网站如果启用了cache，需要先到后台删除例如autoptimize和wp super cache保存的cache
    *   如果你的数据库开启了redis缓存，也要清空redis缓存
3.  到后台设置-->常规里，禁用掉新用户注册，以及更改新注册用户默认身份为普通身份

总结
--

以往对网站插件更新都非常勤快，觉得使用最新的就是最好。不过这次经历告诉自己，能不要更新就不要更新，除非有非用不可的功能。害得自己今晚很喜欢的健身课都没去上。以上。