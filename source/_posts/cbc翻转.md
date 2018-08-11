---
title: cbc翻转
tags: web安全
abbrlink: 62655
date: 2018-05-08 15:50:54
---

# CBC翻转攻击

参考链接:

1. https://blog.csdn.net/csu_vc/article/details/79619309
2. http://wooyun.jozxing.cc/static/drops/tips-7828.html

```php
<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">;
<html>
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
<title>Login Form</title>
<link href="static/css/style.css" rel="stylesheet" type="text/css" />
<script type="text/javascript" src="static/js/jquery.min.js"></script>
<script type="text/javascript">
$(document).ready(function() {
    $(".username").focus(function() {
        $(".user-icon").css("left","-48px");
    });
    $(".username").blur(function() {
        $(".user-icon").css("left","0px");
    });
    $(".password").focus(function() {
        $(".pass-icon").css("left","-48px");
    });
    $(".password").blur(function() {
        $(".pass-icon").css("left","0px");
    });
});
</script>
</head>
<?php
define("SECRET_KEY", file_get_contents('/root/key'));
define("METHOD", "aes-128-cbc");
session_start();
function get_random_iv(){
    $random_iv='';
    for($i=0;$i<16;$i++){
        $random_iv.=chr(rand(1,255));
    }
    return $random_iv;
}
function login($info){
    $iv = get_random_iv();
    $plain = serialize($info);
    $cipher = openssl_encrypt($plain, METHOD, SECRET_KEY, OPENSSL_RAW_DATA, $iv);
    $_SESSION['username'] = $info['username'];
    setcookie("iv", base64_encode($iv));
    setcookie("cipher", base64_encode($cipher));
}
function check_login(){
    if(isset($_COOKIE['cipher']) && isset($_COOKIE['iv'])){
        $cipher = base64_decode($_COOKIE['cipher']);
        $iv = base64_decode($_COOKIE["iv"]);
        if($plain = openssl_decrypt($cipher, METHOD, SECRET_KEY, OPENSSL_RAW_DATA, $iv)){
            $info = unserialize($plain) or die("<p>base64_decode('".base64_encode($plain)."') can't unserialize</p>");
            $_SESSION['username'] = $info['username'];
        }else{
            die("ERROR!");
        }
    }
}
function show_homepage(){
    if ($_SESSION["username"]==='admin'){
        echo '<p>Hello admin</p>';
        echo '<p>Flag is $flag</p>';
    }else{
        echo '<p>hello '.$_SESSION['username'].'</p>';
        echo '<p>Only admin can see flag</p>';
    }
    echo '<p><a href="loginout.php">Log out</a></p>';
}
if(isset($_POST['username']) && isset($_POST['password'])){
    $username = (string)$_POST['username'];
    $password = (string)$_POST['password'];
    if($username === 'admin'){
        exit('<p>admin are not allowed to login</p>');
    }else{
        $info = array('username'=>$username,'password'=>$password);
        login($info);
        show_homepage();
    }
}else{
    if(isset($_SESSION["username"])){
        check_login();
        show_homepage();
    }else{
        echo '<body class="login-body">
                <div id="wrapper">
                    <div class="user-icon"></div>
                    <div class="pass-icon"></div>
                    <form name="login-form" class="login-form" action="" method="post">
                        <div class="header">
                        <h1>Login Form</h1>
                        <span>Fill out the form below to login to my super awesome imaginary control panel.</span>
                        </div>
                        <div class="content">
                        <input name="username" type="text" class="input username" value="Username" onfocus="this.value=\'\'" />
                        <input name="password" type="password" class="input password" value="Password" onfocus="this.value=\'\'" />
                        </div>
                        <div class="footer">
                        <input type="submit" name="submit" value="Login" class="button" />
                        </div>
                    </form>
                </div>
            </body>';
    }
}
?>
</html>
```

首先确定解密后的字符串:

> a:2:{s:8:"username";s:5:"zdmin";s:8:"password";s:5:"12345"}

第一步: 替换cookie中的cipher为反转后的cipher.

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Date    : 2018-03-15 11:45:57
# @Author  : Mr.zhang(s4ad0w.protonmail.com)
# @Link    : http://blog.csdn.net/csu_vc
# username='zdmin'
import base64
import requests
import urllib
iv_raw='DbVDXiIkUNkqwCARqqpYew%3D%3D'  #这里填写第一次返回的iv值
cipher_raw='27cekPxVgR5hbiu6ay%2BUfSmepzwYYL%2FTdbN1K3kmrH2EJ%2FqvBtAdKGGqGytHgJXJzNB26ZRX5dwupd885R%2Bwng%3D%3D'  #这里填写第一次返回的cipher值
print "[*]原始iv和cipher"
print "iv_raw:  " + iv_raw
print "cipher_raw:  " + cipher_raw
print "[*]对cipher解码，进行反转"
cipher = base64.b64decode(urllib.unquote(cipher_raw))
#a:2:{s:8:"username";s:5:"zdmin";s:8:"password";s:5:"12345"}
#s:2:{s:8:"userna
#me";s:5:"zdmin";
#s:8:"password";s
#:3:"12345";}
xor_cipher = cipher[0:9] +  chr(ord(cipher[9]) ^ ord('z') ^ ord('a')) + cipher[10:]  #请根据你的输入自行更改，原理看上面的介绍
xor_cipher=urllib.quote(base64.b64encode(xor_cipher))
print "反转后的cipher：" + xor_cipher
```

第二步: 第一步执行完，得到一个base64之后的字符串，代入下边的脚本，然后替换cookie中的iv为以下脚本输出的新的iv值。

```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
# @Date    : 2018-03-15 11:56:20
# @Author  : csu_vc(s4ad0w.protonmail.com)
# @Link    : http://blog.csdn.net/csu_vc
import base64
import urllib
cipher = 't8hxonGnYZs19dVMEzqeZm1lIjtzOjU6ImFkbWluIjtzOjg6InBhc3N3b3JkIjtzOjU6IjEyMzQ1Ijt9'#填写提交后所得的无法反序列化密文
iv = 'DbVDXiIkUNkqwCARqqpYew%3D%3D'#一开始提交的iv
cipher = urllib.unquote(cipher)
cipher = base64.b64decode(cipher)
iv = base64.b64decode(urllib.unquote(iv))
newIv = ''
right = 'a:2:{s:8:"userna'#被损坏前正确的明文
for i in range(16):
    newIv += chr(ord(right[i])^ord(iv[i])^ord(cipher[i])) #这一步相当于把原来iv中不匹配的部分修改过来
print urllib.quote(base64.b64encode(newIv))
```



