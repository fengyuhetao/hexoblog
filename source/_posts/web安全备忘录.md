---
title: web安全备忘录
abbrlink: 16486
date: 2018-06-30 19:29:10
tags: web安全
---

## dumpfile和outfile的区别

SELECT into outfile: 导出到一个txt文件，可以导出每行记录的，这个很适合导库

SELECT into dump: 只能导出一行数据

如果想把一个可执行二进制文件用into outfile函数导出，导出后，文件会被破坏

因为into outfile函数会在行末端写新行，更致使的是会转义换行符，这样2进制可执行文件就会被破坏

这时，我们能用into dumpfile导出一个完整能执行的2进制文件，它不对任何列或行进行终止，也不执行任何转义处理

总结：

into outfile:导出内容

into dumpfile:导出二进制文件

## .htaccess

```
AddType application/x-httpd-php .png
php_flag engine 1
```

php_flag 打开或者关闭PHP解析，本指令仅在使用PHP的Apache模块版本时才有用，可以基于目录或者虚拟主机来打开或者关闭PHP，将engine off放到httpd.conf文件中适当的位置就可以激活或者禁用PHP。

