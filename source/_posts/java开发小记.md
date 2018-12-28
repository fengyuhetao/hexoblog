---
title: java开发小记
date: 2018-12-27 21:13:11
tags:
---

# 登录保护

## 两次MD5

1. 用户端

   用户输入pass = md5(明文+固定salt)

2. 服务端

   pass = md5(用户输入pass+随机salt)，随机salt存入数据库

