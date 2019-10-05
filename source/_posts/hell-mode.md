---
title: 为你的博客开出地狱模式
tags:
  - Javascript
url: 959.html
id: 959
categories:
  - Wordpress建站
date: 2018-10-20 13:54:33
---

双击会有惊喜哦！
========

引用自[麦葱的博客 ](https://maicong.me/t/250) ， 对源码做了一点修改提升手机平板上的点击体验。目前本站只有本篇博文有此效果 ![为你的博客开出地狱模式](http://pic.l2h.site/l2hsite屏幕快照 2018-10-20 下午8.53.10.png "为你的博客开出地狱模式") (function () {<br /> var \_\_timeout;<br /> var \_\_lastTap = 0;<br /> var \_\_tapNum = 0;<br /> var hell = function hell() {<br /> var hell = document.getElementById('hell');<br /> if (hell) {<br /> hell.parentNode.removeChild(hell);<br /> } else {<br /> var style = document.createElement('style');<br /> style.id = 'hell';<br /> style.textContent = 'body{ background:black; -webkit-transform: scale(-1,1); -ms-transform: scale(-1,1); transform: scale(-1,1); -webkit-filter: sepia(0) saturate(0) invert(1) brightness(1) contrast(1); filter: sepia(0) saturate(0) invert(1) brightness(1) contrast(1); }';<br /> document.head.appendChild(style);<br /> }<br /> };<br /> document.addEventListener('dblclick', hell, false);<br /> document.addEventListener('touchend', function () {<br /> var currentTime = new Date().getTime();<br /> var tapInt = currentTime - \_\_lastTap;<br /> ++\_\_tapNum;<br /> clearTimeout(\_\_timeout);<br /> if (tapInt < 500 ) { if (tapInt > 0) {<br /> if(0 == \_\_tapNum%2) {<br /> hell();<br /> }<br /> }<br /> }<br /> \_\_lastTap = currentTime;<br /> });<br /> })(); 源码如下（插入到页面任意位置即可使用）：

(<script>function () {
 var __timeout;
 var __lastTap = 0;
 var __tapNum = 0;
 var hell = function hell() {
 var hell = document.getElementById('hell');
 if (hell) {
 hell.parentNode.removeChild(hell);
 } else {
 var style = document.createElement('style');
 style.id = 'hell';
 style.textContent = 'body{ background:black; -webkit-transform: scale(-1,1); -ms-transform: scale(-1,1); transform: scale(-1,1); -webkit-filter: sepia(0) saturate(0) invert(1) brightness(1) contrast(1); filter: sepia(0) saturate(0) invert(1) brightness(1) contrast(1); }';
 document.head.appendChild(style);
 }
 };
 document.addEventListener('dblclick', hell, false);
 document.addEventListener('touchend', function () {
 var currentTime = new Date().getTime();
 var tapInt = currentTime - __lastTap;
 ++__tapNum;
 clearTimeout(__timeout);
 if (tapInt < 500 ) {
 if (tapInt > 0) {
 if(0 == __tapNum%2) {
 hell();
 }
 }
 }
 __lastTap = currentTime;
 });
})();</script>