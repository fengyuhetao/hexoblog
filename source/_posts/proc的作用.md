---
title: proc的作用
abbrlink: 9490
date: 2018-06-24 10:55:47
tags:
---

# proc目录的作用

1. /proc/self/environ              读取环境变量

   该文件会保存http请求头，所以发送恶意User-aget,可以通过文件包含，包含该文件getshell

2. /proc/net/arp                     读取arp

   判断当前局域网存活的主机

3. /proc/self[pid]/cmdline              获得当前运行程序命令行参数

