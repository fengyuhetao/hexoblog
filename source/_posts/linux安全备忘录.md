---
title: linux安全备忘录
tags: ctf
abbrlink: 50844
date: 2018-06-04 11:23:22
---

## linux下执行命令绕过字符串过滤

1. 字符串拼接

   > 'l''s'
   >
   > 'ls''-a''l'

2. 文件名绕过

   > cat 'fl''ag'
   >
   > cat 'fl*'
   >
   > cat 'fl??'

## find命令执行

> find /tmp -iname sth -or -exec ls \;                会多次执行
>
> find /tmp -iname sth -or -exec ls \; -quit        只执行一次

## linux提权

### 利用通配符进行Linux本地提权

链接：https://blog.csdn.net/qq_27446553/article/details/80943097

案例: swpuctf 2018 有趣的邮件



