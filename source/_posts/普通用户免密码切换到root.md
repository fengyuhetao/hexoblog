---
title: 普通用户免密码切换到root
date: 2018-05-29 14:44:48
tags:
---

1. sudo vim /etc/sudoers

2. 添加一行:

   > tiandiwuji  ALL=(ALL)       NOPASSWD: ALL

