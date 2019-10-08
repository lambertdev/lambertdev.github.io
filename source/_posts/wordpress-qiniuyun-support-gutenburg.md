---
title: 'Wordpress: 简单修改七牛云图床插件支持Gutenburg(古腾堡)'
tags:
  - Gutenberg
  - wordpress
  - 七牛云
  - 古腾堡
url: 1458.html
id: 1458
categories:
  - 建站
date: 2018-12-09 13:25:37
---

前言
--

博客空间有限，一直使用[七牛云图床](https://www.qiniu.com/)和[开心洋葱的图床插件](http://www.75271.com/2954.html)做图片上传。之前按照《[升級到WORDPRESS 4.9.8](https://www.l2h.site/2018/11/25/%E5%8D%87%E7%B4%9A%E5%88%B0wordpress-4-9-8/)》介绍更新古腾堡编辑器后，发现插件失效。本文介绍简单修改，让Gutenberg后台仍可使用插件做图片上传。

> 注：本文修改简单，并非实现Gutenberg Block，若您想了解如何修改，可以移步[How to build a ](https://wisdomplugin.com/build-gutenberg-block-plugin/)**[Gutenberg block](https://wisdomplugin.com/build-gutenberg-block-plugin/)**[ plugin](https://wisdomplugin.com/build-gutenberg-block-plugin/) 学习了解。

呈现效果
----

![](http://pic.l2h.site/l2hsiteqiniuyun-gutenberg.png)

修改方法
----

1.修改qiniu-cloudtuchuang.php代码如下,注意代码加亮部分:
```PHP
//上传窗口
add_action('do_meta_boxes', 'qiniu_cloudtuchuang_script');
function qiniu_cloudtuchuang_script(){
    wp_enqueue_script( 'qiniu-jquery', plugins_url('js/qiniujq.min.js', __FILE__));
    //wp_enqueue_script( 'plupload-all',plugins_url('js/plupload/js/plupload.full.min.js', __FILE__) );
	wp_enqueue_script( 'plupload-all');
    wp_enqueue_script( 'qiniu', plugins_url('js/qiniu.js', __FILE__));
    wp_enqueue_script( 'qiniu-main', plugins_url('js/main.js', __FILE__ ),array( 'jquery' ));
    wp_enqueue_script( 'qiniu-ui', plugins_url('js/ui.js', __FILE__));
}    
 
add_action('do_meta_boxes', 'qiniu_cloudtuchuang_post_box');
function qiniu_cloudtuchuang_post_box(){
    $options = get_option('qiniu_options');
    if($options['accesskey'] && $options['secretkey']) add_meta_box('qiniu_cloudtuchuang_div', __('七牛云图床'), 'qiniu_cloudtuchuang_post_html', 'post', 'side');
}

add_action('do_meta_boxes', 'qiniu_cloudtuchuang_style');

function qiniu_cloudtuchuang_style(){
	wp_enqueue_style('qiniu_cloudtuchuang_style', plugins_url('css/qiniu_cloudtuchuang.css', __FILE__));
}

function qiniu_cloudtuchuang_post_html(){
	$options = get_option('qiniu_options');
    $host = $options['host'];
    $bucket = $options['bucket'];
    $prefix = $options['prefix'];
    $accesskey = $options['accesskey'];
    $secretkey = $options['secretkey'];
    $imgurl = $options['imgurl'];
    echo "<script>";
    if (!empty($prefix)) {
        echo 'var savekey = true; var prefix = \\''.$prefix.'\\';';
    }else echo 'var savekey = false;var prefix = \\'\\';';
    if (!empty($imgurl)) {
        echo 'var imgurl = true;';
    }else echo 'var imgurl = false;';
    echo 'var host =\\''.  $host . '\\';';
	//$nonce = wp_create_nonce('cloudtuchuang');
    //echo 'var uptokenurl=\\''. plugins_url('token.php', __FILE__) . '?_ajax_nonce='. $nonce .'&secretKey='. $secretkey . '&accessKey=' . $accesskey . '&bucket=' . $bucket . '&prefix=' . $prefix .'\\'</script>';
	echo 'var uptokenurl=\\''. plugins_url('token.php', __FILE__) . '?secretKey='. $secretkey . '&accessKey=' . $accesskey . '&bucket=' . $bucket . '&prefix=' . $prefix .'\\'</script>';
    //echo '<div id="qiniu_cloudtuchuang_post"><div id="pickfiles" href="#" ><span id="spantxt">拖拽上传图片</span></div></div>';
    echo '<div id="qiniu_cloudtuchuang_post_btn"><a id="qiniu-insert-media-button" class="button insert-qiniuyun " title="添加图片" data-editor="content" href="javascript:;">^_^ <span id="spandesc">添加图片</span></a></div>';
     echo 'File(Image) URL:<input type="text" id="qiniu_image_addr" readonly />';;
}
```
2. 修改js/main.js，找到“console.log(img);”，增加如下第二行代码：

console.log(img);
$('#qiniu_image_addr').val(qiniuurl);

3. 执行以上修改后保存退出即完成。

总结
--

以上修改相对还是比较简单，Enjoy it！