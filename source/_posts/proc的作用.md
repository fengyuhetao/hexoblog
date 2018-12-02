---
title: linux常见文件的作用
abbrlink: 9490
date: 2018-06-24 10:55:47
tags:
---

# linux常见文件的作用

1. /proc/self/environ              读取环境变量

   该文件会保存http请求头，所以发送恶意User-aget,可以通过文件包含，包含该文件getshell

2. /proc/net/arp                     读取arp

   判断当前局域网存活的主机

3. /proc/self[pid]/cmdline              获得当前运行程序命令行参数

4. /proc/net/dev                               网卡信息

5. /sys/class/net/eth0/address      mac地址

6. /etc/apache2/apache2.conf        apache2配置文件

7. /proc/mounts 文件系统列表

8.  /proc/cpuinfo CPU信息

9. /proc/meminfo 内存信息

10. /proc/[pid]/mountinfo 文件系统挂载的信息（可以看到docker文件映射的一些信息，如果是运行在容器内的进程，通常能找到重要数据的路径：如配置文件、代码、数据文件等）

11.  /proc/[pid]/fd/[fd] 进程打开的文件（fd是文件描述符id）

12. /proc/[pid]/exe 指向该进程的可执行文件

