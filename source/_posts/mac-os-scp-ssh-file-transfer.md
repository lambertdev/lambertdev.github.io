---
title: MAC OS使用SCP命令进行SSH文件传输
tags:
  - MAC OS
  - SSH
  - 文件传输
url: 1940.html
id: 1940
categories:
  - Wordpress建站
date: 2019-02-09 11:36:38
---

前言
--

建站，进行文件传输是必不可少的。而本人又不希望给服务器开太多的服务端口，因此是拒绝FTP的。而一直知道SSH服务器支持文件传输，为何不复用该端口的服务进行呢？经过搜索，发现MAC OS下可以用SCP命令和服务器进行文件双向传输。

![](https://l2h.site/wp-content/uploads/2019/02/scp_command-1024x353.png)

使用方法
----

### 上传文件到服务器

> **_scp_** /path/filename username@servername:/path/  

*   **/path/filename**: 本地文件目录
*   **username**: 在远端服务器的用户名
*   **servername**：服务器域名或者ip地址
*   **/path/**：远端服务器的目录（即要上传到的位置）

回车后，会要求您输入远端用户的密码，输入密码后便开始上传

### 上传目录到服务器

> **_scp_** _-r_ local\_dir username@servername:remote\_dir  

与上传文件的差异在于_**-r**_参数

### 从服务器下载文件

> **_scp_** username@servername:/path/filename local_dir  

*   **local_dir**：本地目录
*   **/path/filename**：远端文件

### 从服务器下载目录

> **_scp_** -r username@servername:remote\_dir local\_dir  

同样，与下载普通文件的差异只在于_**-r**_参数

结语
--

本节简单介绍了MAC下使用SSH（SCP）进行最常用的上传、下载的方法。SCP的具体参数以及更多用法可以在shell下使用 man scp命令查看。希望对您有所帮助