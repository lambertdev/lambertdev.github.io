---
title: 在线进制转换
url: 1004.html
id: 1004
comments: false
date: 2018-10-21 15:01:26
---

**在线进制转换器**

2进制4进制8进制10进制16进制32进制 2进制 3进制 4进制 5进制 6进制 7进制 8进制 9进制 10进制 11进制 12进制 13进制 14进制 15进制 16进制 17进制 18进制 19进制 20进制 21进制 22进制 23进制 24进制 25进制 26进制 27进制 28进制 29进制 30进制 31进制 32进制 33进制 34进制 35进制 36进制   
  

转换数字 

  
  

2进制4进制8进制10进制16进制32进制 2进制 3进制 4进制 5进制 6进制 7进制 8进制 9进制 10进制 11进制 12进制 13进制 14进制 15进制 16进制 17进制 18进制 19进制 20进制 21进制 22进制 23进制 24进制 25进制 26进制 27进制 28进制 29进制 30进制 31进制 32进制 33进制 34进制 35进制 36进制   
  

转换结果 

概述

在线进制转换器提供了二进制，八进制，十进制，十六进制等相互转换功能。如：

*   二进制转换十进制
*   二进制转八进制
*   二进制转十六进制
*   八进制转十进制
*   八进制转换成十六进制
*   八进制转二进制
*   十进制转二进制
*   十进制转八进制
*   十进制转十六进制
*   十六进制转二进制
*   十六进制转八进制
*   十六进制转十进制
*   ……

$(document).ready(function(){$('\[name="input_"\]').click(function(){$('#input\_num').val($(this).val());$('#input\_value').val("");$('#output\_value').val("")});$('\[name="output\_"\]').click(function(){$('#output\_num').val($(this).val());px(1)});$("#input\_num").change(function(){$("#input\_area input").removeAttr("checked");var val=$(this).val();$("#input\_area input\[value="+val+"\]").attr("checked","checked");$('#input\_value').val("");$('#output\_value').val("")});$("#output\_num").change(function(){$("#output\_area input").removeAttr("checked");var val=$(this).val();$("#output\_area input\[value="+val+"\]").attr("checked","checked");px(1)})});function pxparseFloat(x,y){x=x.toString();var num=x;var data=num.split(".");var you=data\[1\].split("");var sum=0;for(var i=0;i<data\[1\].length;i++){sum+=you\[i\]\*Math.pow(y,-1\*(i+1))}return parseInt(data\[0\],y)+sum}function zhengze(x){var str;x=parseInt(x);if(x<=10){str=new RegExp("^\[+\\\-\]?\[0-"+(x-1)+"\]*\[.\]?\[0-"+(x-1)+"\]*$","gi")}else{var letter="";switch(x){case 11:letter="a";break;case 12:letter="b";break;case 13:letter="c";break;case 14:letter="d";break;case 15:letter="e";break;case 16:letter="f";break;case 17:letter="g";break;case 18:letter="h";break;case 19:letter="i";break;case 20:letter="j";break;case 21:letter="k";break;case 22:letter="l";break;case 23:letter="m";break;case 24:letter="n";break;case 25:letter="o";break;case 26:letter="p";break;case 27:letter="q";break;case 28:letter="r";break;case 29:letter="s";break;case 30:letter="t";break;case 31:letter="u";break;case 32:letter="v";break;case 33:letter="w";break;case 34:letter="x";break;case 35:letter="y";break;case 36:letter="z";break}str=new RegExp("^\[+\\\-\]?\[0-9a-"+letter+"\]*\[.\]?\[0-9a-"+letter+"\]*$","gi")}return str}var n=50;var shurukuang="";var flag="";function px(y){if($("#input\_value").val()!=flag||y){flag=$("#input\_value").val();if($("#input\_num").selectedIndex<n){$("#input\_value").val("");$("#output\_value").val("")}else{var px00=$("#input\_value").val();var px0=px00.match(zhengze($("#input\_num").val()));if(px0){if(px0\[0\].indexOf(".")==-1){var px1=parseInt(px0,$('#input\_num').val())}else{var px1=pxparseFloat(px0,$('#input\_num').val())}px1=px1.toString($('#output\_num').val());$("#output\_value").val(px1);shurukuang=px00}else{$("#input\_value").val(shurukuang)}}n=$("#input\_num").selectedIndex}if($("#input\_value").val()==""){$("#output\_value").val("")}}