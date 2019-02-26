---
title: lamp && lnmp以apt-get的方式安装
tags: 运维
abbrlink: 18428
date: 2018-09-05 20:12:31
---

# ubuntu

## lnmp

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

## lamp

* 安装apapche

  > sudo apt-get install apache

* 安装php

  > sudo apt-get install php

* 安装libapache2-mod-php7.0

  > sudo apt-get install libapache2-mod-php7.0

## 卸载

### 卸载nginx

```
sudo apt-get remove nginx-*
sudo apt-get purge nginx-*
# 删除旧文件目录
sudo find /etc -name "*nginx*" |xargs  rm -rf
```

### 卸载php

```
sudo apt-get --purge remove php7.0*
(或者 sudo apt-get --purge remove php5* libapache2-mod-php5)
sudo apt-get autoremove php7.0*(php5)
```

### 卸载mysql

```
sudo apt-get --purge remove mysql*
sudo apt-get autoremove mysql*

## 最后清理残留文件：
dpkg -l |grep ^rc|awk '{print $2}' |sudo xargs dpkg -P
```

# Centos

迷你版centos安装后配置:

### 激活网卡

解决ifconfig不可用：ip addr 即查看分配网卡情况。 

激活网卡：在文件 /etc/sysconfig/network-scripts/ifcfg-***中

进入编辑模式，将 ONBOOT=no 改为 ONBOOT=yes，就OK

### 防火墙

关闭防火墙:

`systemctl stop firewalld`

临时打开防火墙:

`systemctl start firewalld`

防火墙开启不自启动:

`systemctl disable firewalld`

防火墙开启自启动:

`systemctl enable firewalld`

查看防火墙状态

`systemctl status firewalld`

临时关闭SELinux

`setenforce 0`

临时打开SELinux

`setenforce 1`

开机关闭SELinux

编辑/etc/selinux/config文件，将SELINUX的值设置为disabled。下次开机SELinux就不会启动了。

### 安装mariadb

下载mysql源安装包

`wget http://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm`

安装MySql源

`yum -y install mysql57-community-release-el7-11.noarch.rpm`

查看安装效果

`yum repolist enabled | grep mysql.*`

安装mysql:

`yum install mysql-community-server`

启动mysql:

`systemctl start  mysqld.service`

查看运行状态：

`systemctl status mysqld.service`

查看初始化数据库密码:

`grep "password" /var/log/mysqld.log`

修改密码:

`ALTER USER 'root'@'localhost' IDENTIFIED BY '****************';`

报错：

```
ERROR 1820 (HY000): You must reset your password using ALTER USER statement before executing this statement.
```

可能是密码太弱，要是想使用弱密码，可以降低密码策略。

`set global validate_password_policy=0;`

在修改密码即可。

设置可远程访问：

`grant all privileges on *.* to 'root'@'%' identified by '[密码]' with grant option;`