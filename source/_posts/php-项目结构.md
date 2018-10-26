---
title: php-项目结构
tags: php	
abbrlink: 61088
date: 2018-09-26 19:18:25
password: php
---

# 整体结构

首先使用`tree -L 1 -d`来看一下所有目录:

```
qianfa@qianfa:~/Desktop/php_debug/php-7.0.12$ tree -L 1 -d
.
├── build
├── ext
├── include
├── libs
├── main
├── modules
├── netware
├── pear
├── sapi
├── scripts
├── tests
├── travis
├── TSRM
├── win32
└── Zend
```


| 目录    | 描述                         |
| ------- | ---------------------------- |
| TSRM    | 线程相关安全的实现           |
| Zend    | PHP解析器的核心实现          |
| build   | linux下编译相关的目录        |
| ext     | PHP的扩展                    |
| main    | PHP的主要代码                |
| netware | 网络目录，socket的定义与实现 |
| pear    | PHP扩展及应用的代码仓库      |
| sapi    | PHP的应用层接口              |
| scripts | Linux下的脚本目录            |
| tests   | 测试脚本目录                 |
| travis  | 用于构建，非PHP特有目录      |
| win32   | Windows下编译PHP的相关脚本   |
| modules | 仅有opcache.a和opcache.so    |
| libs    | 目录为空                     |
| include | 目录为空                     |

核心目录主要包括`sapi,main,zend,ext,TSRM`，也是需要重点关注目录。

php整体架构图如下:

![](/assets/php/2015111014002047.jpg)

# sapi(Server API)

我们知道，php执行的方法有多种，常见的比如cli模式，fpm/cgi模式，apache2handler模式，除此以外还有embed模式，litespeed模式，这两种没怎么用过。

```
qianfa@qianfa:~/Desktop/php_debug/php-7.0.12/sapi$ tree -L 1
.
├── apache2handler
├── cgi
├── cli
├── embed
├── fpm
├── litespeed
├── phpdbg
└── tests
```

SAPI则负责PHP对外提供服务的规范，它定义了结构体`sapi_module_struct`，该结构体定义了模式启动，关闭，激活，失效等多个钩子函数指针，每个模式将这些函数指针指向自己的函数，从而可以轻松扩展PHP对外服务的方式。

当需要调用服务器相关信息时，全部通过SAPI接口中对应方法调用实现，而这对应的方法在各个服务器抽象层实现时都会有各自的实现。此外，由于很多操作的通用性，有很大一部分的接口方法使用的是默认方法。

调用示意图如下:

![](/assets/php/TIM截图20180926201050.png)

# main

php的核心文件所在，主要实现PHP的基本设施，承接SAPI的请求，分析出要执行的脚本文件和参数，并对环境和配置进行初始化，比如初始化变量和常量、注册函数、解析配置文件、加载扩展等等。

参考示意图:

![](/assets/php/TIM截图20180926203443.png)

# Zend

Zend引擎的实现目录,主要负责PHP的语法实现、内存管理及脚本的编译运行环境，opcode的执行以及扩展机制的实现等，它由编译器、执行器两部分组成。Zend引擎类似于Java虚拟机，opcode则类似于java字节码。

编译器负责将PHP代码进行词法、语法分析，并生成抽象语法树，然后进一步编译为opcode，opcode是Zend虚拟机可识别的指令，php7一共有173个opcode，所有的语法都是由这些opcode组成的。执行器负责执行编译器输出的opcode。

![](/assets/php/TIM截图20180926205248.png)

# ext(extension)

官方扩展目录，它是扩展PHP内核功能的一种方式，分为PHP扩展与zend扩展，都支持用户自定义开发，这两种都比较常见，PHP扩展有gd、json、date、array等，而我们熟知的opcache就是Zend扩展。其中包括了绝大多数PHP的函数的定义和实现，如array系列，pdo系列，spl系列等函数的实现，都在这个目录中。个人写的扩展在测试时也可以放到这个目录，方便测试和调试。

![](/assets/php/TIM截图20180926205422.png)

# TSRM(hread Safe Resource Manager)

 线程安全资源管理器。PHP的线程安全是构建在TSRM库之上的，PHP实现中常见的*G宏通常是对TSRM的封装。

全局变量就是定义在函数外的变量，它属于公共资源，在多线程的环境下，访问公共资源就可能会引起冲突，TSRM就是为解决该问题而诞生的。它为每个线程分配一个独立的自增ID，该ID作为当前线程的全局变量内存区的索引，从而实现线程的完全独立。

其实PHP大部分SAPI都是单线程的，所以并不需要过多关注线程安全，但是在Apache或者用户自己实现的PHP环境下，就需要考虑线程安全问题了。

# 参考链接:

* https://blog.csdn.net/enoch612/article/details/82467895
* 《深入理解PHP内核》