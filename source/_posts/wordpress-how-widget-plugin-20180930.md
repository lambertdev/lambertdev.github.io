---
title: Wordpress-写插件增加小工具(Widget)的方法
tags:
  - wordpress
url: 758.html
id: 758
categories:
  - 建站
date: 2018-09-30 23:53:29
---

写这篇文章的目的，是因为在使用**WP-MulticolLinks**插件时，发现只能增加一个小工具(Widget)到工具栏。如下图，这样没有办法按照类别增加多列友链小工具。 ![Wordpress-写插件增加小工具(Widget)的方法](http://pic.l2h.site/l2hsite1-1.png "Wordpress-写插件增加小工具(Widget)的方法") 于是对该插件做了修改，可以支持在后台拖动多个多列友情链接小工具。 ![Wordpress-写插件增加小工具(Widget)的方法](http://pic.l2h.site/l2hsite2.png "Wordpress-写插件增加小工具(Widget)的方法") 以下是在插件中增加代码实现多列的方法：

*   注册我们的小工具，在widget界面初始化时可以在后台界面中增加我们的小工具
```PHP
// 以下为在widgets界面初始化时，注册我们自己定义的小工具

function register_ex_widget() {
    register_widget( 'EX_Widget' );
}
add_action( 'widgets_init', 'register_ex_widget' );

*   定义小工具的类，继承自WP_Widget，这个基类是可以创建多个实例的

// 自定小工具类，继承WP_Widget
class EX_Widget extends WP_Widget {
    // 初始化设定，定义小工具的名字等基本信息
    function EX_Widget() {
        $widget_ops = array( 'classname' => 'side_ex', 'description' => '设定多列友情链接');
        $control_ops = array( 'width' => 300, 'height' => 500, 'id_base' => 'side_ex-widget' );
        $this->WP_Widget( 'side_ex-widget', '多列友情链接', $widget_ops, $control_ops );
    }
    // 小工具在工具栏显示的函数，之后介绍具体实现
    function widget( $args, $instance ) {
  
    }
    // 更新Widget时执行该函数
    function update( $new_instance, $old_instance ) {
        return $instance;
    }
    // Widget菜单的显示时执行该函数
    function form( $instance ) {
    }
}
```
*   widget函数实现
```PHP
function widget( $args, $instance ) {
   // 以上均为转化参数用于后续传递
   $this_title = empty($instance['title']) ? __('Links', 'wp-multicollinks') : $instance['title'];
   $orderbyParam = 'name';
   if ($instance['orderby'] == 2) {
     $orderbyParam = 'url';
   } else if ($instance['orderby'] == 3) {
     $orderbyParam = 'rating';
   } else if ($instance['orderby'] == 4) {
     $orderbyParam = 'rand';
   }
   $orderParam = 'ASC';
   if ($instance['order'] == 2) {
     $orderParam = 'DESC';
   }
   // 组合参数字符串，用于wp_multicollinks函数执行时使用。wp_multicollinks为显示友情链接的函数，原插件就有定义
   $argsBinding = 	  'limit='	    . $instance['number'] 
           . '&columns='	. $instance['columns'] 
           . '&category='	. $instance['category'] 
           . '&orderby='	. $orderbyParam 
           . '&order='		. $orderParam 
           . '&navigator='	. ($instance['navigator'] ? 'true' : 'false');
               extract( $args );// $args裡可以拿到所在栏位的资讯，如before_widget、after_widget..等
   echo $before_widget;
   echo "<h2 class=\\"widget-title\\">$this_title</h2>";
   echo '<ul>';
   wp_multicollinks($argsBinding);
   echo '</ul>';       
   echo $after_widget;	
   }
```
*   后台小工具显示及更新函数实现
```PHP
// 於後台更新Widget時會做的事
    function update( $new_instance, $old_instance ) {
        $instance = $old_instance;
        //用填写的值给新创建的Widget实例赋值
    $instance['title'] = strip_tags( $new_instance['title'] );
    $instance['number'] = strip_tags( $new_instance['number'] );
    $instance['columns'] = strip_tags( $new_instance['columns'] );
    $instance['category'] = strip_tags( $new_instance['category'] );
    $instance['orderby'] = strip_tags( $new_instance['orderby'] );
    $instance['order'] = strip_tags( $new_instance['order'] );
    $instance['navigator'] = strip_tags( $new_instance['navigator'] );
    
        return $instance;
    }
     
    // Widget在後台模組頁的外觀
    function form( $instance ) {
        // 可以用数组设定显示小工具表格的时的预设值
    $defaults = array('title'=>'友情链接');
                $instance = wp_parse_args( (array) $instance, $defaults ); ?>
  // 表项的显示，显示时应该要能加载出之前已经保存的内容	
    <p>
      <label  for="<?php echo $this->get_field_id( 'title' ); ?>">
        <?php _e('Title: ', 'wp-multicollinks'); ?>
        <input class="widefat" id="title" name="<?php echo $this->get_field_name( 'title' ); ?>" type="text" value="<?php echo $instance['title']; ?>" />
      </label>
    </p>
    <p>
      <label for="<?php echo $this->get_field_id( 'number' ); ?>">
          <?php _e('Number of links to show: ', 'wp-multicollinks'); ?>
          
      </label>
      <input style="width: 25px;" id="number" name="<?php echo $this->get_field_name( 'number' );?>" type="text" value="<?php echo $instance['number']; ?>" />
      <br />
      <small><?php _e('(0 for ∞)', 'wp-multicollinks'); ?></small>
    </p>
       //以下省略剩余表项代码
   }
```
最后，放上插件链接，[点击下载](http://pic.l2h.site/l2hsitel2h-multicollinks.7z) 后解压到wordpress的plugin目录，启用插件即可使用 P.S. 原插件由mg12(http://www.fighton.cn/)编写