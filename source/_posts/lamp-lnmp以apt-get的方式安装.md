---
title: lamp && lnmp以apt-get的方式安装
tags: 运维
abbrlink: 18428
date: 2018-09-05 20:12:31
---

# lnmp

* 安装nginx

  > sudo apt-get install nginx

* 安装php

  > sudo apt-get install php

* 安装php-fpm

  > sudo apt-get install php-fpm

* 配置 `/etc/nginx/`

  ```shell
  # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
  #
  location ~ \.php$ {
     include snippets/fastcgi-php.conf;
  
     # With php7.0-cgi alone:
     # fastcgi_pass 127.0.0.1:9000;
     # With php7.0-fpm:
     fastcgi_pass unix:/run/php/php7.0-fpm.sock;
  }
  ```

* 重启nginx

* 操作命令:

  > service nginx [start|restart|stop]
  >
  > service php-fpm [start|restart|stop]

# lamp

* 安装apapche

  > sudo apt-get install apache

* 安装php

  > sudo apt-get install php

* 安装libapache2-mod-php7.0

  > sudo apt-get install libapache2-mod-php7.0

# 卸载

## 卸载nginx

```
sudo apt-get remove nginx-*
sudo apt-get purge nginx-*
# 删除旧文件目录
sudo find /etc -name "*nginx*" |xargs  rm -rf
```

## 卸载php

```
sudo apt-get --purge remove php7.0*
(或者 sudo apt-get --purge remove php5* libapache2-mod-php5)
sudo apt-get autoremove php7.0*(php5)
```

## 卸载mysql

```
sudo apt-get --purge remove mysql*
sudo apt-get autoremove mysql*

## 最后清理残留文件：
dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
```