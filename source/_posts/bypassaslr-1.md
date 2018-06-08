---
title: bypassaslr_1
date: 2018-05-21 16:30:00
tags:
---

# 绕过ASLR--return2plt

## 编译代码

1. 源代码

   ```c
   #include<stdio.h>
   #include <string.h>
   /* Eventhough shell() function isnt invoked directly, its needed
      here since 'system@PLT' and 'exit@PLT' stub code should be pres
      ent in executable to successfully exploit it. */
   void shell() {
   	system("/bin/sh");
   	exit(0);
   }
   int main(int argc, char* argv[]) {
   	int i=0;
   	char buf[256];
   	read(0, buf, 500);
   	printf("%s\n",buf);
   	return 0;
   }
   ```

2. 编译命令

   ```shell
   #echo 2 > /proc/sys/kernel/randomize_va_space
   $gcc -g -fno-stack-protector -no-pie -o vuln vuln.c
   $sudo chown root vuln
   $sudo chgrp root vuln
   $sudo chmod +s vuln
   ```

## 调试代码

