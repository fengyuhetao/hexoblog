---
title: linux命令执行
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