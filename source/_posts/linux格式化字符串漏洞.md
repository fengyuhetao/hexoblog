---
title: linux格式化字符串漏洞
tags: ctf
abbrlink: 54897
date: 2018-06-08 17:55:43
---

# 格式化漏洞

### 漏洞代码:

```c
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
void win()
{
system("/bin/sh");
}
void main(int argc, char *argv[])
{
char buf[103];
fgets(buf, 103, stdin);
buf[strlen(buf)-1] = 0x0;
printf(buf);
exit(0);
}
```

### 编译

> gcc -Wno-format-security print_format.c -o print_format

### 前置知识

- %d 用于读取10进制数值

- %x 用于读取16进制数值

- %s 用于读取字符串值
- %c 用于读取字符
- %p 用于读取指针地址
- %n 用于讲当前字符串的长度打印到var中，例 printf（"test %hn",&var[其中var为两个字节]） printf（"test %n",&var[其中var为一个字节]）





### 计算偏移和填充



>  offset.py

```
from pwn import *
context.log_level = 'debug'

def exec_fmt(payload):
    p = process("./vuln")
    p.sendline(payload)
    info = p.recv()
    p.close()
    return info

autofmt = FmtStr(exec_fmt)

print autofmt.offset
```

