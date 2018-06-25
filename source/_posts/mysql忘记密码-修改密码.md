---
title: mysql忘记密码/修改密码
date: 2018-05-29 15:29:54
tags: mysql
---

1.  关闭mysql

   > service mysql stop

2. 无密码登录msyql

   > mysqld_safe --skip-grant-tables &  

3. root连接mysql

   > mysql -u root

4. 修改密码

   > use mysql
   >
   > update user set password=PASSWORD('admin') where User='root';  

5. 刷新权限

   > flush privileges;

6. 重启mysql

   > service msyql restart

