---
title: php命令执行函数
abbrlink: 45992
date: 2018-07-15 09:56:37
tags: web安全
---

代码或命令执行函数：

1. system                    允许执行一个外部程序并回显输出，类似于 passthru() 
2. shell_exec
3. exec                        允许执行一个外部程序 
4. curl_exec
5. popen               可通过 popen() 的参数传递一条命令，并对 popen() 所打开的文件进行执行。
6. proc_open               执行一个命令并打开文件指针用于读取以及写入 
7. passthru                   允许执行一个外部程序并回显输出 
8. pcntl_exec
9. popepassthru 
10. assert
11. eval
12. preg_replace              \e修饰符          PHP 5.5.0 起， 传入 “\e” 修饰符的时候，会产生一个 E_DEPRECATED 错误； PHP 7.0.0 起，会产生 E_WARNING 错误，同时 “\e” 也无法起效。 
13. create_functioin            创建一个匿名函数。  
14. call_user_func             把第一个参数作为回调函数调用  
15. call_user_func_array      调用回调函数，并把一个数组参数作为回调函数的参数  

其他:

1. scandir
2. glob                 列目录
3. phpinfo
4. chroot                   可改变当前 PHP 进程的工作根目录，仅当系统支持 CLI 模式 PHP 时才能工作，且该函数不适用于 Windows 系统。 
5. chgrp                   改变文件或目录所属的用户组 
6. chown                  改变文件或目录的所有者 
7. error_log                将错误信息发送到指定位置（文件)，在某些版本的 PHP 中，可使用 error_log() 绕过 PHP safe mode， 执行任意命令 
8. openlog
9. syslog                 可调用 UNIX 系统的系统层 syslog() 函数。
10. proc_get_status    获取使用 proc_open() 所打开进程的信息    
11. ini_alter                是 ini_set() 函数的一个别名函数，功能与 ini_set() 相同 
12. ini_set                  可用于修改、设置 PHP 环境配置参数 
13. ini_restore          可用于恢复 PHP 环境配置参数到其初始值。
14. dl()                       在 PHP 进行运行过程当中（而非启动时）加载一个 PHP 外部模块。
15. pfsockopen         建立一个 Internet 或 UNIX 域的 socket 持久连接。
16. fsocket
17. readlink           返回符号连接指向的目标文件内容
18. symlink            在 UNIX 系统中建立一个符号链接。
19. stream_socket_server             建立一个 Internet 或 UNIX 服务器连接
20. putenv                    用于在 PHP 运行时改变系统字符集环境。在低于 5.2.6 版本的 PHP 中，可利用该函数修改系统字符集环境后，利用 sendmail 指令发送特殊参数执行系统 SHELL 命令。