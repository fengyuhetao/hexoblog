---
title: .htaccess的配置
date: 2018-06-24 10:58:29
tags:
---

1. 解析其他后缀的文件

   > AddType application/x-httpd-php .jpg 

   该配置会解析.jpg后缀的文件

2. php_flag

   >php_flag engine off

   关闭php解析，当前目录下所有php文件不在解析

   > php_flag engine 1

   开启php解析

3. Options

   > Options All -Indexes

   不允许列目录