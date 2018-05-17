---
title: url相对路径和绝对路径
date: 2018-03-27 09:53:14
tags:
- 前端
---
jquery.min.js的绝对路径:

```
http://39.107.33.96:20000/static/js/jquery.min.js
```

##### 当前url: http://39.107.33.96:20000/index.php/view/

html中引入jquery.min.js,相对路径:
```
<script src="../static/js/jquery.min.js"></script>
```   

此时: 浏览器解析js的路径:  
```
http://39.107.33.96:20000/index.php/static/js/jquery.min.js  找不到
```


##### 当前url: http://39.107.33.96:20000/index.php/view

html中引入jquery.min.js，采用相对路径:
```
<script src="../static/js/jquery.min.js"></script>
```   

此时: js的路径:   http://39.107.33.96:20000/static/js/jquery.min.js 可以找到

#### 注意:http://39.107.33.96:20000/index.php/view/article/1152 可访问

##### 当前url: http://39.107.33.96:20000/index.php/view/article/1152/..%2f..%2f..%2f..%2findex.php

对于php而言，它获得的请求是url解码后的，%2F会被解码为/，apache和nginx会按照目录的方式来返回我们请求的资源。
也就是访问

```
http://39.107.33.96:20000/index.php/view/article/1152/../../../../index.php
=>
http://39.107.33.96:20000/index.php
```

浏览器在寻找js资源的时候，并没有对%2f进行解码，就认为
..%2f..%2f..%2f..%2findex.php这一坨是一段数据，但是又没有人来接收这段数据，相当于报废。
就好比输入url-https://www.baidu.com?id=1，向百度传递了一个参数id，但它后端没有接收的代码，相当于没有传递

js的路径：

```
<script src="/static/js/jquery.min.js"></script>
解析之后:
http://39.107.33.96:20000/static/js/jquery.min.js
```

```
<script src="static/js/jquery.min.js"></script>
解析之后:
http://39.107.33.96:20000/index.php/view/article/1152/static/js/jquery.min.js
```

采用相对路径引入js:
当我们向服务器提交这个请求的时候，服务器会按照phpinfo模式来读取这个url，

读到..%2f..%2f..%2f..%2findex.php这里就读不下去了，识别不了，退一步，把前面能识别的内容返回回来，也就是http://39.107.33.96:20000/index.php/view/article/1152
也就是url:

```
http://39.107.33.96:20000/index.php/view/article/1152/static/js/jquery.min.js
```
返回的内容就是:
```
http://39.107.33.96:20000/index.php/view/article/1152
```
中返回的内容































