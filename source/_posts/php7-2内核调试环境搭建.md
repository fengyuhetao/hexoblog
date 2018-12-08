---
title: php7.2内核调试环境搭建
tags: php
abbrlink: 36891
date: 2018-12-05 10:10:45
---

## 工具

* PHP7.2.12
* gdb
* pwndbg
* clion

## php安装

```
wget http://hk1.php.net/get/php-7.2.12.tar.gz/from/this/mirror
tar -zxvf php-7.2.12.tar.gz
# php安装基本步骤比较简单，关键是configure的配置选项
./configure --prefix=/usr/local/php7/php7.2 --enable-debug --enable-fpm
make && make install

cd /usr/bin
sudo ln -s /usr/local/php7/php7.2/bin/php php7
```

## gdb调试

```
sudo apt install gdb
git clone https://github.com/pwndbg/pwndbg
cd pwndbg
./setup.sh
```

创建php文件

```
cd ~/Desktop/php_debug/
vim test.php
```

调试test.php

```
qianfa@qianfa:~/Desktop/php_debug$ gdb -q php7
pwndbg: loaded 175 commands. Type pwndbg [filter] for a list.
pwndbg: created $rebase, $ida gdb functions (can be used with print/break)

Reading symbols from php7...done.
pwndbg> b main
Breakpoint 1 at 0xa0d041: file /home/qianfa/Desktop/php_debug/php-7.2.12/sapi/cli/php_cli.c, line 1202.
pwndbg> r test.php
Starting program: /usr/bin/php7 test.php

Breakpoint 1, main (argc=2, argv=0x7fffffffde78) at /home/qianfa/Desktop/php_debug/php-7.2.12/sapi/cli/php_cli.c:1202
1202	{
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
───────────────────────────────────[ REGISTERS ]───────────────────────────────────
 RAX  0xa0d029 (main) ◂— push   rbp
 RBX  0x0
 RCX  0xa0
 RDX  0x7fffffffde90 —▸ 0x7fffffffe238 ◂— 0x52455041505f434c ('LC_PAPER')
 RDI  0x2
 RSI  0x7fffffffde78 —▸ 0x7fffffffe221 ◂— '/usr/bin/php7'
 R8   0xa17e60 (__libc_csu_fini) ◂— ret    
 R9   0x7ffff7de7ab0 (_dl_fini) ◂— push   rbp
 R10  0x846
 R11  0x7ffff6f4a740 (__libc_start_main) ◂— push   r14
 R12  0x4234e0 (_start) ◂— xor    ebp, ebp
 R13  0x7fffffffde70 ◂— 0x2
 R14  0x0
 R15  0x0
 RBP  0x7fffffffdd90 —▸ 0xa17df0 (__libc_csu_init) ◂— push   r15
 RSP  0x7fffffffdc50 —▸ 0x7fffffffde78 —▸ 0x7fffffffe221 ◂— '/usr/bin/php7'
 RIP  0xa0d041 (main+24) ◂— mov    rax, qword ptr fs:[0x28]
────────────────────────────────────[ DISASM ]─────────────────────────────────────
 ► 0xa0d041 <main+24>     mov    rax, qword ptr fs:[0x28] <0xa0d029>
   0xa0d04a <main+33>     mov    qword ptr [rbp - 8], rax
   0xa0d04e <main+37>     xor    eax, eax
   0xa0d050 <main+39>     mov    dword ptr [rbp - 0x128], 0
   0xa0d05a <main+49>     mov    dword ptr [rbp - 0x124], 0
   0xa0d064 <main+59>     mov    dword ptr [rbp - 0x120], 0
   0xa0d06e <main+69>     mov    qword ptr [rbp - 0x110], 0
   0xa0d079 <main+80>     mov    dword ptr [rbp - 0x12c], 1
   0xa0d083 <main+90>     mov    dword ptr [rbp - 0x11c], 0
   0xa0d08d <main+100>    mov    qword ptr [rbp - 0x108], 0
   0xa0d098 <main+111>    mov    qword ptr [rbp - 0x100], 0
─────────────────────────────────[ SOURCE (CODE) ]─────────────────────────────────
In file: /home/qianfa/Desktop/php_debug/php-7.2.12/sapi/cli/php_cli.c
   1197 #ifdef PHP_CLI_WIN32_NO_CONSOLE
   1198 int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nShowCmd)
   1199 #else
   1200 int main(int argc, char *argv[])
   1201 #endif
 ► 1202 {
   1203 #if defined(PHP_WIN32)
   1204 # ifdef PHP_CLI_WIN32_NO_CONSOLE
   1205 	int argc = __argc;
   1206 	char **argv = __argv;
   1207 # else
─────────────────────────────────────[ STACK ]─────────────────────────────────────
00:0000│ rsp  0x7fffffffdc50 —▸ 0x7fffffffde78 —▸ 0x7fffffffe221 ◂— '/usr/bin/php7'
01:0008│      0x7fffffffdc58 ◂— 0x2013a8f38
02:0010│      0x7fffffffdc60 —▸ 0x4234e0 (_start) ◂— xor    ebp, ebp
03:0018│      0x7fffffffdc68 —▸ 0x7fffffffde70 ◂— 0x2
04:0020│      0x7fffffffdc70 ◂— 0x0
... ↓
07:0038│      0x7fffffffdc88 —▸ 0x7ffff7de6ac6 (_dl_fixup+214) ◂— mov    r8, rax
───────────────────────────────────[ BACKTRACE ]───────────────────────────────────
 ► f 0           a0d041 main+24
   f 1     7ffff6f4a830 __libc_start_main+240
Breakpoint main
pwndbg>
```

没问题，可以正常调试。

## Clion调试

安装Clion，略

在Clion中导入php7.2.12

![](/assets/php/TIM截图20181205103709.png)

打开CMakeLists.txt

```
include_directories(.)
add_custom_target(makefile COMMAND make && make install WORKING_DIRECTORY ${PROJECT_SOURCE_DIR})
```

缺少了第一行，我这里会出错，估计是因为目录没有全部include的原因，教程上却没问题，很奇怪。

![](/assets/php/TIM截图20181205112938.png)

然后，打开菜单栏`Run -> Edit Configurations...`，Target选择makefile、**Executable选择PHP的可执行二进制程序**，我这里因为执行了`sudo ln -s /usr/local/php7/php7.2/bin/php php7`，所以选择`/usr/bin/php7`,如果没有这样做的话，就选择`/usr/local/php7/php7.2/bin/php`，Program arguments填写要执行的脚本名称、Working Directory填写要执行脚本的存放目录，配置见下图。

![](/assets/php/TIM截图20181205113251.png)

调试:

创建test.php。一定要在Working Directory中创建，不然找不到。

![](/assets/php/TIM截图20181205113554.png)

在`sapi/cli/php_cli.c`中main函数处下断点，点击Debug即可。

![](/assets/php/TIM截图20181205113737.png)

参考链接:

* http://www.cnblogs.com/enochzzg/p/9547595.html

* https://blog.csdn.net/baixiaoshi/article/details/73744280x