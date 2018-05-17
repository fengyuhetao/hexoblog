---
title: pwn
date: 2018-05-09 20:55:11
tags:
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

## checksec检测到的保护机制

1. cannary  (栈保护)

   栈溢出保护是一种缓冲区溢出攻击缓解手段，当函数存在缓冲区溢出攻击漏洞时，攻击者可以覆盖栈上的返回地址来让shellcode能够得到执行。当启用栈保护后，函数开始执行的时候会先往栈里插入cookie信息，当函数真正返回的时候会验证cookie信息是否合法，如果不合法就停止程序运行。攻击者在覆盖返回地址的时候往往也会将cookie信息给覆盖掉，导致栈保护检查失败而阻止shellcode的执行。在Linux中我们将cookie信息称为canary。 

   gcc选项: 

   * gcc4.2版本-fstack-protector和-fstack-protector-all编译参数开启栈保护功能，4.9新增了-fstack-protector-strong编译参数让保护的范围更广。 
   * -fno-stack-protector 不开启栈保护 

   > gcc -fno-stack-protector -o test test.c  //禁用栈保护 
   >
   > gcc -fstack-protector -o test test.c   //启用堆栈保护，不过只为局部变量中含有 char 数组的函数插入保护代码 
   >
   > gcc -fstack-protector-all -o test test.c //启用堆栈保护，为所有函数插入保护代码 

2. fortify

   它其实和栈保护都是gcc的新的为了增强保护的一种机制，防止缓冲区溢出攻击。由于并不是太常见，也没有太多的了解。

3. NX(windows下叫 DEP)

   NX（DEP）的基本原理是将数据所在内存页标识为不可执行，当程序溢出成功转入shellcode时，程序会尝试在数据页面上执行指令，此时CPU就会抛出异常，而不是去执行恶意指令。 

   gcc编译器默认开启了NX选项，如果需要关闭NX选项，可以给gcc编译器添加-z execstack参数。 

4. PIE(windows下ASLR)（address space layout randomization）

   一般情况下NX（Windows平台上称其为DEP）和地址空间分布随机化（ASLR）会同时工作。 

   > 0 - 表示关闭进程地址空间随机化。 
   >
   > 1 - 表示将mmap的基址，stack和vdso页面随机化。 
   >
   > 2 - 表示在1的基础上增加栈（heap）的随机化。 

   可以防范基于Ret2libc方式的针对DEP的攻击。ASLR和DEP配合使用，能有效阻止攻击者在堆栈上运行恶意代码。

   Built as PIE：位置独立的可执行区域（position-independent executables）。这样使得在利用缓冲溢出和移动操作系统中存在的其他内存崩溃缺陷时采用面向返回的编程（return-oriented programming）方法变得难得多。

   liunx下关闭PIE的命令如下：

   > sudo -s echo 0 > /proc/sys/kernel/randomize_va_space 

5. RELPO

   设置符号重定向表格为只读或在程序启动时就解析并绑定所有动态符号，从而减少对GOT（Global Offset Table）攻击。 

## gdb中print和x的区别

print 就是打印变量（参数是什么，就打印什么），x但因给定变量代表的内存地址里的值（x后边的参数是地址值，打印的是地址所在内存单元的值）。比如:

> p/x *$rax                                     ; \$1=0x41414141 
>
> x/x $rax                                       ;\$2=0x41

差不多，区别在于打印出来的字节多少而已。