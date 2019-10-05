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
  - Wordpress建站
date: 2018-12-09 13:25:37
---

前言
--

博客空间有限，一直使用[七牛云图床](https://www.qiniu.com/)和[开心洋葱的图床插件](http://www.75271.com/2954.html)做图片上传。之前按照《[升級到WORDPRESS 4.9.8](https://l2h.site/2018/11/25/%E5%8D%87%E7%B4%9A%E5%88%B0wordpress-4-9-8/)》介绍更新古腾堡编辑器后，发现插件失效。本文介绍简单修改，让Gutenberg后台仍可使用插件做图片上传。

> 注：本文修改简单，并非实现Gutenberg Block，若您想了解如何修改，可以移步[How to build a ](https://wisdomplugin.com/build-gutenberg-block-plugin/)**[Gutenberg block](https://wisdomplugin.com/build-gutenberg-block-plugin/)**[ plugin](https://wisdomplugin.com/build-gutenberg-block-plugin/) 学习了解。

呈现效果
----

![](http://pic.l2h.site/l2hsiteqiniuyun-gutenberg.png)

修改方法
----

1.修改qiniu-cloudtuchuang.php代码如下,注意代码加亮部分:

//上传窗口
add\_action('do\_meta\_boxes', 'qiniu\_cloudtuchuang_script');
function qiniu\_cloudtuchuang\_script(){
    wp\_enqueue\_script( 'qiniu-jquery', plugins\_url('js/qiniujq.min.js', \_\_FILE__));
    //wp\_enqueue\_script( 'plupload-all',plugins\_url('js/plupload/js/plupload.full.min.js', \_\_FILE__) );
	wp\_enqueue\_script( 'plupload-all');
    wp\_enqueue\_script( 'qiniu', plugins\_url('js/qiniu.js', \_\_FILE__));
    wp\_enqueue\_script( 'qiniu-main', plugins\_url('js/main.js', \_\_FILE__ ),array( 'jquery' ));
    wp\_enqueue\_script( 'qiniu-ui', plugins\_url('js/ui.js', \_\_FILE__));
}    
 
add\_action('do\_meta\_boxes', 'qiniu\_cloudtuchuang\_post\_box');
function qiniu\_cloudtuchuang\_post_box(){
    $options = get\_option('qiniu\_options');
    if($options\['accesskey'\] && $options\['secretkey'\]) add\_meta\_box('qiniu\_cloudtuchuang\_div', __('七牛云图床'), 'qiniu\_cloudtuchuang\_post_html', 'post', 'side');
}

add\_action('do\_meta\_boxes', 'qiniu\_cloudtuchuang_style');

function qiniu\_cloudtuchuang\_style(){
	wp\_enqueue\_style('qiniu\_cloudtuchuang\_style', plugins\_url('css/qiniu\_cloudtuchuang.css', \_\_FILE\_\_));
}

function qiniu\_cloudtuchuang\_post_html(){
	$options = get\_option('qiniu\_options');
    $host = $options\['host'\];
    $bucket = $options\['bucket'\];
    $prefix = $options\['prefix'\];
    $accesskey = $options\['accesskey'\];
    $secretkey = $options\['secretkey'\];
    $imgurl = $options\['imgurl'\];
    echo "<script>";
    if (!empty($prefix)) {
        echo 'var savekey = true; var prefix = \\''.$prefix.'\\';';
    }else echo 'var savekey = false;var prefix = \\'\\';';
    if (!empty($imgurl)) {
        echo 'var imgurl = true;';
    }else echo 'var imgurl = false;';
    echo 'var host =\\''.  $host . '\\';';
	//$nonce = wp\_create\_nonce('cloudtuchuang');
    //echo 'var uptokenurl=\\''. plugins\_url('token.php', \_\_FILE__) . '?\_ajax\_nonce='. $nonce .'&secretKey='. $secretkey . '&accessKey=' . $accesskey . '&bucket=' . $bucket . '&prefix=' . $prefix .'\\'</script>';
	echo 'var uptokenurl=\\''. plugins\_url('token.php', \_\_FILE__) . '?secretKey='. $secretkey . '&accessKey=' . $accesskey . '&bucket=' . $bucket . '&prefix=' . $prefix .'\\'</script>';
    //echo '<div id="qiniu\_cloudtuchuang\_post"><div id="pickfiles" href="#" ><span id="spantxt">拖拽上传图片</span></div></div>';
    echo '<div id="qiniu\_cloudtuchuang\_post\_btn"><a id="qiniu-insert-media-button" class="button insert-qiniuyun " title="添加图片" data-editor="content" href="javascript:;">^\_^ <span id="spandesc">添加图片</span></a></div>';
     echo 'File(Image) URL:<input type="text" id="qiniu\_image\_addr" readonly />';;
}

2\. 修改js/main.js，找到“console.log(img);”，增加如下第二行代码：

console.log(img);
$('#qiniu\_image\_addr').val(qiniuurl);

3\. 执行以上修改后保存退出即完成。

总结
--

以上修改相对还是比较简单，Enjoy it！